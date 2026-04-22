# New MCP Prompt Template Checklist

`@mcp.prompt()` ships a reusable prompt template that the MCP client can surface as a slash command or saved prompt. It's *not* the same thing as a ragbits `Prompt` class — an MCP prompt is metadata the client renders; a ragbits `Prompt` is code that runs server-side.

A natural use: expose canonical ragbits `Prompt` classes to end users as ready-made commands.

## When to reach for prompt templates

- You want the user (via their client) to trigger a known workflow without typing the full instructions
- You want to ship a library of "starters" that encode good defaults for the capability the server provides
- The template needs parameters the user fills in at call time

## Shape

```python
from mcp.server.fastmcp import FastMCP
from mcp.server.fastmcp.prompts import base

mcp = FastMCP("{app-name}")


@mcp.prompt()
def summarize_ticket(ticket_text: str) -> list[base.Message]:
    """Summarize a support ticket — use when a user drops in a ticket body."""
    return [
        base.UserMessage(
            f"Summarize this support ticket in 3 bullet points. "
            f"Call out urgency if it's time-sensitive.\n\n{ticket_text}"
        ),
    ]


@mcp.prompt()
def compare_options(option_a: str, option_b: str) -> list[base.Message]:
    """Side-by-side comparison of two options — use for product / vendor / design tradeoffs."""
    return [
        base.UserMessage(f"Option A: {option_a}"),
        base.UserMessage(f"Option B: {option_b}"),
        base.UserMessage("Compare these on: fit, risk, cost, reversibility."),
    ]
```

## Quality gates

- [ ] Prompt parameters are typed — clients render a form from the signature
- [ ] Docstring states when to pick this prompt over another — client surfaces this in its command menu
- [ ] Return type is `list[base.Message]` (or `str` for single-user-message prompts)
- [ ] Template body is directly actionable — no "[insert context here]" placeholders

## Prompts vs. tools vs. resources

- **Tool**: does something on the user's behalf (search, classify, send). Arguments fit a call.
- **Resource**: provides read-only content. Identified by URI.
- **Prompt template**: a reusable message the client can send *to its own LLM* with the user's parameters filled in. The server never actually runs the LLM for a prompt template.
