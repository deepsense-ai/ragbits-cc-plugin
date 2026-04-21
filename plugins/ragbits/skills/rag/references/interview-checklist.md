# Interview Checklist

Domain-specific questions that help produce a better system prompt, vector store choice, and feature set.
Use these when the user provides minimal context. Combine into one message — do not ask one at a time.

## Core (always relevant)

- **App name**: Directory/module name? (snake_case preferred)
- **Purpose**: What knowledge does the RAG serve and what questions should it answer? (1–3 sentences)
- **LLM**: Answer-generation model? (default: `gpt-4o-mini`; any LiteLLM-compatible model works)
- **Embedding model**: (default: `text-embedding-3-small`; also `text-embedding-3-large`, `voyage-*`, local via `ollama/*`, etc.)

## Documents

- Which formats? (PDF, DOCX, MD, HTML, CSV, TXT — ragbits auto-parses 20+ formats)
- How many documents roughly, and how often do they change?
- Any access restrictions, PII, or sensitive content?
- Multilingual? If so, pick an embedding model that supports the target languages.

## Vector Store

Pick based on persistence and scale:
- **in-memory** — demos, tests, small datasets re-ingested every process start. No install overhead.
- **chroma** — local persistence to a folder. Good for single-user knowledge bases and prototypes.
- **qdrant** — local file store, in-memory, or remote server. Best when the dataset grows or you need hybrid search.

Ask if the user expects the data to survive restarts. "Yes" rules out `in-memory`.

## Retrieval Quality

- Is retrieval quality already good enough, or does the user suspect weak recall?
  - If weak recall: suggest `rephraser` (expands queries into multiple variants before embedding).
  - If noisy top-k: suggest `reranker` (LLM re-scores candidates after retrieval).
  - Both are additive — use either, both, or neither.
- Typical question style? (keyword-like vs. full-sentence) — longer natural-language queries benefit more from rephrasing.

## Delivery Surface

- **CLI only** — quickest loop, no web framework. Good for internal tools and pipelines.
- **Chat UI** — served via `ragbits api run`, exposes streaming, references, conversation history.

If chat-ui, ask:
- Should users log in? → `auth`
- Should users rate answers? → `feedback`
- Branded look (logo, title, starter questions)? → `theme`

## Deployment

- Where will this run? (local dev, container, managed service)
- Which env vars need to be set? (`OPENAI_API_KEY`, `QDRANT_URL`, etc.)
- Any rate limits, cost caps, or SLA constraints?

## Answer Generation Style

- Short factual answers vs. longer explanations with citations?
- Should the bot refuse when context is insufficient, or always attempt an answer?
- Tone and persona (formal/casual, first-person/third-person)?

These feed directly into the `SYSTEM_PROMPT` string in `search.py` / `app.py`. A specific domain-grounded system prompt meaningfully outperforms the generic template.
