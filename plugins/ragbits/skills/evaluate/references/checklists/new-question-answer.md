# Question-Answer Target Checklist

Quality gates for building an evaluation that scores **answer quality** using LLM-as-judge metrics — correctness vs. ground truth, faithfulness to retrieved context, relevance to the question, and style consistency.

## Dependency

```toml
dependencies = [
    "ragbits-evaluate",
    "ragbits-core",
]
```

Add `"ragbits-document-search"` when the QA pipeline performs retrieval before answering.

## Metric Inventory

All four live under `ragbits.evaluate.metrics.question_answer` and are LLM-as-judge:

| Metric class                            | Scores                                                                              |
|-----------------------------------------|-------------------------------------------------------------------------------------|
| `QuestionAnswerAnswerCorrectness`       | Is the answer factually correct relative to the ground-truth answer?                |
| `QuestionAnswerAnswerFaithfulness`      | Is the answer grounded in the retrieved context (vs. hallucinated)?                 |
| `QuestionAnswerAnswerRelevance`         | Does the answer actually address the question?                                      |
| `QuestionAnswerAnswerConsistency`       | Does the answer's style match the reference answer?                                 |

Pick the ones that matter. Most projects want `correctness` + `faithfulness` + `relevance`; `consistency` is narrower and only valuable when style adherence is a product requirement.

## Cost Warning

Each metric × each sample = one LLM call. A 500-sample dataset evaluated on three metrics = **1,500 judge calls**. Budget:

- Default `--judge-model gpt-4o-mini` — cheap enough for medium datasets
- `gpt-4o` / Claude Sonnet — use when judge accuracy is the bottleneck and the dataset is small
- Local models via `ollama/*` — free but judge quality varies widely; validate on a labeled sanity-check set first

Surface the expected call count in the README so users can size their runs.

## Canonical `config` Block

```python
config = {
    "evaluation": {
        "dataloader": {
            "type": "ragbits.evaluate.dataloaders.question_answer:QuestionAnswerDataLoader",
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
            "type": "ragbits.evaluate.pipelines.question_answer:QuestionAnswerWithContextPipeline",
            "config": {
                "llm": {
                    "type": "ragbits.core.llms:LiteLLM",
                    "config": {"model_name": "{answer_model}"},
                },
                # (when the pipeline performs retrieval, also supply vector_store + embedder here —
                #  same factory shape as in new-document-search.md)
            },
        },
        "metrics": {
            "answer_correctness": {
                "type": "ragbits.evaluate.metrics.question_answer:QuestionAnswerAnswerCorrectness",
                "config": {
                    "llm_judge": {
                        "type": "ragbits.core.llms:LiteLLM",
                        "config": {"model_name": "{judge_model}"},
                    },
                },
            },
            "answer_faithfulness": {
                "type": "ragbits.evaluate.metrics.question_answer:QuestionAnswerAnswerFaithfulness",
                "config": {
                    "llm_judge": {
                        "type": "ragbits.core.llms:LiteLLM",
                        "config": {"model_name": "{judge_model}"},
                    },
                },
            },
            "answer_relevance": {
                "type": "ragbits.evaluate.metrics.question_answer:QuestionAnswerAnswerRelevance",
                "config": {
                    "llm_judge": {
                        "type": "ragbits.core.llms:LiteLLM",
                        "config": {"model_name": "{judge_model}"},
                    },
                },
            },
        },
    }
}
```

### Dataset Shape

`QuestionAnswerDataLoader` expects samples with at minimum a `question` and a `ground_truth_answer` field (sometimes `reference_answer`). For faithfulness, it also uses `context` (the retrieved snippets). When wiring a custom source, ensure the field names line up — or write a lightweight wrapper that maps your columns onto what the dataloader expects.

### Answer vs. Judge Models

- **Answer model** (inside `pipeline.llm`) — the model under test. Usually the one the user's RAG/chat currently uses.
- **Judge model** (inside each metric's `llm_judge`) — the grader. Keep it **separate** from the answer model so you don't ask a model to grade its own output. Cheap fast models (`gpt-4o-mini`) usually suffice.

## Choosing the Judge Model

Disagreement with human judgment is the honest measure. In practice:

- `gpt-4o-mini` agrees with humans ~80–90% on correctness/relevance for straightforward QA
- `gpt-4o` buys a few more points at several-× the cost
- For niche domains (legal, medical, code), validate the judge against 20–50 human-scored samples before trusting the full run

## Validation Checklist

- [ ] `pyproject.toml` lists `ragbits-evaluate` and whatever the pipeline needs (`ragbits-core`, optionally `ragbits-document-search`)
- [ ] Dataset source is real and accessible; README documents which fields must exist
- [ ] Answer model and judge model are distinct — the judge is not scoring its own output
- [ ] README surfaces the expected call count (`dataset_size × metric_count`) and estimated cost
- [ ] Only the metrics the user actually cares about are wired — omitting `consistency` (and friends) keeps cost proportional to what the user will read
- [ ] `matching_strategy` / judge prompt thresholds (where surfaced) are called out in the README so future readers know what "good" means
