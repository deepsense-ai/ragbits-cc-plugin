# New Upload Checklist

Quality gates for adding file upload support to the generated chat application.

## Dependency

Upload support uses FastAPI's `UploadFile`, which ships with `ragbits-chat`. No extra install.

## Imports

```python
from fastapi import UploadFile
```

## `upload_handler` Method

Add an `upload_handler` method to your `Chat` class. `ragbits-chat` wires the UI's drag-and-drop / attach button to this method automatically.

```python
class Chat(ChatInterface):
    conversation_history = True

    # ... ui_customization, feedback_config, __init__, chat() ...

    async def upload_handler(self, file: UploadFile) -> None:
        """Persist or process an uploaded file."""
        content = await file.read()
        print(f"Received file: {file.filename}, size: {len(content)} bytes")
```

`file.filename`, `file.content_type`, and `await file.read()` are the common fields. The handler receives one file per call — the UI calls it once per attached file.

## Storage Patterns

The template prints the upload metadata as a placeholder. Swap in one of these patterns based on what the user needs:

### 1. Local disk

```python
from pathlib import Path

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)


async def upload_handler(self, file: UploadFile) -> None:
    target = UPLOAD_DIR / file.filename
    target.write_bytes(await file.read())
```

Good for local dev and single-node deployments. Not suitable for horizontally-scaled services — uploads land on whichever replica handled the request.

### 2. Parse and store in-memory state

When the chat should **use** the uploaded file immediately (e.g., "summarize this PDF"), parse it and attach to session state:

```python
from ragbits.document_search.documents.document import DocumentMeta


async def upload_handler(self, file: UploadFile) -> None:
    content = await file.read()
    # stash raw bytes or parsed text on self for the next chat() turn
    self._pending_upload = (file.filename, content)
```

Then in `chat()`, pop `self._pending_upload` and work it into the prompt. Works best for single-session scratch work.

### 3. Push to object storage

```python
import boto3

s3 = boto3.client("s3")


async def upload_handler(self, file: UploadFile) -> None:
    s3.upload_fileobj(file.file, "my-bucket", f"uploads/{file.filename}")
```

Production-grade. Pair with a signed URL you hand back via `self.create_reference(...)` on the next chat turn.

## Surface Uploads in `chat()`

The UI shows the attached file, but you may also want to acknowledge it in the reply. The `ChatContext` passed into `chat()` typically includes the uploaded file references once `upload_handler` has run, letting you reference them on the following turn:

```python
async def chat(self, message: str, history: ChatFormat, context: ChatContext):
    if context.uploaded_files:
        for uf in context.uploaded_files:
            yield self.create_reference(title=uf.filename, content="Attached by user")
    # ... normal chat flow ...
```

Exact field name depends on the ragbits version — check `ChatContext` in your installed package if the attribute isn't present.

## Accepted File Types

Restrict types in `upload_handler` rather than trusting the UI:

```python
ALLOWED_TYPES = {"application/pdf", "text/plain", "text/markdown"}


async def upload_handler(self, file: UploadFile) -> None:
    if file.content_type not in ALLOWED_TYPES:
        raise ValueError(f"Unsupported content type: {file.content_type}")
    # ... proceed ...
```

The UI renders the error and the user retries with a different file.

## When the Upload Is the Whole Point

If the app is **primarily** document-grounded Q&A rather than chat with optional uploads, prefer the `document-search` feature (see `new-document-search.md`) or spin up a dedicated RAG via `create-rag`. `upload` is for ad-hoc attachments; `document-search` is for a curated corpus.

## Validation Checklist

- [ ] `upload_handler` is `async def` and takes a single `UploadFile` parameter
- [ ] Uploads are read with `await file.read()` (not synchronous `file.file.read()`)
- [ ] Content type or extension is validated before storage — never trust the client
- [ ] Size limits are documented or enforced (`UploadFile` doesn't enforce a default)
- [ ] Storage location (disk, S3, in-memory) matches the deployment model
- [ ] The README tells the user which file types are accepted and how uploads are used
