# Ragbits MCP Server — Deep Reference

Bridge patterns for exposing ragbits primitives through an MCP server built on FastMCP. Read this when the base template in `SKILL.md` isn't enough — e.g. combining multiple backends, managing expensive resources across requests, or returning structured output.

## Table of contents
- [Why FastMCP](#why-fastmcp)
- [Minimal agent server](#minimal-agent-server)
- [RAG server with lifespan](#rag-server-with-lifespan)
- [Typed Prompt server with structured output](#typed-prompt-server-with-structured-output)
- [Combining backends](#combining-backends)
- [Resources and prompt templates](#resources-and-prompt-templates)
- [Logging and the stdio trap](#logging-and-the-stdio-trap)
- [Testing and debugging](#testing-and-debugging)

## Why FastMCP

`FastMCP` is the high-level decorator-based API inside the official `mcp` SDK (`pip install "mcp[cli]"`). It handles JSON-RPC plumbing, the capability handshake, and tool-schema generation from Python type hints, so tool authoring collapses to:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool()
async def do_thing(query: str) -> str:
    """Docstring becomes the tool description shown to the MCP client."""
    ...

if __name__ == "__main__":
    mcp.run()  # stdio by default
```

Ragbits does not provide an equivalent. The only server-side choice is FastMCP (or the lower-level `mcp.server` if you need fine control over JSON-RPC — rare).

## Minimal agent server

Wraps a ragbits `Agent`. One tool, one module-level singleton. This is the shape the `agent` backend produces.

```python
# backend.py
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class AskInput(BaseModel):
    query: str


class AssistantPrompt(Prompt[AskInput]):
    system_prompt = "You are a helpful assistant. Keep answers concise."
    user_prompt = "{{ query }}"


class Backend:
    def __init__(self) -> None:
        self._agent = Agent(
            llm=LiteLLM(model_name="gpt-4.1", use_structured_output=True),
            prompt=AssistantPrompt,
            tools=[],
        )

    async def ask(self, query: str) -> str:
        result = await self._agent.run(AskInput(query=query))
        return result.content


backend = Backend()
```

```python
# server.py
from mcp.server.fastmcp import FastMCP
from .backend import backend

mcp = FastMCP("my-agent-mcp")

@mcp.tool()
async def ask(query: str) -> str:
    """Ask the assistant a question. Returns the answer as plain text."""
    return await backend.ask(query)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Notes:
- `backend` is a module-level singleton — constructed once at import, reused across tool calls. That matters because the `LiteLLM` instance holds a connection pool.
- `.content` unwraps the `AgentResult`. Returning `result` directly leaks `AgentResult(content=..., metadata=..., ...)` to the client, which shows up as a useless `repr` in the MCP inspector — the single most common bridge bug.

## RAG server with lifespan

Vector stores are expensive to initialize. Use FastMCP's lifespan hook so the server warms the index once and reuses it across calls.

```python
# backend.py
from contextlib import asynccontextmanager
from dataclasses import dataclass

from ragbits.core.embeddings.dense import LiteLLMEmbedder
from ragbits.core.vector_stores.chroma import ChromaVectorStore
from ragbits.document_search import DocumentSearch


@dataclass
class Deps:
    search: DocumentSearch


@asynccontextmanager
async def lifespan(_server):
    vector_store = ChromaVectorStore(
        client=...,            # ChromaDB client
        index_name="docs",
        embedder=LiteLLMEmbedder(model_name="text-embedding-3-small"),
    )
    search = DocumentSearch(vector_store=vector_store)
    # ingest or connect to an existing index here
    yield Deps(search=search)
```

```python
# server.py
from mcp.server.fastmcp import FastMCP, Context
from .backend import lifespan

mcp = FastMCP("docs-mcp", lifespan=lifespan)


@mcp.tool()
async def search_docs(query: str, top_k: int = 5, ctx: Context | None = None) -> list[dict]:
    """
    Search the documentation corpus for passages relevant to the query.

    Args:
        query: Natural-language question or keyword search.
        top_k: How many passages to return (default 5).

    Returns:
        A list of {text, score, source} dicts, most relevant first.
    """
    deps = ctx.request_context.lifespan_context  # typed as Deps
    elements = await deps.search.search(query, config={"k": top_k})
    return [
        {"text": e.text_representation, "score": e.score, "source": e.document_meta.source}
        for e in elements
    ]
```

The `ctx` parameter is an opt-in: FastMCP injects `Context` when the tool signature names it. That's how you reach lifespan state without a global.

## Typed Prompt server with structured output

When the tool's job is one shot of constrained LLM output, skip the Agent and invoke a `Prompt` directly. FastMCP converts a Pydantic return value into a JSON schema the MCP client can rely on.

```python
# backend.py
from pydantic import BaseModel
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class ClassifyInput(BaseModel):
    ticket: str


class Classification(BaseModel):
    category: str
    urgency: int  # 1-5
    reason: str


class ClassifyPrompt(Prompt[ClassifyInput, Classification]):
    system_prompt = "Classify the support ticket. Urgency: 1 routine, 5 critical."
    user_prompt = "{{ ticket }}"


llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)


async def classify(ticket: str) -> Classification:
    prompt = ClassifyPrompt(ClassifyInput(ticket=ticket))
    return await llm.generate(prompt)
```

```python
# server.py
from mcp.server.fastmcp import FastMCP
from .backend import Classification, classify

mcp = FastMCP("classify-mcp")


@mcp.tool()
async def classify_ticket(ticket: str) -> Classification:
    """Classify a support ticket into a category with urgency 1–5 and a short reason."""
    return await classify(ticket)
```

The `Classification` Pydantic model defines the tool's output schema automatically. Clients that support structured tool output can consume the fields directly.

## Combining backends

Nothing stops you from mixing an agent tool and a RAG tool on one server. One `FastMCP` instance, multiple `@mcp.tool()` entries, each delegating to its own singleton:

```python
@mcp.tool()
async def search_docs(query: str, top_k: int = 5) -> list[dict]: ...

@mcp.tool()
async def ask_assistant(query: str) -> str: ...
```

Keep every tool's docstring explicit about when *it* is the right one to pick. The MCP client has to disambiguate between them on the fly — vague docstrings cause misrouting.

## Resources and prompt templates

FastMCP also exposes two lesser-known capabilities that map cleanly onto ragbits:

- `@mcp.resource("docs://{id}")` — read-only data fetched by URI. Back it with a `DocumentSearch.get(id)` or a direct file read. Resources are for *content the model consumes*, not actions.
- `@mcp.prompt()` — a reusable prompt template the client can offer as a command. A natural fit for surfacing canonical ragbits `Prompt` classes to end users.

See the per-feature checklists.

## Logging and the stdio trap

`stdio` transport uses stdout for JSON-RPC. Any stray `print()` in a tool body will corrupt the protocol and crash the session with an opaque parse error. Defenses:

- Configure `logging.basicConfig(level=logging.INFO)` once at module scope — `logging` writes to stderr by default.
- Use `logger.info(...)` in tool bodies, never `print(...)`.
- If a dependency prints to stdout, redirect it: `contextlib.redirect_stdout(sys.stderr)` around the offending call.

`sse` and `streamable-http` transports aren't affected — stdout is free for normal use.

## Testing and debugging

- `mcp dev {package}/server.py` opens the MCP Inspector (from `mcp[cli]`). It speaks to the server over stdio and gives you a UI to call tools and inspect schemas. This is the single most productive debugging tool — use it before wiring into a real client.
- Logs go to stderr; the Inspector shows them in its right-hand panel.
- For SSE/HTTP transports, `curl` the `/` endpoint to confirm the server is up before pointing a client at it.
- Errors raised from tool bodies surface to the client as `McpError`. Raise plain exceptions; FastMCP wraps them. Don't silently `return "error: ..."` — that looks like success to the client.

## Cross-references

- Client side (ragbits agent consuming external MCP servers): `../../agent/references/checklists/new-mcp.md`
- Agent authoring: `../../agent/SKILL.md`
- RAG authoring: `../../rag/SKILL.md`
- Official `mcp` SDK: https://github.com/modelcontextprotocol/python-sdk
- FastMCP docs: https://github.com/modelcontextprotocol/python-sdk#fastmcp
