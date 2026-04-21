# Ragbits RAG Specification

Advanced patterns for the ragbits `DocumentSearch` class. Read this when the base templates in SKILL.md aren't enough.

## Core Imports

Prefer the public re-exports from `ragbits.document_search` and `ragbits.core.vector_stores`. The packages surface everything you need from the root:

```python
from ragbits.core.embeddings.dense import LiteLLMEmbedder, LiteLLMEmbedderOptions
from ragbits.core.llms import LiteLLM
from ragbits.core.vector_stores.base import VectorStoreOptions
from ragbits.document_search import DocumentSearch, DocumentSearchOptions
from ragbits.document_search.documents.document import DocumentMeta
```

## DocumentSearch Constructor

```python
document_search = DocumentSearch(
    vector_store=vector_store,        # required — owns the embedder
    reranker=reranker,                # optional — see checklists/new-reranker.md
    query_rephraser=rephraser,        # optional — see checklists/new-rephraser.md
    parser=custom_parser,             # optional — override default parser selection
    enricher=custom_enricher,         # optional — post-process parsed elements
)
```

The embedder is attached to the **vector store**, not the `DocumentSearch`. This decoupling lets you swap stores without touching embedder config, and it mirrors the canonical ragbits examples.

## Ingestion

### From local files

```python
from ragbits.document_search.documents.document import DocumentMeta

documents = [
    DocumentMeta.from_local_path("documents/policy.pdf"),
    DocumentMeta.from_local_path("documents/handbook.docx"),
]
result = await document_search.ingest(documents)

print(result.successful)   # list[IngestedDocument]
print(result.failed)       # list[FailedDocument]
```

### From literal strings

Useful for tests, demos, and seed data:

```python
documents = [
    DocumentMeta.from_literal("Vacation policy: 20 days annually."),
    DocumentMeta.from_literal("Sick leave: up to 10 paid days per year."),
]
await document_search.ingest(documents)
```

### Supported formats

`DocumentMeta.from_local_path()` auto-detects format and dispatches to the appropriate parser (Docling or Unstructured by default). Works out of the box for:

- Text: `.txt`, `.md`, `.html`, `.json`, `.xml`, `.csv`, `.tsv`
- Office: `.pdf`, `.docx`, `.pptx`, `.xlsx`
- Email: `.eml`, `.msg`
- Images (with VLM): `.png`, `.jpg`, `.jpeg`, `.tiff`

If a format needs a specific parser, pass `parser=...` to the `DocumentSearch` constructor.

## Searching

### Basic

```python
results = await document_search.search("vacation policy")

for element in results:
    print(element.text_representation)
    print(element.document_meta.source)
```

`results` is a list of `Element` objects. Each element has `.text_representation` (str | None) and `.document_meta` (the source).

### With options

`DocumentSearchOptions` lets you tune the search per call without recreating the pipeline:

```python
from ragbits.core.vector_stores.base import VectorStoreOptions
from ragbits.document_search import DocumentSearchOptions

options = DocumentSearchOptions(
    vector_store_options=VectorStoreOptions(
        k=5,                    # number of results (default: 10 on most stores)
        score_threshold=0.7,    # drop anything below this cosine similarity
    ),
)
results = await document_search.search("vacation policy", options=options)
```

## Embedder Configuration

### LiteLLMEmbedder

Defaults work for OpenAI's `text-embedding-3-small`. Override dimensions or timeouts when needed:

```python
from ragbits.core.embeddings.dense import LiteLLMEmbedder, LiteLLMEmbedderOptions

embedder = LiteLLMEmbedder(
    model_name="text-embedding-3-small",
    default_options=LiteLLMEmbedderOptions(
        dimensions=1024,
        timeout=1000,
    ),
)
```

Any LiteLLM-supported embedding model works (OpenAI, Cohere, Voyage, Vertex, local via Ollama/LM Studio, etc.). Match the embedding model's **output dimensions** across ingest and query — mixing dimensions silently breaks retrieval.

## Vector Store Listing

Every vector store supports listing entries (useful for debugging):

```python
all_documents = await vector_store.list()
for doc in all_documents:
    print(doc.metadata["content"])
```

## Audit / Tracing

Enable CLI tracing to see retrieval internals during development:

```python
from ragbits.core.audit import set_trace_handlers

set_trace_handlers("cli")
```

Call this once at module load (usually at the top of `ingest.py` or a `__main__` entry point). Remove or gate behind an env var for production.

## Composing with Rerankers and Rephrasers

See `references/checklists/new-reranker.md` and `references/checklists/new-rephraser.md` for full setup. Both components plug into the `DocumentSearch` constructor:

```python
from ragbits.document_search.retrieval.rephrasers.llm import LLMQueryRephraser
from ragbits.document_search.retrieval.rerankers.litellm import LiteLLMReranker

llm = LiteLLM(model_name="gpt-4o-mini")

document_search = DocumentSearch(
    vector_store=vector_store,
    query_rephraser=LLMQueryRephraser(llm=llm),
    reranker=LiteLLMReranker(model="gpt-4o-mini"),
)
```

Rephrasers rewrite the user query before embedding; rerankers re-score the top-k chunks after retrieval. Use one, the other, or both — they are independent.

## Related Reference Files

- `references/checklists/new-vector-store.md` — backend selection + `get_document_search()` body
- `references/checklists/new-chat-ui.md` — `ChatInterface`, `RagbitsAPI`, streaming
- `references/checklists/new-auth.md` — authentication backends
- `references/checklists/new-feedback.md` — feedback forms
- `references/checklists/new-theme.md` — UI customization
- `references/checklists/new-reranker.md` — reranker wiring
- `references/checklists/new-rephraser.md` — query rephraser wiring
