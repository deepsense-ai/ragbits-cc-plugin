# New MCP Resource Checklist

`@mcp.resource(uri_template)` exposes read-only data fetched by URI. Resources are for *content the model consumes*, not actions — use `@mcp.tool()` when the operation has side effects or requires arguments beyond a URI.

## When to reach for resources

- Static reference material the client wants to pin into context ("the company style guide", "the schema of this table")
- Per-document fetches from a RAG corpus: resource URI `docs://{id}` → `DocumentSearch.get(id)`
- Environment snapshots (current config, feature flags) the client should inspect before answering

## Shape

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("{app-name}")


@mcp.resource("docs://{doc_id}")
async def get_doc(doc_id: str) -> str:
    """Fetch the full text of a document by ID."""
    return await backend.get_document_text(doc_id)


@mcp.resource("config://current")
async def current_config() -> dict:
    """The server's active configuration — model, index path, limits."""
    return backend.describe_config()
```

The URI template's `{placeholder}` names must match the function parameters. A static URI (no placeholders) works too.

## Quality gates

- [ ] Resource is truly read-only — no side effects, no LLM calls with cost
- [ ] URI scheme is stable and documented — clients cache against it
- [ ] Return type is `str`, `bytes`, or JSON-serializable — FastMCP handles serialization
- [ ] For per-document resources, the backing store supports efficient lookup by ID (not a full scan)

## When to use a tool instead

- The operation writes, sends, or charges — that's a tool
- The operation needs LLM synthesis (e.g. "summarize this doc") — make it a tool so the client knows it costs something
- The inputs don't fit naturally into a URI template — use a tool with arguments
