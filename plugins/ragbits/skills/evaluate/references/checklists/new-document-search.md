# Document Search Target Checklist

Quality gates for building an evaluation that measures retrieval quality (precision / recall / F1 / rank-aware metrics) over a document-search pipeline.

## Dependency

Append to `pyproject.toml`:

```toml
dependencies = [
    "ragbits-evaluate",
    "ragbits-core",
    "ragbits-document-search",
]
```

Add the vector-store extra when the evaluated pipeline uses a persistent store: `"ragbits-core[chroma]"` or `"ragbits-core[qdrant]"`.

## Canonical `config` Block

```python
config = {
    "evaluation": {
        "dataloader": {
            "type": "ragbits.evaluate.dataloaders.document_search:DocumentSearchDataLoader",
            "config": {
                "source": {
                    "type": "ragbits.core.sources:HuggingFaceSource",
                    "config": {
                        "path": "deepsense-ai/synthetic-rag-dataset_v1.0",
                        "split": "train",
                    },
                },
            },
        },
        "pipeline": {
            "type": "ragbits.evaluate.pipelines.document_search:DocumentSearchPipeline",
            "config": {
                "vector_store": {
                    "type": "ragbits.core.vector_stores.chroma:ChromaVectorStore",
                    "config": {
                        "client": {"type": "EphemeralClient"},
                        "index_name": "baseline",
                        "distance_method": "l2",
                        "default_options": {"k": 3, "score_threshold": -1.2},
                        "embedder": {
                            "type": "ragbits.core.embeddings.dense:LiteLLMEmbedder",
                            "config": {"model_name": "{embedding_model}"},
                        },
                    },
                },
                "ingest_strategy": {
                    "type": "ragbits.document_search.ingestion.strategies.batched:BatchedIngestStrategy",
                    "config": {"batch_size": 10},
                },
                "parser_router": {
                    "txt": {
                        "type": "ragbits.document_search.ingestion.parsers.unstructured:UnstructuredDocumentParser",
                    },
                },
                "source": {
                    "type": "ragbits.core.sources:HuggingFaceSource",
                    "config": {
                        "path": "micpst/hf-docs",
                        "split": "train[:5]",
                    },
                },
            },
        },
        "metrics": {
            "precision_recall_f1": {
                "type": "ragbits.evaluate.metrics.document_search:DocumentSearchPrecisionRecallF1",
                "config": {
                    "matching_strategy": {
                        "type": "RougeChunkMatch",
                        "config": {"threshold": 0.5},
                    },
                },
            },
        },
    }
}
```

### Anatomy

| Block                   | What it is                                                                            |
|-------------------------|---------------------------------------------------------------------------------------|
| `dataloader.source`     | Where the **test queries + ground-truth chunks** come from                            |
| `pipeline.source`       | Where the **documents being searched** come from (ingested into the vector store)     |
| `pipeline.vector_store` | The retrieval backend being evaluated — swap to compare configs                       |
| `pipeline.ingest_strategy` | Controls how documents are embedded and inserted (batching, parallelism)           |
| `metrics.*`             | One or more scoring functions applied to retrieval output                             |

The two `source` blocks are distinct: the dataloader source carries labeled queries, while the pipeline source is the corpus to search. In the canonical demo above, a synthetic RAG dataset provides the queries and Hugging Face docs are the corpus. Real projects swap both for the user's own data.

## Adding Rank-Aware Metrics

To measure whether the right chunk appears at the top of the result list (not just anywhere in the top-k), add a second metric entry:

```python
"metrics": {
    "precision_recall_f1": { ... },
    "ranked_retrieval": {
        "type": "ragbits.evaluate.metrics.document_search:DocumentSearchRankedRetrievalMetrics",
        "config": {
            "matching_strategy": {
                "type": "RougeChunkMatch",
                "config": {"threshold": 0.5},
            },
        },
    },
},
```

`ranked_retrieval` reports MRR- / NDCG-style numbers on top of the precision/recall basics.

## Matching Strategies

`RougeChunkMatch` with a threshold of ~0.5 is the canonical default — it treats a retrieval as correct when the ROUGE overlap with any ground-truth chunk meets the threshold. Alternatives live alongside it in `ragbits.evaluate.metrics.document_search` (exact match, LLM-judge match); pick the one that matches how "correct" is defined in the user's domain.

Threshold tuning: start at 0.5. If results look too generous, lift to 0.7. If nothing ever matches, drop to 0.3 and inspect a few samples manually to see whether ground truth really is present in retrieved chunks.

## Validation Checklist

- [ ] `pyproject.toml` lists `ragbits-evaluate`, `ragbits-core`, and `ragbits-document-search`
- [ ] Vector-store extra is declared when the pipeline uses chroma/qdrant
- [ ] Both `dataloader.source` and `pipeline.source` point at real, accessible datasets (the template's `deepsense-ai/synthetic-rag-dataset_v1.0` + `micpst/hf-docs` work out of the box as a smoke test)
- [ ] `matching_strategy.threshold` is documented in the README so future readers understand what "correct" means
- [ ] At least one metric is declared under `metrics`; when rank matters, `ranked_retrieval` is also included
- [ ] README tells the user how to swap the dataloader source for their own test set
