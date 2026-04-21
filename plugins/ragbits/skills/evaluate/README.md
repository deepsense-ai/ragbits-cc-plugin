# ragbits:evaluate

Skill that scaffolds a ragbits evaluation project — complete, immediately runnable, built on the canonical `Evaluator.run_from_config` entry point. Ships with targets for document-search and question-answer out of the box, plus optional hooks for custom pipelines, custom metrics, synthetic dataset generation, and Optuna-driven hyperparameter sweeps.

## What it generates

- `pyproject.toml` with `ragbits-evaluate` + `ragbits-core` (plus `ragbits-document-search` when the target is document-search)
- `evaluate.py` with a factory-style `config` dict and `Evaluator.run_from_config(config)` that prints `results.metrics` and time performance
- `README.md` with setup, run instructions, how to swap the dataset source, and how to read the metrics
- Optional: `optimize.py` (Optuna sweep), `generate.py` (synthetic dataset), `pipeline.py` / `metric.py` (custom components)

## Example: document-search precision/recall benchmark

Invoking the skill with:

```
/evaluate retrieval-bench Compare two embedding models on our FAQ corpus --target document-search
```

should produce a `retrieval-bench/` project with:

- `ragbits-evaluate`, `ragbits-core`, `ragbits-document-search` in `pyproject.toml`
- `evaluate.py` with a `DocumentSearchDataLoader` sourced from a HuggingFace dataset, a `DocumentSearchPipeline` using `ChromaVectorStore` + `LiteLLMEmbedder`, and a `DocumentSearchPrecisionRecallF1` metric with a `RougeChunkMatch` strategy
- A README explaining how to swap in the real corpus + labeled queries and how to interpret precision / recall / F1 / timing output

## Example: LLM-as-judge QA scoring with optimization

```
/evaluate answer-bench Grade our support bot's answers for correctness and faithfulness --target question-answer --judge-model gpt-4o-mini --features optimize
```

produces a project with correctness + faithfulness + relevance metrics, an Optuna sweep that tunes the retrieval `k` and `score_threshold` against the first metric, and a README that surfaces the expected LLM call count so users can budget the run.

## When to pick a different skill

- **Building the RAG pipeline itself** (not evaluating one) → use `ragbits:rag`
- **Building a chat UI** → `ragbits:chat`
- **Building a headless agent** → `ragbits:agent`

## References

| File                                                    | Purpose                                                                 |
|---------------------------------------------------------|-------------------------------------------------------------------------|
| `references/evaluate-spec.md`                           | Factory config pattern, `Evaluator` results shape, CLI, `Optimizer`     |
| `references/interview-checklist.md`                     | Domain questions that drive target and metric selection                 |
| `references/checklists/new-document-search.md`          | `DocumentSearchDataLoader` + `DocumentSearchPipeline` + precision/recall/F1 |
| `references/checklists/new-question-answer.md`          | QA dataloader + QA pipeline + LLM-as-judge metrics, with cost guidance  |
| `references/checklists/new-custom-pipeline.md`          | Subclassing `EvaluationPipeline` for non-standard workloads             |
| `references/checklists/new-custom-metric.md`            | Subclassing `Metric` for exact-match, regex, or bespoke LLM-judge scoring |
| `references/checklists/new-dataset-generator.md`        | `DatasetGenerationPipeline` for synthetic test data                     |
| `references/checklists/new-optimizer.md`                | Optuna-backed hyperparameter sweeps via `Optimizer.run_from_config`     |
