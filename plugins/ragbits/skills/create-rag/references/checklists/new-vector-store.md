# New Vector Store Checklist

Quality gates for picking and wiring a vector store into the generated `ingest.py`.

## Backend Selection

| Backend       | Persistence | Install extra            | When to pick                                            |
|---------------|-------------|--------------------------|---------------------------------------------------------|
| `in-memory`   | None        | (core only)              | Demos, tests, small datasets re-ingested every run      |
| `chroma`      | Local dir   | `ragbits-core[chroma]`   | Single-user prototypes, local knowledge bases           |
| `qdrant`      | File or server | `ragbits-core[qdrant]` | Larger corpora, hybrid search, production workloads     |

`pgvector` and `weaviate` are also supported by ragbits — see the upstream examples when those are requested.

## Dependency Wiring

The generated `pyproject.toml` lists `ragbits-core` by default. When the vector store requires an extra, replace the plain entry with the extra form:

```toml
dependencies = [
    "ragbits-core[chroma]",          # or [qdrant]
    "ragbits-document-search",
]
```

## `get_document_search()` — In-Memory

`InMemoryVectorStore` is ephemeral. The factory must cache the `DocumentSearch` and auto-ingest once, so the app does not start empty on every process launch.

```python
from ragbits.core.vector_stores.in_memory import InMemoryVectorStore

_document_search: DocumentSearch | None = None


async def get_document_search() -> DocumentSearch:
    """Return a cached DocumentSearch, auto-ingesting documents on first call."""
    global _document_search
    if _document_search is not None:
        return _document_search

    embedder = LiteLLMEmbedder(model_name="{embedding_model}")
    vector_store = InMemoryVectorStore(embedder=embedder)
    document_search = DocumentSearch(vector_store=vector_store{extra_kwargs})

    documents = [
        DocumentMeta.from_local_path(p)
        for p in DOCUMENTS_DIR.iterdir()
        if p.is_file()
    ]
    if documents:
        await document_search.ingest(documents)

    _document_search = document_search
    return document_search
```

`{extra_kwargs}` is empty by default; append `, reranker=reranker` or `, query_rephraser=rephraser` if those features are enabled.

## `get_document_search()` — Chroma

```python
import chromadb
from ragbits.core.vector_stores.chroma import ChromaVectorStore


async def get_document_search() -> DocumentSearch:
    """Create a DocumentSearch backed by a persistent local Chroma store."""
    embedder = LiteLLMEmbedder(model_name="{embedding_model}")
    chroma_client = chromadb.PersistentClient(path=".chroma_data")
    vector_store = ChromaVectorStore(
        client=chroma_client,
        index_name="{app_name_snake}",
        embedder=embedder,
    )
    return DocumentSearch(vector_store=vector_store{extra_kwargs})
```

No cache, no auto-ingest — the user runs `python -m {app_name_snake}.ingest` once and the data survives restarts. Document the workflow in the README.

## `get_document_search()` — Qdrant

```python
from qdrant_client import AsyncQdrantClient
from ragbits.core.vector_stores.qdrant import QdrantVectorStore


async def get_document_search() -> DocumentSearch:
    """Create a DocumentSearch backed by a local Qdrant store."""
    embedder = LiteLLMEmbedder(model_name="{embedding_model}")
    qdrant_client = AsyncQdrantClient(path=".qdrant_data")
    vector_store = QdrantVectorStore(
        client=qdrant_client,
        index_name="{app_name_snake}",
        embedder=embedder,
    )
    return DocumentSearch(vector_store=vector_store{extra_kwargs})
```

For a remote Qdrant server, swap `path=".qdrant_data"` for `url="http://host:6333"` and (optionally) `api_key=...`. For an entirely in-memory Qdrant, use `AsyncQdrantClient(location=":memory:")` — it behaves like `in-memory` without the cache/auto-ingest trick.

## Validation Checklist

- [ ] `pyproject.toml` lists the correct `ragbits-core` extra for the selected backend
- [ ] `ingest.py` imports only the vector store it uses (no dead imports)
- [ ] `get_document_search()` matches the template for its backend — in-memory caches + auto-ingests, persistent stores do neither
- [ ] `{app_name_snake}` is used as `index_name` so collections don't collide across apps
- [ ] The README tells the user how to re-ingest on document changes (and that in-memory re-ingests automatically)
- [ ] Embedding model in ingestion matches the one used at query time — always; changing it invalidates stored vectors
