# Transport Checklist

Choose one of three transports when calling `mcp.run(transport=...)`. The choice affects deployment, auth, and how the MCP client is wired.

| Transport | When to use | Wiring |
|-----------|-------------|--------|
| `stdio` (default) | Local subprocess, single user. The right choice for Claude Desktop / Code. | Client spawns the process and talks over stdin/stdout. |
| `sse` | Remote server, one-way streaming to the client. Works behind a reverse proxy. | Client connects to an HTTP URL with SSE. |
| `streamable-http` | Remote server, bidirectional streaming, long-running sessions. | Client POSTs to the `/messages` endpoint and reads a chunked response. |

## stdio

Default. Nothing to configure beyond `mcp.run(transport="stdio")`.

**Client wiring (Claude Desktop — `claude_desktop_config.json`, Claude Code — `.claude.json`):**

```json
{
  "mcpServers": {
    "{app-name}": {
      "command": "python",
      "args": ["-m", "{app_name_snake}.server"],
      "env": {
        "OPENAI_API_KEY": "sk-..."
      }
    }
  }
}
```

Gotchas:
- **No stdout writes from tool bodies** — stdout is the protocol channel. Use `logging` (defaults to stderr).
- The launcher controls CWD. Use absolute paths, or resolve paths against `os.environ["DATA_DIR"]` so the server is portable.
- `env` in the client config is the easiest place to inject secrets; `.env` files are not auto-loaded.

## sse

```python
mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

**Client wiring:** point the MCP client at `http://localhost:8000/sse` (or wherever it's deployed).

Good for:
- Servers shared by multiple clients
- Deployment behind nginx / Cloud Run / ECS — SSE is boring HTTP
- Long generate-streaming responses

## streamable-http

```python
mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
```

Bidirectional streaming over HTTP. Pick this over SSE when the client needs to send tool responses mid-stream (rare) or when the deployment environment prefers a single endpoint over the SSE split.

## Auth

Neither SSE nor streamable-http authenticates by default. Add a header check via a FastMCP lifespan hook:

```python
import os
from contextlib import asynccontextmanager
from fastapi import HTTPException


EXPECTED_TOKEN = os.environ["MCP_BEARER_TOKEN"]


@asynccontextmanager
async def lifespan(server):
    @server.middleware("http")
    async def auth(request, call_next):
        auth = request.headers.get("authorization", "")
        if not auth.startswith("Bearer ") or auth.removeprefix("Bearer ") != EXPECTED_TOKEN:
            raise HTTPException(401, "unauthorized")
        return await call_next(request)
    yield
```

For stdio transport auth isn't meaningful — the client already has full process control.

## Environment variables

Every transport should read secrets from the environment, never hard-code. The canonical set:

| Var | Purpose |
|-----|---------|
| `OPENAI_API_KEY` (or LiteLLM equivalent) | LLM calls |
| `MCP_BEARER_TOKEN` | HTTP auth (SSE / streamable-http only) |
| `DATA_DIR` | Absolute path for vector-store data, cached indexes, etc. |

Document the required vars in the generated README so the user doesn't have to guess when the server refuses to start.
