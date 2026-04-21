---
name: create-chat
description: Scaffold a new ragbits chat application. Triggers whenever the user wants to create, build, or set up a conversational chat app, chatbot, web assistant, or streaming Q&A interface powered by the ragbits library — including simple chat UIs, login-protected bots, feedback-enabled assistants, branded chat surfaces, file-upload chats, tool-using agent chats, and RAG-backed document chats served via the ragbits web UI. Use even for "build me a ragbits chatbot with a web UI", "I need a branded chat assistant", or "set up a streaming chat that calls my tools".
---

# Create a Ragbits Chat Application

Scaffold a complete, immediately runnable ragbits chat application. Generate the full template first, install dependencies, then let the user customize.

## Parse Arguments

Parse `$ARGUMENTS`:
- **app-name** (first positional, default: `my-chat`) — project directory name
- **description** (remaining free-form text, optional) — what the chat should do; drives the system prompt and feature inference
- **--model MODEL** (default: `gpt-4o-mini`) — any LiteLLM-compatible model name
- **--features FEATURES** (comma-separated, optional) — any of `auth`, `agents`, `upload`, `feedback`, `theme`, `document-search`

If `$ARGUMENTS` is empty and there is no prior context describing what to build, ask one message:

> "What should the chat be called, and what will it do? (e.g. `support-bot A chat assistant that answers product questions`)"

Generate immediately after receiving an answer. Map natural-language cues to features without asking follow-ups:
- "login", "users", "accounts" → `auth`
- "tools", "calls my APIs", "takes actions" → `agents`
- "file upload", "attach documents", "drag and drop" → `upload`
- "thumbs up/down", "rating", "feedback forms" → `feedback`
- "branded UI", "logo", "customize look" → `theme`
- "search my docs", "answer from my knowledge base", "RAG" → `document-search`

For deeper domain questions (tone, starter flows, audience), see `references/interview-checklist.md`.

## Project Structure

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    └── app.py          # ChatInterface subclass + RagbitsAPI launcher
```

`{app_name_snake}` = app-name converted to snake_case (hyphens → underscores). Always write an empty `__init__.py` so the directory is a valid Python package.

Add `tools.py` (when `agents`), a `documents/` folder + `ingest.py` (when `document-search`), or upload storage scaffolding (when `upload`) per the relevant checklist.

## Generate Files

### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits-chat",
]
```

Append based on configuration (combine as needed):
- `agents` feature → add `"ragbits-agents"`
- `document-search` feature → add `"ragbits-document-search"` and `"ragbits-core"`

### app.py

Canonical pattern. `ChatInterface` is the base class; `chat()` is an async generator that **yields** typed responses (text chunks, references, live updates, images, follow-ups). `RagbitsAPI` wraps it and serves the built-in web UI.

```python
from collections.abc import AsyncGenerator

from ragbits.chat.interface import ChatInterface
from ragbits.chat.interface.types import ChatContext, ChatResponse
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat

# --- feature imports: add only what the selected features require ---
{feature_imports}
# --- end feature imports ---


{feature_level_definitions}


class Chat(ChatInterface):
    """Ragbits chat application."""

    conversation_history = True
    {optional_class_attributes}

    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")
        {optional_constructor_extras}

    async def chat(
        self,
        message: str,
        history: ChatFormat,
        context: ChatContext,
    ) -> AsyncGenerator[ChatResponse, None]:
        {chat_body}

    {optional_upload_handler}


if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    {ragbitsapi_invocation}
```

Fill `{...}` placeholders based on the selected features. The **base** chat body (no features) streams the LLM reply with conversation history:

```python
messages: ChatFormat = [*history, {"role": "user", "content": message}]
async for chunk in self.llm.generate_streaming(messages):
    yield self.create_text_response(chunk)
```

Write a domain-appropriate `SYSTEM_PROMPT` when the description is specific, and prepend it to `messages` as a `{"role": "system", "content": SYSTEM_PROMPT}` entry. Generic chats can stay prompt-less.

Notes on the template:
- `conversation_history = True` lets the built-in UI replay prior turns back into `history`. Set to `False` for stateless Q&A surfaces.
- `chat()` is an async generator. Each `yield self.create_*(...)` is a streaming event the UI renders incrementally.
- The ragbits CLI (`ragbits api run module:Class`) is the preferred launcher — it picks up logging, lifespan, and dev-mode niceties. `python -m ...` works too.

For advanced yield types (live updates, references, images, state, follow-ups, usage reports) see `references/chat-spec.md`.

### README.md

Include:
- What the chat does (from the description)
- Prerequisites (`OPENAI_API_KEY`, or the env var for the chosen model)
- Setup: `pip install -e .`
- Run — list all three methods:
  1. `ragbits api run {app_name_snake}.app:Chat` (preferred)
  2. `python -m {app_name_snake}.app`
  3. `python {app_name_snake}/app.py` (direct script execution)
- With `auth`: include the `--auth` flag and seed credentials
- With `upload`: note the expected file types and how uploads are handled
- With `document-search`: how to add documents and ingest them
- List of enabled features, customization pointers to the reference files

Always use **dots** in Python module paths (`my_chat.app`), not slashes. Slashes appear only when invoking a script file directly (`python path/to/app.py`).

## Component Extensions

Read the relevant checklist when the description or feature flag calls for it. Generate the component without asking follow-up questions.

| User mentions / flag              | Read                                             | What to add                                            | Dependency                |
|-----------------------------------|--------------------------------------------------|--------------------------------------------------------|---------------------------|
| `auth` feature                    | `references/checklists/new-auth.md`              | `ListAuthenticationBackend` factory + `__main__` wiring | `ragbits-chat`            |
| `feedback` feature                | `references/checklists/new-feedback.md`          | Pydantic forms + `FeedbackConfig`                      | `ragbits-chat`            |
| `theme` feature                   | `references/checklists/new-theme.md`             | `UICustomization` block                                | `ragbits-chat`            |
| `agents` feature                  | `references/checklists/new-agents.md`            | `Agent` + tools + streaming tool events                | `ragbits-agents`          |
| `upload` feature                  | `references/checklists/new-upload.md`            | `upload_handler(UploadFile)` + storage guidance        | `ragbits-chat` (bundles FastAPI) |
| `document-search` feature         | `references/checklists/new-document-search.md`   | `DocumentSearch` + `ingest.py` + reference-yielding chat | `ragbits-document-search` |

For advanced patterns (state updates, live-update types, follow-up messages, custom response classes, per-user settings) see `references/chat-spec.md`.

## Install Dependencies

After generating all files, detect if working inside a local ragbits source repo:

```bash
find . -maxdepth 6 -name "ragbits-core" -type d 2>/dev/null | head -1
```

**If found (developing ragbits from source):**
```bash
pip install -e {path_to_repo}/packages/ragbits-core \
            -e {path_to_repo}/packages/ragbits-chat
```

Add matching local packages per feature:
- `agents` → also `-e {path_to_repo}/packages/ragbits-agents`
- `document-search` → also `-e {path_to_repo}/packages/ragbits-document-search`

**Otherwise:**
```bash
cd {app-name} && pip install -e .
```

Wait for installation to complete. If it fails, show the error and suggest a fix.

## Summary

After files are generated and dependencies are installed, print a summary matching the selected shape:

```
Created {app-name}/
  {app_name_snake}/app.py      — ChatInterface subclass + launcher
  pyproject.toml               — project + dependencies
  README.md                    — setup, run, customization

Features: {list of enabled features, or "none"}

Run:
  cd {app-name}
  ragbits api run {app_name_snake}.app:Chat

Next steps:
  • Tune the system prompt in app.py for your domain
  • Add tools (with --features agents): see references/checklists/new-agents.md
  • Add login (with --features auth):    see references/checklists/new-auth.md
  • Brand the UI (with --features theme): see references/checklists/new-theme.md
```

Show the `--auth` login hint and seed credentials when `auth` is enabled. Add a documents/ingestion instruction block when `document-search` is enabled.
