# Ragbits Chat Specification

Advanced patterns for the ragbits `ChatInterface` class. Read this when the base template in SKILL.md isn't enough.

## Core Imports

Prefer the public re-exports from `ragbits.chat.interface` and `ragbits.chat.interface.types`. Everything the chat surface needs lives here:

```python
from collections.abc import AsyncGenerator

from ragbits.chat.interface import ChatInterface
from ragbits.chat.interface.types import (
    ChatContext,
    ChatResponse,
    ErrorContent,
    ErrorResponse,
    LiveUpdateType,
    ResponseContent,
)
from ragbits.chat.api import RagbitsAPI
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat
```

## `ChatInterface` — Class Attributes

These are declared at the class level, not inside `__init__`, because `ChatInterface` inspects them before instantiation to configure the UI.

| Attribute              | Type                | Purpose                                                                     |
|------------------------|---------------------|-----------------------------------------------------------------------------|
| `conversation_history` | `bool`              | When `True`, prior turns are replayed to `chat()` via the `history` argument |
| `ui_customization`     | `UICustomization`   | Header, welcome message, starter questions, favicon — see `new-theme.md`    |
| `feedback_config`      | `FeedbackConfig`    | Like/dislike forms — see `new-feedback.md`                                  |
| `user_settings`        | `UserSettings`      | Per-user form rendered in the UI's settings pane                            |

## `chat()` — What to Yield

`chat()` is an `async def ... -> AsyncGenerator[ChatResponse, None]`. Each `yield self.create_*(...)` streams an event to the UI. Yield in the order the UI should render — typically **references before answers** so citations appear above the text.

| Helper                                                            | When to use                                                                                     |
|-------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| `self.create_text_response(chunk)`                                | Streaming LLM tokens                                                                            |
| `self.create_reference(title, content, url=None)`                 | Source citations (RAG chunks, retrieved documents)                                              |
| `self.create_state_update({...})`                                 | Patch the client's state — custom dashboards, progress counters                                 |
| `self.create_image_response(id, url)`                             | Inline images from tools, document pages, or generated art                                      |
| `self.create_live_update(id, type, title, detail=None)`           | Progress events ("Searching...", "Reranking...", "Calling weather API")                         |
| `self.create_followup_messages([...])`                            | Suggested next questions after the answer                                                       |
| `self.create_usage_response(usage)`                               | Report token usage back to the UI for observability                                             |
| `ErrorResponse(content=ErrorContent(message="..."))`              | Structured error the UI renders instead of a text reply                                         |

### LiveUpdateType

`create_live_update` takes a `LiveUpdateType`:

- `LiveUpdateType.START` — show a spinner/title for the operation
- `LiveUpdateType.FINISH` — mark the operation complete (optionally with a final detail)

Use the same `id` for `START` and `FINISH` so the UI pairs them. Tool calls are the canonical use case — see `new-agents.md` for the tool-streaming pattern.

## `chat()` Arguments

| Arg       | Type           | Contents                                                                            |
|-----------|----------------|-------------------------------------------------------------------------------------|
| `message` | `str`          | The current user turn                                                               |
| `history` | `ChatFormat`   | `list[dict]` of prior turns in OpenAI-style role/content format                     |
| `context` | `ChatContext`  | Session metadata — user id, conversation id, uploaded file refs, UI settings, etc. |

`history` is empty on the first turn when `conversation_history = True`, and always empty when `conversation_history = False`.

## Starting the Server

`RagbitsAPI` wraps a `ChatInterface` subclass into a FastAPI app with the built-in web UI:

```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(Chat).run()
```

With an auth backend:

```python
RagbitsAPI(Chat, auth_backend=get_auth_backend()).run()
```

The **preferred launcher** for scripted runs is the ragbits CLI:

```bash
ragbits api run my_chat.app:Chat
```

Pass `--auth my_chat.app:get_auth_backend` to enable auth via the CLI. The CLI handles dev-mode reloading, structured logging, and the `--host`/`--port` flags.

## Multi-turn Conversation

When `conversation_history = True`, rebuild the messages list with `history` prepended:

```python
messages: ChatFormat = [
    {"role": "system", "content": SYSTEM_PROMPT},
    *history,
    {"role": "user", "content": message},
]
```

The system prompt should be constant across turns — don't vary it based on the latest message, or the UI's replay will drift.

## Combining Features

Features compose — a single chat can have `auth`, `theme`, `feedback`, `agents`, and `document-search` at once. Rough layering:

1. **Class attributes** (top of class): `conversation_history`, `ui_customization`, `feedback_config`, `user_settings`
2. **Constructor** (`__init__`): instantiate `self.llm`, `self.agent`, `self.document_search`, or anything else the `chat()` body uses
3. **`chat()` body**: retrieve context, yield references, stream the answer, emit live updates for long operations
4. **`upload_handler`**: separate method for uploads — see `new-upload.md`
5. **`__main__`**: `RagbitsAPI(Chat, auth_backend=...)` when auth is enabled, plain `RagbitsAPI(Chat)` otherwise

Keep the class focused. If `chat()` grows past ~50 lines, extract helper methods (`_retrieve_context`, `_build_messages`) — the readability win compounds when layering features.

## Related Reference Files

- `references/checklists/new-auth.md` — `ListAuthenticationBackend`, seed users, `--auth` flag
- `references/checklists/new-feedback.md` — `LikeForm`/`DislikeForm`, `FeedbackConfig`, `handle_feedback`
- `references/checklists/new-theme.md` — `UICustomization` header, welcome, starter questions, favicon
- `references/checklists/new-agents.md` — `Agent`, tools, streaming `ToolCall`/`ToolCallResult` into live updates
- `references/checklists/new-upload.md` — `upload_handler`, `UploadFile`, storage patterns
- `references/checklists/new-document-search.md` — `DocumentSearch` wiring, reference yielding, ingestion script
