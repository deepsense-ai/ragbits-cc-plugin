# Interview Checklist

Domain questions that shape backend, transport, and tool surface. Use these when the user provides minimal context. Combine into one message — do not ask one at a time.

## Core (always relevant)

- **Server name**: What directory/module name? (snake_case preferred)
- **Purpose**: What ragbits capability is being exposed, and what will MCP clients use it for?
- **Primary client**: Claude Desktop? Claude Code? Cursor? A custom script? Determines transport.

## Backend selection

Pick one — don't combine unless the user explicitly wants multiple tools on one server.

- Does it need reasoning and tool use? → **agent** (ragbits `Agent`)
- Does it need to retrieve passages from a corpus? → **rag** (`DocumentSearch`)
- Is it a single structured LLM call? → **prompt** (`Prompt[In, Out]` + `LiteLLM`)
- No LLM at all — just exposing Python functions? → **plain**

## RAG-specific (if backend = rag)

- Which vector store? (Chroma, Qdrant, pgvector, in-memory for dev)
- Is there an existing index to connect to, or does the server need to ingest at startup?
- Embedding model? (default `text-embedding-3-small`)
- Should the server expose a plain `search` tool, or also an `ask` tool that calls an LLM over the retrieved passages?

## Agent-specific (if backend = agent)

- Which LLM? (default `gpt-4.1`; any LiteLLM-compatible)
- Does the agent need tools of its own (weather API, database, file ops)?
- Any guardrails / input validation / max-length constraints?

## Transport

- **stdio** (default): local subprocess — the right choice for Claude Desktop / Code
- **sse**: HTTP+SSE server for remote clients
- **streamable-http**: bidirectional HTTP streaming, long-running sessions
- Auth needed? (bearer token, shared secret) — only meaningful for SSE/HTTP

## Operational

- Environment variables the server needs (`OPENAI_API_KEY`, vector-store credentials)
- Is this run locally by the user, or deployed somewhere? (container, systemd, serverless)
- Expected request volume — shapes whether to cache tool schemas, pool connections, etc.
