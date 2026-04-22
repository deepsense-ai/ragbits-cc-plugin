# New Agent-Backed MCP Tool Checklist

Quality gates for exposing a ragbits `Agent` as one or more MCP tools.

## Dependencies

```toml
dependencies = [
    "mcp[cli]>=1.2",
    "ragbits-agents",
]
```

## Shape

Two files: `backend.py` owns the ragbits `Agent`; `server.py` owns the MCP surface. One singleton, imported once.

```python
# backend.py
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class AskInput(BaseModel):
    query: str


class AssistantPrompt(Prompt[AskInput]):
    system_prompt = """
    {Real system prompt — specific to the domain the server exposes.
    3–6 sentences, not a placeholder.}
    """
    user_prompt = "{{ query }}"


class AgentBackend:
    def __init__(self) -> None:
        self._agent = Agent(
            llm=LiteLLM(model_name="gpt-4.1", use_structured_output=True),
            prompt=AssistantPrompt,
            tools=[],  # add ragbits tool functions here
        )

    async def ask(self, query: str) -> str:
        result = await self._agent.run(AskInput(query=query))
        return result.content  # unwrap — DO NOT return the AgentResult


backend = AgentBackend()
```

```python
# server.py
from mcp.server.fastmcp import FastMCP

from .backend import backend

mcp = FastMCP("{app-name}")


@mcp.tool()
async def ask(query: str) -> str:
    """
    {One sentence the MCP client will show. Describe when to use this tool
    and what the answer looks like — this drives the client's tool selection.}
    """
    return await backend.ask(query)


def main() -> None:
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

## Quality gates

- [ ] `backend.py` constructs the `Agent` once at module scope; tool bodies reuse it
- [ ] Tool body returns `result.content`, not the raw `AgentResult`
- [ ] Tool docstring names the *when*, not just the *what*
- [ ] System prompt is domain-specific (not "You are a helpful assistant")
- [ ] If the agent takes tools, they have type hints and docstrings — the LLM reads them
- [ ] No `print()` in tool bodies when transport is stdio — use `logging`

## Common extensions

- **Multiple tools on one agent**: add more `@mcp.tool()` functions that all delegate to `backend.ask(...)` with prompt-routing logic, or give each tool its own `Prompt` subclass and a second method on `AgentBackend`.
- **Streaming**: FastMCP supports streaming tool output via `Context.info(...)` during a tool call. Useful when the underlying agent produces long answers.
- **Hooks**: attach ragbits hooks to the backend `Agent` for logging / guardrails / confirmation. See `../../agent/references/checklists/new-hook.md` — nothing MCP-specific about it.

## When to reach for the `rag` or `prompt` backend instead

- If every tool call boils down to "retrieve passages and return them", the RAG backend is simpler and cheaper — no LLM on the server path.
- If every tool call is a single structured LLM call with a fixed schema, the `prompt` backend returns typed output directly and skips the Agent loop.
