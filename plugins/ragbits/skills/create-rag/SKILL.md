---
name: create-rag
description: Scaffold a new ragbits RAG (Retrieval-Augmented Generation) application. Triggers whenever the user wants to create, build, or set up a RAG app, document search, knowledge base, question-answering system, or document-grounded chat powered by the ragbits library — including CLI search, web-based chat UIs, persistent vector stores, auth-protected knowledge bases, and reranker/rephraser-enhanced pipelines. Use even for "build a ragbits RAG over my docs", "I need a knowledge base with web UI", or "set up a document Q&A bot".
---

# Create a Ragbits RAG Application

Scaffold a complete, immediately runnable ragbits RAG application. Generate the full template first, install dependencies, then let the user customize.

## Parse Arguments

Parse `$ARGUMENTS`:
- **app-name** (first positional, default: `my-rag`) — project directory name
- **description** (remaining free-form text, optional) — domain and goals; drives the system prompt and feature inference
- **--model MODEL** (default: `gpt-4o-mini`) — any LiteLLM-compatible model for answer generation
- **--embedding-model MODEL** (default: `text-embedding-3-small`) — embedding model for documents and queries
- **--vector-store STORE** (default: `in-memory`) — one of `in-memory`, `chroma`, `qdrant`
- **--features FEATURES** (comma-separated, optional) — any of `chat-ui`, `auth`, `feedback`, `theme`, `reranker`, `rephraser`

If `$ARGUMENTS` is empty and there is no prior context describing what to build, ask one message:

> "What should the RAG app be called, and what knowledge does it serve? (e.g. `policy-qa A Q&A bot over HR policy PDFs`)"

Generate immediately after receiving an answer. Map natural-language cues to features without asking follow-ups:
- "web UI", "chat interface", "chatbot" → `chat-ui`
- "login", "users", "accounts" → `auth` (implies `chat-ui`)
- "thumbs up/down", "rating" → `feedback` (implies `chat-ui`)
- "branded UI", "logo", "customize look" → `theme` (implies `chat-ui`)
- "better results", "rerank" → `reranker`
- "query expansion", "rewrite queries" → `rephraser`

For deeper domain questions (document formats, access patterns, scale), see `references/interview-checklist.md`.

## Project Structure

```
{app-name}/
├── pyproject.toml
├── README.md
├── documents/
│   └── sample.md
└── {app_name_snake}/
    ├── __init__.py
    ├── ingest.py        # DocumentSearch factory + CLI ingestion entry point
    └── search.py        # CLI Q&A loop          (if NOT chat-ui)
    └── app.py           # ChatInterface + API   (if chat-ui)
```

`{app_name_snake}` = app-name converted to snake_case (hyphens → underscores). Always write an empty `__init__.py` so the directory is a valid Python package.

## Generate Files

### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits-core",
    "ragbits-document-search",
]
```

Append based on configuration (combine as needed):
- `chat-ui` feature → add `"ragbits-chat"`
- `--vector-store chroma` → replace `ragbits-core` with `"ragbits-core[chroma]"`
- `--vector-store qdrant` → replace `ragbits-core` with `"ragbits-core[qdrant]"`

### documents/sample.md

Copy `assets/sample.md` (bundled with this skill, located next to `SKILL.md`) into `{app-name}/documents/sample.md`. This gives the user something ingestible on first run.

### ingest.py

Canonical pattern. `DocumentSearch` owns ingestion and retrieval; the embedder lives on the vector store. `DocumentMeta.from_local_path()` handles PDF, DOCX, MD, HTML, and the other 20+ formats ragbits parses automatically.

```python
import asyncio
from pathlib import Path

from ragbits.core.embeddings.dense import LiteLLMEmbedder
from ragbits.document_search import DocumentSearch
from ragbits.document_search.documents.document import DocumentMeta

# --- vector store import: pick exactly one block based on --vector-store ---
{vector_store_imports}
# --- end vector store import ---

DOCUMENTS_DIR = Path(__file__).parent.parent / "documents"


{get_document_search_definition}


async def ingest_documents() -> None:
    """Ingest all files from the documents/ directory."""
    document_search = await get_document_search()

    documents = [
        DocumentMeta.from_local_path(p)
        for p in DOCUMENTS_DIR.iterdir()
        if p.is_file()
    ]
    if not documents:
        print("No documents found in the documents/ directory.")
        return

    print(f"Ingesting {len(documents)} document(s)...")
    result = await document_search.ingest(documents)
    print(f"Successfully ingested {len(result.successful)} document(s).")
    if result.failed:
        print(f"Failed to ingest {len(result.failed)} document(s).")


if __name__ == "__main__":
    asyncio.run(ingest_documents())
```

See `references/checklists/new-vector-store.md` for the exact `get_document_search()` body per backend. The key detail: `InMemoryVectorStore` is ephemeral, so the in-memory variant must cache the `DocumentSearch` in a module-level variable and auto-ingest on first call — otherwise `search.py`/`app.py` would start empty every time. Persistent stores (`chroma`, `qdrant`) skip both the cache and the auto-ingest.

If the user selected `--features reranker` or `--features rephraser`, wire those into the DocumentSearch constructor following `references/checklists/new-reranker.md` or `references/checklists/new-rephraser.md`.

### search.py — when NOT chat-ui

CLI loop that retrieves relevant chunks and asks the LLM to answer grounded in them.

```python
import asyncio

from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat

from {app_name_snake}.ingest import get_document_search


SYSTEM_PROMPT = (
    "You are a helpful assistant that answers questions based on the provided context. "
    "Use the context below to answer the user's question. "
    "If the context doesn't contain relevant information, say so.\n\n"
    "Context:\n{context}"
)


async def search_and_answer(query: str) -> str:
    document_search = await get_document_search()
    llm = LiteLLM(model_name="{model}")

    results = await document_search.search(query)
    if not results:
        return "No relevant documents found for your query."

    context = "\n\n---\n\n".join(
        element.text_representation
        for element in results
        if element.text_representation
    )

    messages: ChatFormat = [
        {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
        {"role": "user", "content": query},
    ]
    return await llm.generate(messages)


async def main() -> None:
    print("{App Name}")
    print("=" * 40)
    print("Type 'quit' to exit.\n")

    while True:
        query = input("Question: ").strip()
        if query.lower() in ("quit", "exit", "q"):
            break
        if not query:
            continue
        print("\nSearching...")
        answer = await search_and_answer(query)
        print(f"\nAnswer: {answer}\n")


if __name__ == "__main__":
    asyncio.run(main())
```

Fill `{App Name}` with a user-facing title derived from the description. Write the `SYSTEM_PROMPT` so it reflects the actual domain when the description is specific — the generic template is a starting point, not a destination.

### app.py — when chat-ui

Build on `ChatInterface` with streaming answers, yielded references, and conversation history. The full reference is `references/checklists/new-chat-ui.md`; it also documents how to layer on `auth`, `feedback`, and `theme` when those features are requested.

### README.md

Include:
- What the RAG app does (from the description)
- Prerequisites (`OPENAI_API_KEY`, or the env var for the chosen model / embedding model)
- Install: `pip install -e .`
- Place documents in `documents/` (PDF, DOCX, MD, TXT, HTML, etc.)
- Ingest: `python -m {app_name_snake}.ingest` (persistent stores only — in-memory auto-ingests)
- Run:
  - NOT chat-ui: `python -m {app_name_snake}.search`
  - chat-ui: `ragbits api run {app_name_snake}.app:RAGChat` (preferred) **or** `python -m {app_name_snake}.app`
  - With `auth`: include test credentials and note the `--auth` flag
- Vector store info, enabled features, how to swap backends
- Pointers to the reference files for adding rerankers/rephrasers/customization

Always use **dots** in Python module paths (`my_rag.app`), not slashes. Slashes appear only when invoking a script file directly (`python path/to/app.py`).

## Component Extensions

Read the relevant checklist when the description or feature flag calls for it. Generate the component without asking follow-up questions.

| User mentions / flag               | Read                                             | What to add                                      | Dependency                |
|------------------------------------|--------------------------------------------------|--------------------------------------------------|---------------------------|
| vector store selection             | `references/checklists/new-vector-store.md`     | `get_document_search()` body + imports           | `ragbits-core[chroma\|qdrant]` as needed |
| `chat-ui` feature                  | `references/checklists/new-chat-ui.md`          | `app.py` with `ChatInterface` + `RagbitsAPI`     | `ragbits-chat`            |
| `auth` feature                     | `references/checklists/new-auth.md`             | `ListAuthenticationBackend` factory in `app.py`  | `ragbits-chat`            |
| `feedback` feature                 | `references/checklists/new-feedback.md`         | Pydantic forms + `FeedbackConfig`                | `ragbits-chat`            |
| `theme` feature                    | `references/checklists/new-theme.md`            | `UICustomization` block                          | `ragbits-chat`            |
| `reranker` feature                 | `references/checklists/new-reranker.md`         | `LiteLLMReranker` wired into `DocumentSearch`    | `ragbits-document-search` |
| `rephraser` feature                | `references/checklists/new-rephraser.md`        | `LLMQueryRephraser` wired into `DocumentSearch`  | `ragbits-document-search` |

For advanced patterns (custom parsers, `DocumentSearchOptions`, hybrid search, multimodal, external parser providers) see `references/rag-spec.md`.

## Install Dependencies

After generating all files, detect if working inside a local ragbits source repo:

```bash
find . -maxdepth 6 -name "ragbits-core" -type d 2>/dev/null | head -1
```

**If found (developing ragbits from source):**
```bash
pip install -e {path_to_repo}/packages/ragbits-core \
            -e {path_to_repo}/packages/ragbits-document-search
```

Add matching local packages per feature:
- `chat-ui` → also `-e {path_to_repo}/packages/ragbits-chat`
- `--vector-store chroma` → `-e "{path_to_repo}/packages/ragbits-core[chroma]"`
- `--vector-store qdrant` → `-e "{path_to_repo}/packages/ragbits-core[qdrant]"`

**Otherwise:**
```bash
cd {app-name} && pip install -e .
```

Wait for installation to complete. If it fails, show the error and suggest a fix.

## Summary

After files are generated and dependencies are installed, print a summary matching the selected shape:

```
Created {app-name}/
  {app_name_snake}/ingest.py   — DocumentSearch factory + ingestion CLI
  {app_name_snake}/{search.py|app.py}   — {CLI Q&A loop | chat UI}
  documents/sample.md          — sample document (replace with yours)
  pyproject.toml               — project + dependencies
  README.md                    — setup, run, customization

Vector store: {selected store}
Features:     {list of enabled features, or "none"}

Run:
  cd {app-name}
  {if persistent store} python -m {app_name_snake}.ingest
  {CLI:}     python -m {app_name_snake}.search
  {chat-ui:} ragbits api run {app_name_snake}.app:RAGChat

Next steps:
  • Drop PDFs/DOCX/MD into documents/ and re-ingest
  • Tune the system prompt in {search.py|app.py} for your domain
  • Add a reranker: see references/checklists/new-reranker.md
  • Add a query rephraser: see references/checklists/new-rephraser.md
  • Customize the chat UI: see references/checklists/new-theme.md
```

Mention the in-memory auto-ingest caveat when the in-memory store is used, and show the `--auth` login hint when `auth` is enabled.
