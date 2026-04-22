# New Prompt-Backed MCP Tool Checklist

Quality gates for exposing a typed ragbits `Prompt[In, Out]` as an MCP tool. Use this when every call is a single structured LLM invocation — no agent loop, no retrieval, no tool use.

## Dependencies

```toml
dependencies = [
    "mcp[cli]>=1.2",
    "ragbits-core",
]
```

## Shape

`Prompt[In, Out]` + `LiteLLM.generate()` is the tightest possible path: the LLM returns a Pydantic instance, the MCP tool returns the same instance, and FastMCP projects its schema onto the MCP tool output.

```python
# backend.py
from pydantic import BaseModel, Field
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class ClassifyInput(BaseModel):
    ticket: str


class Classification(BaseModel):
    category: str = Field(description="refund | tech | billing")
    urgency: int = Field(ge=1, le=5, description="1 routine, 5 critical")
    reason: str


class ClassifyPrompt(Prompt[ClassifyInput, Classification]):
    system_prompt = """
    Classify the support ticket into one category and rate its urgency on a 1–5 scale.
    Be decisive — pick exactly one category.
    """
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

mcp = FastMCP("{app-name}")


@mcp.tool()
async def classify_ticket(ticket: str) -> Classification:
    """Classify a support ticket into a category with urgency 1–5 and a short reason."""
    return await classify(ticket)


def main() -> None:
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

## Quality gates

- [ ] `Prompt` is declared with both generic parameters: `Prompt[InputModel, OutputModel]`
- [ ] `LiteLLM(..., use_structured_output=True)` — otherwise the output isn't guaranteed to parse
- [ ] Output model uses `Field(description=...)` on non-obvious fields so the LLM has explicit semantics
- [ ] Tool signature uses the Pydantic output model as its return type — FastMCP reads it for the tool schema
- [ ] `llm` is constructed once at module scope — connection pool reuse

## Common extensions

- **Multiple prompts on one server**: declare a second `Prompt` + method, add a second `@mcp.tool()` entry. Avoid a single mega-prompt that branches internally — separate tools give the MCP client cleaner selection.
- **Input validation**: add Pydantic validators to the input model. Raised `ValidationError` surfaces to the client as a proper MCP error.
- **Enum outputs**: use `str, Enum` or `Literal["a", "b", "c"]` on output fields for stricter schemas.

## When to reach for the agent or rag backend instead

- If the task needs tool use or iterative reasoning → `agent` backend.
- If the task needs retrieval from a corpus → `rag` backend.
- If the prompt is free-form text with no structure, you can still use this backend but drop the output model (`Prompt[InputModel]` with `str` default).
