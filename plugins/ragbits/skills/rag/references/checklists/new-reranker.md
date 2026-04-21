# New Reranker Checklist

Quality gates for adding an LLM-based reranker to the generated `DocumentSearch`.

A reranker re-scores the top-k results returned by the vector store using an LLM. It trades latency and token cost for higher precision — worth it when users complain that "the right chunk is in the corpus but the answer doesn't use it."

## Dependency

Rerankers live in `ragbits-document-search`, already listed in the generated `pyproject.toml`. No extras are needed.

## Imports

Add to `ingest.py`:

```python
from ragbits.core.llms import LiteLLM
from ragbits.document_search.retrieval.rerankers.litellm import LiteLLMReranker
```

## Wiring into `get_document_search()`

Inside the factory, after the vector store is built:

```python
llm = LiteLLM(model_name="{model}")
reranker = LiteLLMReranker(model="{model}")

document_search = DocumentSearch(
    vector_store=vector_store,
    reranker=reranker,
    # query_rephraser=... (when the rephraser feature is also enabled)
)
```

Use the **same model** as the answer generator by default — it keeps the dependency list short. For higher quality, pick a stronger model (e.g. `gpt-4o`) at the cost of more latency.

## Model Choice Guidance

- `gpt-4o-mini` / `claude-3-5-haiku-*` — fast, cheap, usually enough for ranking top-10 chunks
- `gpt-4o` / `claude-sonnet-*` — higher precision; consider when retrieval quality is the blocker
- Local models via `ollama/*` — viable when you already run one for answer generation

The reranker only sees the candidate snippets, not the full documents. A small fast model performs well here because the task is relative comparison, not synthesis.

## When Not to Use a Reranker

- The vector store already returns the right chunk near the top — rerankers add no value and cost latency.
- The corpus is tiny (say, under a few hundred chunks) — the top-k is often already the full corpus.
- Tight latency budgets — the extra LLM call typically adds 500ms–2s.

Suggest skipping the reranker in those cases even if the user named the feature.

## Validation Checklist

- [ ] `ingest.py` imports `LiteLLMReranker` and `LiteLLM` only when the reranker feature is enabled
- [ ] `DocumentSearch(...)` receives `reranker=reranker`
- [ ] The reranker model is documented in the README (users need to know which key to set)
- [ ] If the user also enabled `rephraser`, both are passed to `DocumentSearch` in the same constructor call
- [ ] The chosen model supports the required env var (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.)
