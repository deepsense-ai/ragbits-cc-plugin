---
name: create-agent
description: Scaffold a new ragbits agent application. Triggers whenever the user wants to create, build, generate, or set up a new agent powered by the ragbits library — including simple chatbots, tool-using assistants, A2A servers, and multi-agent orchestrators. Use even for "build a ragbits agent that does X", "I need a bot that uses ragbits", or "make me a ragbits assistant".
---

# Create a Ragbits Agent

Scaffold a complete, immediately runnable ragbits agent. Generate the full template first, install dependencies, then let the user customize.

## Parse Arguments

Parse `$ARGUMENTS`:
- **app-name** (first positional, default: `my-agent`) — project directory name
- **description** (remaining text after app-name, optional) — what the agent should do
- **--model MODEL** (default: `gpt-4.1`) — any LiteLLM-compatible model name

If `$ARGUMENTS` is empty and there is no prior context describing what to build, ask one message:

> "What should the agent be called, and what will it do? (e.g. `customer-support A bot that handles refund questions`)"

Generate immediately after receiving an answer — do not ask follow-up questions.

## Project Structure

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    ├── agent.py        # main agent, prompt, entry point
    └── tools.py        # tool functions (edit to add yours)
```

`{app_name_snake}` = app-name converted to snake_case (hyphens → underscores).

## Generate Files

### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits",
]
```

### agent.py

```python
import asyncio

from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt

from .tools import TOOLS


class {Name}Input(BaseModel):
    """Input model for the {Name} agent."""

    message: str


class {Name}Prompt(Prompt[{Name}Input]):
    """Prompt for the {Name} agent."""

    system_prompt = """
    {Write a specific, purposeful system prompt based on the agent description.
    Include the domain, key responsibilities, and any constraints.
    3–6 sentences — concrete, not generic.}
    """

    user_prompt = "{{ message }}"


async def run_agent(message: str) -> str:
    """Run the agent with a single message."""
    llm = LiteLLM(model_name="{model}")
    agent = Agent(llm=llm, prompt={Name}Prompt, tools=TOOLS)
    return await agent.run({Name}Input(message=message))


async def main() -> None:
    """Interactive chat loop."""
    print("{App Name}")
    print("=" * 40)
    print("Type 'quit' to exit.\n")

    while True:
        user_input = input("YOU: ").strip()
        if user_input.lower() in ("quit", "exit", "q"):
            break
        if not user_input:
            continue
        result = await run_agent(user_input)
        print(f"\nASSISTANT: {result}\n")


if __name__ == "__main__":
    asyncio.run(main())
```

Fill `{Name}` with PascalCase agent name. Write a **real** system prompt based on the description — not a placeholder.

### tools.py

```python
"""
Tools for the {Name} agent.

Each tool must:
- Have a clear docstring (the LLM reads it to decide when to call the tool)
- Use type hints on all parameters and return value
- Return str, dict, or a Pydantic model

Reference: references/checklists/new-tool.md
"""


def example_tool(query: str) -> str:
    """
    Example placeholder tool — replace with your actual implementation.

    Args:
        query: The input or search query.

    Returns:
        A result string.
    """
    # TODO: implement
    return f"Result for: {query}"


TOOLS = [example_tool]
```

### README.md

Include:
- What the agent does (from the description)
- Prerequisites (`OPENAI_API_KEY` or the env var for the chosen model)
- Setup: `pip install -e .`
- How to run: `python -m {app_name_snake}.agent`
- Customization guide: editing tools.py, updating system_prompt, adding hooks/MCP/A2A
- Brief reference to checklist files for advanced features

## Component Extensions

If the description (or user message) mentions any of the following, read the corresponding checklist and add the component **without asking**:

| User mentions | Read | What to generate |
|---------------|------|-----------------|
| hooks / logging / guardrails / confirmation / audit | `references/checklists/new-hook.md` | `hooks.py` + update Agent constructor |
| MCP / model context protocol / external tool server | `references/checklists/new-mcp.md` | MCP server config in agent.py |
| A2A / serve / expose as server / other agents call it | `references/checklists/new-a2a.md` | A2A server wrapper in agent.py |
| orchestrate / coordinate / remote agents / multi-agent | `references/checklists/new-a2a.md` | Remote agent discovery + `execute_agent` tool |

For advanced patterns (streaming, structured output, conversation history, tool context injection), see `references/agent-spec.md`.

## Install Dependencies

After generating all files, detect if working inside a local ragbits source repo:

```bash
find . -maxdepth 6 -name "ragbits-core" -type d 2>/dev/null | head -1
```

**If found (developing ragbits from source):**
```bash
pip install -e {path_to_repo}/packages/ragbits-core \
            -e {path_to_repo}/packages/ragbits-agents
```

**Otherwise:**
```bash
cd {app-name} && pip install -e .
```

Wait for installation to complete. If it fails, show the error and suggest a fix.

## Summary

After creating all files and installing dependencies, print:

```
Created {app-name}/
  {app_name_snake}/agent.py    — agent, prompt, entry point
  {app_name_snake}/tools.py    — tool functions (customize these)
  pyproject.toml               — project + dependencies
  README.md                    — setup and usage

Run:
  cd {app-name}
  python -m {app_name_snake}.agent

Next steps:
  • Edit tools.py — replace example_tool with your real tools
  • Tune the system_prompt in agent.py for your domain
  • Add hooks (logging, guardrails): see references/checklists/new-hook.md
  • Add MCP servers: see references/checklists/new-mcp.md
  • Expose as A2A: see references/checklists/new-a2a.md
```