# New Query Rephraser Checklist

Quality gates for adding an LLM-based query rephraser to the generated `DocumentSearch`.

A rephraser rewrites the user's question before it is embedded and sent to the vector store. Use it when retrieval misses results because of terse, keyword-like, or ambiguous queries — the rephraser expands them into fuller, more embedder-friendly forms.

## Dependency

Rephrasers live in `ragbits-document-search`, already listed in the generated `pyproject.toml`. No extras are needed.

## Imports

Add to `ingest.py`:

```python
from ragbits.core.llms import LiteLLM
from ragbits.document_search.retrieval.rephrasers.llm import LLMQueryRephraser
```

## Wiring into `get_document_search()`

After the vector store is built:

```python
llm = LiteLLM(model_name="{model}")
rephraser = LLMQueryRephraser(llm=llm)

document_search = DocumentSearch(
    vector_store=vector_store,
    query_rephraser=rephraser,
    # reranker=... (when the reranker feature is also enabled)
)
```

Reuse the answer-generation LLM when possible. If the answer model is expensive, drop to a cheaper model for the rephrasing step — the task is lightweight.

## Model Choice Guidance

- `gpt-4o-mini` / `claude-3-5-haiku-*` — almost always sufficient; rephrasing is short and structured
- Stronger models rarely pay off here; spend the budget on answer generation or reranking instead

## Combined with Rerankers

Rephrasers and rerankers are additive and independent:

```python
document_search = DocumentSearch(
    vector_store=vector_store,
    query_rephraser=LLMQueryRephraser(llm=llm),
    reranker=LiteLLMReranker(model="{model}"),
)
```

Pipeline order is: rephrase → embed → retrieve top-k → rerank → return.

## When Not to Use a Rephraser

- Queries are already full sentences ("What's the vacation policy for new hires?") — rephrasing yields little.
- Latency-sensitive flows where a synchronous LLM call before retrieval is too slow.
- Multilingual corpora where the rephrasing LLM may drift languages unexpectedly. Pin the output language in the prompt or skip the rephraser.

## Validation Checklist

- [ ] `ingest.py` imports `LLMQueryRephraser` and `LiteLLM` only when the feature is enabled
- [ ] `DocumentSearch(...)` receives `query_rephraser=rephraser`
- [ ] Model choice is documented in the README alongside the answer-generation model
- [ ] Rephraser and reranker, when both enabled, share the same constructor call
- [ ] Latency impact is called out in the README so users know retrieval becomes LLM-dependent
