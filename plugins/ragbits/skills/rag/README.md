# ragbits:rag

Skill that scaffolds a new ragbits RAG (Retrieval-Augmented Generation) application — complete, immediately runnable, with the canonical `LiteLLMEmbedder` + vector store + `DocumentSearch` pattern, wired either to a CLI Q&A loop or to a `ChatInterface` web UI.

## What it generates

- `pyproject.toml` with the right `ragbits-core` / `ragbits-document-search` / `ragbits-chat` packages (plus the vector-store extra when needed)
- `ingest.py` with a backend-appropriate `get_document_search()` factory and a CLI ingestion entry point
- `search.py` with an interactive CLI Q&A loop (when `chat-ui` is **not** selected), or
- `app.py` with a `RAGChat(ChatInterface)` subclass, streaming answers, yielded references, and a `RagbitsAPI` launcher (when `chat-ui` is selected)
- `documents/sample.md` so the app has something ingestible on first run
- `README.md` with setup, run, and customization instructions
- Optional auth factory, feedback forms, UI customization, reranker, or query rephraser — added only when the user's arguments/description call for them

## Example: HR policy knowledge base with chat UI

Invoking the skill with:

```
/ragbits:rag policy-qa A Q&A bot over our HR policy PDFs with a web UI, login, and like/dislike feedback --vector-store chroma
```

should produce a `policy-qa/` project with:

- `ragbits-core[chroma]` + `ragbits-document-search` + `ragbits-chat` in `pyproject.toml`
- `ingest.py` using `ChromaVectorStore` with a persistent local collection named `policy_qa`
- `app.py` with a `RAGChat` that streams answers, yields per-chunk references, wears a domain-appropriate `ui_customization`, carries a `feedback_config` with `LikeForm`/`DislikeForm`, and launches via `RagbitsAPI(RAGChat, auth_backend=get_auth_backend()).run()`
- A README showing how to ingest, how to launch with `ragbits api run ... --auth ...`, and the seed credentials

## References

| File                                               | Purpose                                                                 |
|----------------------------------------------------|-------------------------------------------------------------------------|
| `references/rag-spec.md`                           | Advanced `DocumentSearch` patterns — options, tracing, composition      |
| `references/interview-checklist.md`                | Domain questions for a more accurate system prompt & feature selection  |
| `references/checklists/new-vector-store.md`        | Backend selection and `get_document_search()` body per store            |
| `references/checklists/new-chat-ui.md`             | `ChatInterface`, streaming, references, `RagbitsAPI`                    |
| `references/checklists/new-auth.md`                | Authentication backends and wiring                                      |
| `references/checklists/new-feedback.md`            | Like/dislike form models and `FeedbackConfig`                           |
| `references/checklists/new-theme.md`               | UI customization (header, welcome, starter questions, favicon)          |
| `references/checklists/new-reranker.md`            | LLM reranker wiring and when to use it                                  |
| `references/checklists/new-rephraser.md`           | Query rephraser wiring and when to use it                               |
