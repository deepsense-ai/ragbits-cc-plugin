# New Document Search Checklist

Quality gates for adding RAG (document search) to the generated chat application.

When document search is the **primary** purpose of the app (not an add-on to a general chat), prefer the `rag` skill — it scaffolds a richer layout with a dedicated `ingest.py` factory, vector-store selection, optional rerankers and rephrasers, and deeper reference docs. This checklist covers the case where chat is primary and document-grounded answers are one capability among several.

## Dependency

Append to `pyproject.toml`:

```toml
dependencies = [
    "ragbits-chat",
    "ragbits-document-search",
    "ragbits-core",
]
```

For persistent vector stores add the extra: `"ragbits-core[chroma]"` or `"ragbits-core[qdrant]"`.

## Imports

```python
from ragbits.core.embeddings.dense import LiteLLMEmbedder
from ragbits.core.vector_stores.in_memory import InMemoryVectorStore
from ragbits.document_search import DocumentSearch
from ragbits.document_search.documents.document import DocumentMeta
```

Swap the vector store import for `ChromaVectorStore` or `QdrantVectorStore` when persistence is required — see `plugins/ragbits/skills/rag/references/checklists/new-vector-store.md` for the per-backend setup.

## Minimal Pattern — In-Memory with Auto-Ingest

`InMemoryVectorStore` is ephemeral. Cache the `DocumentSearch` on the class (or in a module-level variable) so the app doesn't start empty on every request:

```python
from pathlib import Path

DOCUMENTS_DIR = Path(__file__).parent.parent / "documents"


class Chat(ChatInterface):
    conversation_history = True

    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")
        self._document_search: DocumentSearch | None = None

    async def _get_document_search(self) -> DocumentSearch:
        if self._document_search is not None:
            return self._document_search

        embedder = LiteLLMEmbedder(model_name="text-embedding-3-small")
        vector_store = InMemoryVectorStore(embedder=embedder)
        document_search = DocumentSearch(vector_store=vector_store)

        documents = [
            DocumentMeta.from_local_path(p)
            for p in DOCUMENTS_DIR.iterdir()
            if p.is_file()
        ]
        if documents:
            await document_search.ingest(documents)

        self._document_search = document_search
        return document_search
```

## Persistent Stores

For Chroma or Qdrant, follow the per-backend recipes in `plugins/ragbits/skills/rag/references/checklists/new-vector-store.md`. The key difference: persistent stores skip the cache and the auto-ingest, so the user runs a one-time ingestion script. Add `ingest.py` alongside `app.py`:

```python
# ingest.py
import asyncio
from {app_name_snake}.app import Chat


async def main() -> None:
    chat = Chat()
    await chat._get_document_search()  # triggers ingestion for in-memory
    print("Ingestion complete.")


if __name__ == "__main__":
    asyncio.run(main())
```

For Chroma/Qdrant, factor `_get_document_search` into a standalone function so `ingest.py` can call it without instantiating the full `Chat`.

## Yielding References

Retrieve context, emit references **before** streaming the answer so they render above the reply, then stream the grounded answer:

```python
SYSTEM_PROMPT = (
    "You are a helpful assistant that answers questions based on the provided context. "
    "Use the context below. If it doesn't contain relevant information, say so.\n\n"
    "Context:\n{context}"
)


async def chat(
    self,
    message: str,
    history: ChatFormat,
    context: ChatContext,
) -> AsyncGenerator[ChatResponse, None]:
    document_search = await self._get_document_search()
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

## Sample Document

Copy `assets/sample.md` (bundled with `rag`) or a domain-appropriate placeholder into `{app-name}/documents/sample.md` so the user has something ingestible on first run. When this skill is invoked directly and no bundled sample exists, generate a short placeholder document that matches the app's stated domain.

## Combining with `agents`, `feedback`, `auth`, `theme`

All additive:
- `agents`: give the agent a `search_docs` tool wrapping the `DocumentSearch` — lets the LLM decide when to retrieve rather than retrieving every turn
- `feedback`: users can mark whether citations were helpful — feeds into retrieval tuning
- `auth`: per-user document access (filter `results` by user role)
- `theme`: starter questions drawn from the corpus ("What's our vacation policy?")

## Validation Checklist

- [ ] `pyproject.toml` lists `ragbits-document-search` and `ragbits-core` (plus the vector-store extra when persistent)
- [ ] `_get_document_search` caches on the instance for in-memory stores
- [ ] `chat()` yields references **before** streaming the answer so the UI renders them above the reply
- [ ] A `SYSTEM_PROMPT` instructs the LLM to ground answers in the context and refuse when context is insufficient
- [ ] `documents/` folder exists with at least one sample file
- [ ] README documents how to add documents and (for persistent stores) how to re-ingest
- [ ] If document search is the primary feature, suggest the `rag` skill instead — it's purpose-built for this shape
