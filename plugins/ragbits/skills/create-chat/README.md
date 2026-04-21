# ragbits:create-chat

Skill that scaffolds a new ragbits chat application — complete, immediately runnable, with the canonical `ChatInterface` + `LiteLLM` + `RagbitsAPI` pattern, and optional layers for auth, feedback, theming, tool-using agents, file upload, and document search.

## What it generates

- `pyproject.toml` with `ragbits-chat` (plus `ragbits-agents` / `ragbits-document-search` when those features are selected)
- `app.py` with a `Chat(ChatInterface)` subclass, streaming LLM replies, and a `RagbitsAPI` launcher
- `README.md` with setup, three equivalent run commands, and customization pointers
- Optional: `tools.py` (for `agents`), `documents/` + ingestion wiring (for `document-search`), auth factory + seed credentials (for `auth`), feedback form models (for `feedback`), `UICustomization` block (for `theme`), and `upload_handler` (for `upload`)

## Example: branded support chat with login and feedback

Invoking the skill with:

```
/create-chat support-bot A product-support assistant that answers common questions --features auth,feedback,theme
```

should produce a `support-bot/` project with:

- `ragbits-chat` in `pyproject.toml`
- `app.py` with a `Chat` class that carries a domain-appropriate `ui_customization`, a `feedback_config` with `LikeForm`/`DislikeForm`, and launches via `RagbitsAPI(Chat, auth_backend=get_auth_backend()).run()`
- A `get_auth_backend()` factory with two seed users and an `InMemorySessionStore`
- A README showing all three run commands, `--auth` flag, and the seed credentials flagged as demo-only

## When to pick a different skill

- **Document search is the primary purpose** (not a chat add-on) → use `ragbits:create-rag`. It ships with vector-store selection, a dedicated ingest script, and optional reranker/rephraser features.
- **Building a headless agent** (CLI, cron job, A2A server, orchestrator) without a web UI → use `ragbits:create-agent`.

## References

| File                                                    | Purpose                                                                 |
|---------------------------------------------------------|-------------------------------------------------------------------------|
| `references/chat-spec.md`                               | Advanced `ChatInterface` patterns — yield types, live updates, state    |
| `references/interview-checklist.md`                     | Domain questions for a more accurate system prompt & feature selection  |
| `references/checklists/new-auth.md`                     | Authentication backends, seed users, `--auth` flag                      |
| `references/checklists/new-feedback.md`                 | Like/dislike forms and `FeedbackConfig`                                 |
| `references/checklists/new-theme.md`                    | UI customization (header, welcome, starter questions, favicon)          |
| `references/checklists/new-agents.md`                   | `Agent`, tools, streaming `ToolCall`/`ToolCallResult` as live updates   |
| `references/checklists/new-upload.md`                   | `upload_handler`, storage patterns, content-type validation             |
| `references/checklists/new-document-search.md`          | `DocumentSearch`, reference yielding, ingestion wiring                  |
