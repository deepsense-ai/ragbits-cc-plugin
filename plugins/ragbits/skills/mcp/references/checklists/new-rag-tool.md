# New RAG-Backed MCP Tool Checklist

Quality gates for exposing a ragbits `DocumentSearch` pipeline as MCP tools.

## Dependencies

```toml
dependencies = [
    "mcp[cli]>=1.2",
    "ragbits-document-search",
    "ragbits-core[chroma]",   # or [qdrant], [pgvector] — match the vector store
]
```

## Shape

Vector stores and embedders are expensive to construct; use FastMCP's lifespan hook so they warm up once.

```python
# backend.py
from contextlib import asynccontextmanager
from dataclasses import dataclass

import chromadb
from ragbits.core.embeddings.dense import LiteLLMEmbedder
from ragbits.core.vector_stores.chroma import ChromaVectorStore
from ragbits.document_search import DocumentSearch


@dataclass
class Deps:
    search: DocumentSearch


@asynccontextmanager
async def lifespan(_server):
    client = chromadb.PersistentClient(path="./chroma-data")
    vector_store = ChromaVectorStore(
        client=client,
        index_name="docs",
        embedder=LiteLLMEmbedder(model_name="text-embedding-3-small"),
    )
    search = DocumentSearch(vector_store=vector_store)
    # If you need to ingest on startup:
    # await search.ingest(sources=[...])
    yield Deps(search=search)
```

```python
# server.py
from mcp.server.fastmcp import FastMCP, Context

from .backend import lifespan

mcp = FastMCP("{app-name}", lifespan=lifespan)


@mcp.tool()
async def search_docs(query: str, top_k: int = 5, ctx: Context | None = None) -> list[dict]:
    """
    Search the documentation corpus for passages relevant to the query.

    Args:
        query: Natural-language question or keyword.
        top_k: How many passages to return (default 5, max recommended 20).

    Returns:
        A list of {text, score, source} dicts, most relevant first.
    """
    deps = ctx.request_context.lifespan_context
    elements = await deps.search.search(query, config={"k": top_k})
    return [
        {
            "text": e.text_representation,
            "score": getattr(e, "score", None),
            "source": getattr(e.document_meta, "source", None),
        }
        for e in elements
    ]


def main() -> None:
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

## Optional: ask-over-docs tool

When the user wants a "chat with my docs" surface instead of raw passages, add a second tool that pipes retrieval through an LLM:

```python
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt
from pydantic import BaseModel

class AskInput(BaseModel):
    query: str
    context: list[str]

class AskPrompt(Prompt[AskInput]):
    system_prompt = "Answer the question using only the provided context. Cite passage indices."
    user_prompt = "Question: {{ query }}\n\nContext:\n{% for c in context %}[{{ loop.index0 }}] {{ c }}\n{% endfor %}"


llm = LiteLLM(model_name="gpt-4.1")


@mcp.tool()
async def ask_docs(query: str, top_k: int = 5, ctx: Context | None = None) -> str:
    """Answer a question grounded in the documentation corpus, with passage citations."""
    deps = ctx.request_context.lifespan_context
    elements = await deps.search.search(query, config={"k": top_k})
    prompt = AskPrompt(AskInput(query=query, context=[e.text_representation for e in elements]))
    return await llm.generate(prompt)
```

Keep `search_docs` around even when `ask_docs` exists — clients often want to do their own synthesis.

## Quality gates

- [ ] Vector store constructed in `lifespan`, not in the tool body
- [ ] Embedder instance is reused across requests
- [ ] Tool accepts `top_k` so clients can trade off recall vs. token budget
- [ ] Returned dicts use stable keys (`text`, `score`, `source`) — clients parse these
- [ ] Index path is absolute or relative to a configured `DATA_DIR`, never relative to CWD (stdio launcher may set CWD anywhere)
- [ ] Ingestion is idempotent or gated — don't re-ingest on every startup

## When to reach for the agent or prompt backend instead

- If the server's answer needs multi-step reasoning or tool use on top of retrieval, move to the `agent` backend and let the agent call a retrieval tool.
- If you're synthesizing a structured report (not just returning passages), the `prompt` backend with a typed output schema gives clients a cleaner contract.
