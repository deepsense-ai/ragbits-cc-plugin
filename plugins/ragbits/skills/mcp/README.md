# ragbits:mcp

Skill that scaffolds a new **MCP (Model Context Protocol) server** powered by ragbits — complete, immediately runnable, wired to the official `mcp` Python SDK (FastMCP) on the server side and a ragbits primitive (Agent / DocumentSearch / Prompt) on the implementation side.

## Why a bridge?

Ragbits ships MCP **client** classes (`MCPServerStdio`, `MCPServerSse`, `MCPServerStreamableHttp`) for consuming external MCP servers from a ragbits agent. There is **no first-party ragbits MCP server** module — nothing like `ragbits.mcp.server` exists. To expose a ragbits capability over MCP, you bridge: FastMCP hosts the server, each `@mcp.tool()` body calls into ragbits.

The skill generates exactly that bridge.

## What it generates

- `pyproject.toml` with `mcp[cli]` plus the ragbits package(s) matching the chosen backend
- `server.py` with a `FastMCP` instance, one or more `@mcp.tool()` functions, and a `main()` that calls `mcp.run(transport=...)`
- `backend.py` with the ragbits `Agent` / `DocumentSearch` / `Prompt` singleton the tools delegate to
- `README.md` with setup, the `mcp dev` smoke-test command, and paste-ready `mcpServers` config for Claude Desktop / Claude Code

## Example: expose a RAG pipeline to Claude Desktop

```
/ragbits:mcp docs-mcp Expose our product documentation RAG as an MCP server that Claude Desktop can search --backend rag
```

Produces a `docs-mcp/` project whose MCP tool (`search_docs(query, top_k)`) runs a `DocumentSearch` over a configurable corpus and returns scored passages, packaged for stdio transport and ready to drop into `claude_desktop_config.json`.

## Example: expose a ragbits agent

```
/ragbits:mcp support-mcp Wrap our support-triage ragbits agent so any MCP client can call it
```

Defaults to `--backend agent`: the MCP tool forwards the query to a ragbits `Agent` (with system prompt + tools + LiteLLM), unwraps `result.content`, and returns the text reply.

## When to pick a different skill

- **Building the ragbits agent itself** (not exposing one) → use `ragbits:agent`
- **Building the RAG pipeline itself** (not exposing one) → use `ragbits:rag`
- **Building a chat UI over a ragbits backend** → use `ragbits:chat`
- **Consuming an external MCP server from a ragbits agent** → use `ragbits:agent` with the `new-mcp.md` checklist — this skill is the *opposite* direction

## References

| File | Purpose |
|------|---------|
| `references/mcp-spec.md` | FastMCP + ragbits bridge patterns, lifespan, multi-tool servers, structured output |
| `references/interview-checklist.md` | Domain questions that shape backend + transport + tool selection |
| `references/checklists/new-agent-tool.md` | Expose a ragbits `Agent` as an MCP tool |
| `references/checklists/new-rag-tool.md` | Expose a `DocumentSearch` pipeline as MCP `search` / `ask` tools |
| `references/checklists/new-prompt-tool.md` | Expose a typed `Prompt[In, Out]` as an MCP tool with a structured return |
| `references/checklists/new-transport.md` | `stdio` vs `sse` vs `streamable-http` tradeoffs, auth, env vars |
| `references/checklists/new-resource.md` | Expose read-only data as `@mcp.resource()` |
| `references/checklists/new-mcp-prompt.md` | Ship reusable prompt templates as `@mcp.prompt()` |
