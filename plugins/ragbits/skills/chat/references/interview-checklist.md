# Interview Checklist

Domain-specific questions that help produce a better system prompt, feature selection, and UI tone.
Use these when the user provides minimal context. Combine into one message — do not ask one at a time.

## Core (always relevant)

- **App name**: Directory/module name? (snake_case preferred)
- **Purpose**: What should the chat do and who is the audience? (1–3 sentences)
- **Model**: Which LLM? (default: `gpt-4o-mini`; any LiteLLM-compatible model works — `claude-*`, `gemini/*`, `ollama/*`, etc.)

## Conversation shape

- Multi-turn (remembers prior messages) or single-turn Q&A? → sets `conversation_history`
- System prompt — what persona, tone, constraints?
- Starter questions to surface in the UI? (2–4 realistic examples tied to the domain)
- Should the bot refuse out-of-scope requests, or always attempt an answer?

## Tools and actions (→ `agents`)

Ask if the chat needs to **do** things, not just reply:
- Which external APIs does it call? (weather, CRM, search, internal service)
- Does it read/write files, run queries, or call webhooks?
- Are any actions destructive — delete, send, pay — and need confirmation before execution?
- Should tool calls surface progress in the UI ("Calling X...") or stay silent?

## Authentication (→ `auth`)

- Who uses this — internal team, anonymous web visitors, invited customers?
- Do you need per-user history, or is each session disposable?
- SSO / OAuth required, or is a simple username/password list acceptable for now?
- Role-based access (admin/user) or flat?

## Feedback (→ `feedback`)

- Should users rate answers? (thumbs up/down)
- What fields should the follow-up form capture? (reason, issue type, free-text)
- Where should feedback be persisted? (webhook, database, log file)

## Theming (→ `theme`)

- Title, subtitle, and logo (emoji or image URL)?
- Welcome message shown before the first turn?
- Favicon and browser tab title?
- Any visual brand guidelines to respect?

## File upload (→ `upload`)

- Which file types are accepted? (PDF, images, CSV, any)
- Size limits?
- Where should uploads be stored? (local disk, S3, parsed and discarded)
- Should the chat automatically react to the upload, or wait for the user's next message?

## Document search (→ `document-search`)

- What kind of documents and how many? (internal wiki, product manuals, meeting notes)
- Vector store preference? (`in-memory` for demos; `chroma`/`qdrant` for persistence)
- How often do documents change — one-off ingest, daily, live?
- Should the UI show citation chips linked back to the source, or keep answers clean?
- See `plugins/ragbits/skills/rag/` for the RAG-first skill when document search is the primary purpose (rather than a chat add-on)

## Deployment

- Where will this run? (local dev, container, managed service)
- Which env vars need to be set? (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, internal API tokens)
- Any rate limits, cost caps, or SLA constraints?
- Reverse proxy, TLS, or CORS requirements?

These feed directly into the `SYSTEM_PROMPT`, `ui_customization`, and auth/feedback configuration. A specific domain-grounded system prompt meaningfully outperforms the generic template.
