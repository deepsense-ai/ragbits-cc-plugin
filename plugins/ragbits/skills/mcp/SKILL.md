---
name: mcp
description: Scaffold a new MCP (Model Context Protocol) server powered by the ragbits library. Triggers whenever the user wants to create, build, generate, expose, or set up an MCP server that wraps ragbits capabilities — including agents, RAG pipelines, prompts, or plain tools — for consumption by Claude Desktop, Claude Code, Cursor, or any other MCP client. Use even for "expose my ragbits agent over MCP", "make my ragbits RAG available to Claude Desktop", "wrap my DocumentSearch as MCP tools", "turn this ragbits pipeline into an MCP server", or "set up a ragbits MCP server with stdio / SSE / streamable HTTP transport".
---

# Create a Ragbits-Backed MCP Server

Scaffold a complete, immediately runnable MCP server that exposes ragbits primitives (Agent / DocumentSearch / Prompt / plain tools) to any MCP client. Generate the full template first, install dependencies, then let the user customize.

## Important context: ragbits is MCP-client, not MCP-server

The `ragbits-agents` package ships a **client** for MCP (`MCPServerStdio`, `MCPServerSse`, `MCPServerStreamableHttp`) — it lets a ragbits Agent *consume* tools from an external MCP server. Ragbits has no first-party MCP server hosting story.

This skill therefore generates a **bridge project**: the official Python `mcp` SDK (FastMCP) hosts the server, and each MCP tool's body calls into ragbits. That's the only officially supported path today and matches the community pattern.

If what the user actually wants is a ragbits Agent that *calls* an external MCP server, route them to the `ragbits:agent` skill (with the `new-mcp.md` checklist) instead — this skill is for the other direction.

## Parse Arguments

Parse `$ARGUMENTS`:
- **app-name** (first positional, default: `my-mcp-server`) — project directory name
- **description** (remaining text after app-name, optional) — what the server should expose and why
- **--backend BACKEND** (default: inferred) — which ragbits primitive backs the tools: `agent`, `rag`, `prompt`, or `plain`
- **--transport TRANSPORT** (default: `stdio`) — `stdio` (Claude Desktop / Code), `sse`, or `streamable-http`
- **--model MODEL** (default: `gpt-4.1`) — LiteLLM-compatible model name, used when the backend needs an LLM

If `$ARGUMENTS` is empty and there is no prior context describing what to build, ask one message:

> "What should the MCP server be called, and what ragbits capability should it expose? (e.g. `docs-mcp Expose our internal docs RAG to Claude Desktop`)"

Generate immediately after receiving an answer — do not ask follow-up questions.

## Select Backend

Before generating files, infer the backend from the description. Default to **agent** unless the description clearly signals something else. Do not ask the user — pick one and mention the choice in the final summary so they can override.

| Description signals | Backend | What it generates |
|---------------------|---------|-------------------|
| "assistant", "answer", "help with", "chat-style", tool use, multi-step reasoning | `agent` (default) | MCP tool that dispatches to a ragbits `Agent` (Prompt + LiteLLM + tools) |
| "search docs", "RAG", "knowledge base", "retrieve passages", "index", "vector store" | `rag` | MCP tools for `search` (+ optional `ask`) backed by `DocumentSearch` |
| "structured extraction", "classify into", "return a JSON with", "one-shot LLM call", typed input/output | `prompt` | MCP tool that invokes a typed `Prompt[In, Out]` via LiteLLM |
| "no LLM needed", "just expose these Python functions", "wrap this utility" | `plain` | MCP tools that are plain Python — ragbits used only for logging/config |

See `references/mcp-spec.md` for the full patterns and when to combine them (e.g. one server exposing a RAG `search` tool AND an agent `ask` tool).

## Select Transport

Default to `stdio`. Upgrade based on signals:

- "Claude Desktop", "Claude Code", "local", default → `stdio`
- "remote", "deploy", "HTTP", "over the network", "web" → `sse` or `streamable-http`
- "long-running", "persistent connection", "bidirectional streaming" → `streamable-http`

See `references/checklists/new-transport.md` for tradeoffs and env-var handling.

## Project Structure

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    ├── server.py        # FastMCP instance, tool definitions, entry point
    └── backend.py       # ragbits Agent / DocumentSearch / Prompt behind the tools
```

`{app_name_snake}` = app-name converted to snake_case (hyphens → underscores). Always write an empty `__init__.py` so the directory is a valid Python package.

For `--backend plain` the `backend.py` is optional — put the utilities directly in `server.py` if they're small.

## Generate Files

### pyproject.toml

The base install needs two things: the `mcp` SDK (for the server) and whichever ragbits packages match the backend. Keep the dependency list tight — the skill's job is to get running, not to pre-install everything.

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "mcp[cli]>=1.2",                    # FastMCP + the `mcp` CLI for local testing
    {ragbits-dep-block}                 # see below — depends on backend
]
```

`{ragbits-dep-block}` by backend:

| Backend | Dependencies |
|---------|--------------|
| `agent` | `"ragbits-agents"` |
| `rag` | `"ragbits-document-search"`, plus a vector-store extra if the description names one: `"ragbits-core[chroma]"`, `"ragbits-core[qdrant]"`, `"ragbits-core[pgvector]"` |
| `prompt` | `"ragbits-core"` |
| `plain` | `"ragbits-core"` (for logging/config utilities; drop entirely if nothing ragbits-y is used) |

### server.py — the MCP surface

FastMCP is the recommended high-level API of the `mcp` SDK. Tools are declared with `@mcp.tool()`; docstrings and type hints feed the tool schema the MCP client sees. The `run()` call at the bottom chooses the transport.

Canonical shape (fill in the tool bodies from the backend-specific checklist):

```python
"""
{App Name} — MCP server exposing {one-line purpose}.

Run with:
    python -m {app_name_snake}.server
"""

import asyncio
import logging

from mcp.server.fastmcp import FastMCP

from .backend import backend  # async singleton — see backend.py

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

mcp = FastMCP("{app-name}")


@mcp.tool()
async def {primary_tool_name}({primary_tool_args}) -> {primary_tool_return}:
    """
    {One-sentence description the MCP client will show to users.
    Expand with argument meanings and what the tool returns — this is
    the prompt-side contract, so be specific.}
    """
    return await backend.{primary_tool_method}({forwarded_args})


# Add more @mcp.tool() functions as needed. Each one becomes a tool the
# MCP client (Claude Desktop, Claude Code, Cursor, ...) can invoke.


def main() -> None:
    mcp.run(transport="{transport}")  # "stdio" | "sse" | "streamable-http"


if __name__ == "__main__":
    main()
```

Notes:
- `FastMCP` is imported from `mcp.server.fastmcp`, *not* from any ragbits module — there is no `ragbits.mcp.server`.
- Tool docstrings are consumed by the MCP client to decide *when* to call the tool. Treat them as part of the prompt: describe inputs, outputs, and when the tool should be chosen, not just what it does internally.
- For `stdio` transport, do **not** print to stdout from tool bodies — stdout is the protocol channel. Use `logger.info(...)` / stderr. FastMCP already enforces this, but ad-hoc `print()` calls will corrupt the stream.

### backend.py — the ragbits side

Shape the backend around the chosen primitive. Read the matching checklist before generating:

- `--backend agent` → `references/checklists/new-agent-tool.md`
- `--backend rag` → `references/checklists/new-rag-tool.md`
- `--backend prompt` → `references/checklists/new-prompt-tool.md`
- `--backend plain` → skip backend.py; put helpers in `server.py`

Always export a single module-level `backend` object (or equivalent) so `server.py` has one import to worry about. That one indirection is what lets the user swap model / index / tool list without touching the MCP surface.

### README.md

Include:
- What the server does (from the description) and which MCP clients it targets
- Prerequisites: Python 3.10+, `OPENAI_API_KEY` (or the env var for the chosen model), any vector-store credentials for the `rag` backend
- Install: `pip install -e .`
- **Local smoke test** with `mcp dev` (the CLI from `mcp[cli]`): `mcp dev {app_name_snake}/server.py` — opens the MCP Inspector so the user can call tools without plumbing in a real client
- Run directly: `python -m {app_name_snake}.server`
- **Claude Desktop / Code wiring**: paste-ready JSON snippet for the `mcpServers` block in `claude_desktop_config.json` (or `.claude.json`), using `command: "python"` and `args: ["-m", "{app_name_snake}.server"]` — or the SSE/HTTP URL when the transport is remote
- How to add more tools (one `@mcp.tool()` per function, keep backend singleton in `backend.py`)
- Pointers to the per-backend checklists

## Component Extensions

When the description mentions any of these, read the corresponding checklist and add the component **without asking**.

| User mentions | Read | What to add |
|---------------|------|-------------|
| "resources", "static files", "expose this document" | `references/checklists/new-resource.md` | `@mcp.resource()` entries for read-only data |
| "prompts", "reusable prompt templates", "slash commands" | `references/checklists/new-mcp-prompt.md` | `@mcp.prompt()` entries for client-side prompt templates |
| auth, bearer token, API key guarding the server | `references/checklists/new-transport.md` (Auth section) | header check in a lifespan hook |
| ragbits hooks / logging / guardrails | `references/checklists/new-hook.md` from the `ragbits:agent` skill (link to it — don't duplicate) | hooks wired into the backend Agent |

For advanced patterns (multiple tools sharing one backend, lifespan management of vector stores, combining agent + RAG on one server, typed structured outputs) see `references/mcp-spec.md`.

## Install Dependencies

After generating all files, detect if working inside a local ragbits source repo:

```bash
find . -maxdepth 6 -name "ragbits-core" -type d 2>/dev/null | head -1
```

**If found (developing ragbits from source):**
```bash
pip install "mcp[cli]>=1.2" \
            -e {path_to_repo}/packages/ragbits-core
# Add the package(s) matching the backend:
#   agent  → -e {path_to_repo}/packages/ragbits-agents
#   rag    → -e {path_to_repo}/packages/ragbits-document-search
```

**Otherwise:**
```bash
cd {app-name} && pip install -e .
```

Wait for installation to complete. If it fails, show the error and suggest a fix. A common failure is `mcp` resolving to an old unrelated PyPI package — the official one is `mcp` from Anthropic; pin `>=1.2` to be safe.

## Summary

After files are generated and dependencies are installed, print:

```
Created {app-name}/  (backend: {agent|rag|prompt|plain}, transport: {stdio|sse|streamable-http})
  {app_name_snake}/server.py   — FastMCP instance + tool definitions
  {app_name_snake}/backend.py  — ragbits primitive behind the tools
  {app_name_snake}/__init__.py
  pyproject.toml               — mcp[cli] + ragbits dependencies
  README.md                    — setup, smoke test, Claude Desktop wiring

Smoke test (opens the MCP Inspector in your browser):
  cd {app-name}
  mcp dev {app_name_snake}/server.py

Run directly:
  python -m {app_name_snake}.server

Wire into Claude Desktop / Code:
  Add this to your mcpServers config (see README for the exact path):
  {
    "{app-name}": {
      "command": "python",
      "args": ["-m", "{app_name_snake}.server"]
    }
  }

Next steps:
  • Edit backend.py — wire the ragbits Agent / DocumentSearch / Prompt to your real data
  • Add more @mcp.tool() functions in server.py as needed
  • Switch backend: see references/mcp-spec.md
  • Switch transport (SSE / streamable HTTP): see references/checklists/new-transport.md
  • Tighten tool docstrings — they are what drives the MCP client's tool selection
```

Mention any notable tradeoff surfaced by the backend choice (e.g. for `rag`: "the server instantiates the vector store on startup — first call may be slow while the index warms up"; for `agent`: "each tool call invokes an LLM, so cost and latency scale with calls").
