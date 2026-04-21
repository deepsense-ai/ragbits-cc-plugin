# New Chat UI Checklist

Quality gates for generating `app.py` with the ragbits chat interface when the `chat-ui` feature is enabled.

## Dependency

`chat-ui` requires the `ragbits-chat` package. Append it to `pyproject.toml`:

```toml
dependencies = [
    "ragbits-core",                 # swap for [chroma]/[qdrant] as the vector store dictates
    "ragbits-document-search",
    "ragbits-chat",
]
```

## Definition Requirements

- [ ] A subclass of `ChatInterface` named `RAGChat` (or similar)
- [ ] `chat()` is an `async def ... -> AsyncGenerator[ChatResponse, None]` and **yields** (does not return) responses
- [ ] Retrieval happens inside `chat()` by awaiting `get_document_search()` from `ingest.py`
- [ ] Each retrieved element is emitted as a `self.create_reference(...)` chunk so the UI can show sources
- [ ] The LLM is streamed via `llm.generate_streaming(messages)` and chunks are yielded through `self.create_text_response(...)`
- [ ] `conversation_history = True` when multi-turn memory is useful
- [ ] The script exposes a `__main__` block that starts `RagbitsAPI(RAGChat).run()`

## Imports

```python
from collections.abc import AsyncGenerator

from ragbits.chat.interface import ChatInterface
from ragbits.chat.interface.types import ChatContext, ChatResponse
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat

from {app_name_snake}.ingest import get_document_search
```

Add more imports for optional features — see `new-auth.md`, `new-feedback.md`, `new-theme.md`.

## Canonical `RAGChat` Template

```python
SYSTEM_PROMPT = (
    "You are a helpful assistant that answers questions based on the provided context. "
    "Use the context below to answer the user's question. "
    "If the context doesn't contain relevant information, say so.\n\n"
    "Context:\n{context}"
)


class RAGChat(ChatInterface):
    """Ragbits RAG chat application with document search."""

    conversation_history = True

    # Optional: feedback_config = ...        # see new-feedback.md
    # Optional: ui_customization = ...       # see new-theme.md

    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")

    async def chat(
        self,
        message: str,
        history: ChatFormat,
        context: ChatContext,
    ) -> AsyncGenerator[ChatResponse, None]:
        document_search = await get_document_search()
        results = await document_search.search(message)

        rag_context = "\n\n---\n\n".join(
            element.text_representation
            for element in results
            if element.text_representation
        )

        for element in results:
            if not element.text_representation:
                continue
            source = "Document"
            if element.document_meta and element.document_meta.source:
                source = str(element.document_meta.source)
            preview = element.text_representation
            if len(preview) > 200:
                preview = preview[:200] + "..."
            yield self.create_reference(title=source, content=preview)

        messages: ChatFormat = [
            {"role": "system", "content": SYSTEM_PROMPT.format(context=rag_context)},
            *history,
            {"role": "user", "content": message},
        ]

        async for chunk in self.llm.generate_streaming(messages):
            yield self.create_text_response(chunk)
```

Write the `SYSTEM_PROMPT` to match the domain — generic templates produce generic answers.

## Main Block

Without auth:

```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(RAGChat).run()
```

With auth (see `new-auth.md` for the backend factory):

```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(RAGChat, auth_backend=get_auth_backend()).run()
```

## Running the App

Three equivalent ways (document all three in the README):

1. `ragbits api run {app_name_snake}.app:RAGChat` — preferred; picks up ragbits' CLI conventions
2. `python -m {app_name_snake}.app` — works anywhere Python does
3. `python {app_name_snake}/app.py` — direct script; only use when the filesystem path is more convenient than the module path

When `auth` is enabled, add `--auth` to the `ragbits api run` invocation and surface the test credentials.

## Advanced Yield Types

`ChatInterface` supports more than text and references:

| Helper                                       | Use for                                                             |
|----------------------------------------------|---------------------------------------------------------------------|
| `self.create_text_response(chunk)`           | Streaming LLM tokens                                                |
| `self.create_reference(title, content, url)` | Source citations shown alongside the answer                         |
| `self.create_state_update(dict)`             | Patch the client-side state (custom app features)                   |
| `self.create_image_response(id, url)`        | Inline images from tools or retrieval                               |
| `self.create_live_update(id, type, text)`    | Progress events ("Searching...", "Reranking...")                    |
| `self.create_followup_messages(list)`        | Suggested next questions                                            |

Reach for these when the base template isn't enough; the canonical patterns live in `examples/chat/tutorial.py` in the ragbits repo.

## Validation Checklist

- [ ] `pyproject.toml` contains `ragbits-chat`
- [ ] `app.py` imports `get_document_search` from the generated `ingest.py`, not a separate instance
- [ ] `chat()` yields references **before** streaming the answer, so the UI renders them above the reply
- [ ] The same system prompt is used in `app.py` and any companion `search.py`, or the divergence is intentional
- [ ] The `__main__` block matches the auth setup (present or absent)
- [ ] The README documents all three run commands and any `--auth` credentials
