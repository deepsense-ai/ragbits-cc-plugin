# Ragbits Evaluate Specification

Advanced patterns for `ragbits-evaluate`. Read this when the base templates in SKILL.md aren't enough.

## Core Imports

```python
from ragbits.evaluate.evaluator import Evaluator
from ragbits.evaluate.optimizer import Optimizer
from ragbits.evaluate.dataset_generator import DatasetGenerationPipeline

# Abstract bases for custom components
from ragbits.evaluate.pipelines.base import EvaluationPipeline
from ragbits.evaluate.metrics.base import Metric
from ragbits.evaluate.dataloaders.base import DataLoader
```

## The Factory Config Pattern

Every pluggable component in `ragbits-evaluate` is declared as a `{"type": "...", "config": {...}}` dict. `type` is a fully-qualified import path with a colon between module and class; `config` is passed to the class constructor. This is the same pattern used across `ragbits-core` (vector stores, embedders) and `ragbits-document-search` (parsers, ingest strategies).

```python
config = {
    "evaluation": {
        "dataloader": {
            "type": "ragbits.evaluate.dataloaders.document_search:DocumentSearchDataLoader",
            "config": {
                "source": {
                    "type": "ragbits.core.sources:HuggingFaceSource",
                    "config": {"path": "deepsense-ai/synthetic-rag-dataset_v1.0", "split": "train"},
                },
            },
        },
        "pipeline": { "type": "...", "config": { ... } },
        "metrics":  { "precision_recall_f1": { "type": "...", "config": { ... } } },
    }
}
```

### Why the indirection

Three big wins:

1. **The optimizer can swap components without touching your code.** Mark a field with `"optimize": True` and a `"range"` and Optuna-backed `Optimizer` sweeps the space.
2. **The CLI can run the same config.** `ragbits evaluate run --target-factory-path ... --dataloader-factory-path ... --metrics-factory-path ...` reads factories the same way.
3. **Config can live in YAML/JSON.** The same dict shape serializes cleanly for checked-in experiment definitions.

## `Evaluator.run_from_config`

```python
results = await Evaluator.run_from_config(config)
```

Returns an object with:

- `results.metrics: dict[str, float]` — one entry per metric output. A single metric class can emit multiple keys (e.g., `DocumentSearchPrecisionRecallF1` emits `precision`, `recall`, `f1`).
- `results.time_perf.total_time_in_seconds: float` — end-to-end wall time
- `results.time_perf.samples_per_second: float` — throughput
- `results.time_perf.latency_in_seconds: float` — per-sample latency

Print these or log them. Downstream tooling (dashboards, CI gates) typically reads `results.metrics`.

## Built-in Components (v1.x)

### Pipelines
- `ragbits.evaluate.pipelines.document_search:DocumentSearchPipeline`
- `ragbits.evaluate.pipelines.question_answer:*` (QA pipelines)
- Custom → subclass `ragbits.evaluate.pipelines.base.EvaluationPipeline`

### Dataloaders
- `ragbits.evaluate.dataloaders.document_search:DocumentSearchDataLoader`
- `ragbits.evaluate.dataloaders.question_answer:*`
- All dataloaders accept a nested `source` factory — use `ragbits.core.sources:HuggingFaceSource`, local file sources, or your own.

### Metrics — Document Search
- `ragbits.evaluate.metrics.document_search:DocumentSearchPrecisionRecallF1` — precision, recall, F1 per retrieval
- `ragbits.evaluate.metrics.document_search:DocumentSearchRankedRetrievalMetrics` — rank-aware (MRR, NDCG-style)
- Both accept a `matching_strategy` config — `RougeChunkMatch` with a `threshold` is the canonical one

### Metrics — Question Answer (LLM-as-judge)
- `QuestionAnswerAnswerCorrectness` — answer vs. ground truth
- `QuestionAnswerAnswerFaithfulness` — answer grounded in retrieved context
- `QuestionAnswerAnswerRelevance` — answer relevant to the question
- `QuestionAnswerAnswerConsistency` — style consistency with reference

These scale cost linearly with dataset size — each sample is an LLM call against the judge model. Use `gpt-4o-mini` or `claude-3-5-haiku-*` for the judge unless precision is critical.

## `Optimizer.run_from_config`

Sweeps hyperparameters using Optuna. Config extends the evaluation config with an outer `optimizer` block and `"optimize": true` / `"range": [...]` markers on parameters:

```python
config = {
    "optimizer": {
        "direction": "maximize",
        "n_trials": 5,
        "max_retries_for_trial": 1,
    },
    "evaluation": {
        "pipeline": {
            "type": "...",
            "config": {
                "vector_store": {
                    "type": "...",
                    "config": {
                        "default_options": {
                            "k": {"optimize": true, "range": [1, 10]},
                            "score_threshold": {"optimize": true, "range": [-1.5, -0.5]},
                        },
                    },
                },
            },
        },
        # dataloader + metrics identical to evaluate.py
    },
}

configs_with_scores = Optimizer.run_from_config(config)
```

Returns a list of `(config, score)` pairs ranked by the objective. The utility `ragbits.evaluate.utils:log_optimization_to_file(configs_with_scores, Path("optimization"))` persists them for later inspection.

The objective score is the **first metric** in the metrics config by default. Put the metric you're optimizing for first, or configure the objective explicitly in the `optimizer` block.

## Dataset Generation

`DatasetGenerationPipeline` synthesizes QA pairs (or retrieval-style datasets) from raw documents, useful when no labeled test set exists. See `references/checklists/new-dataset-generator.md` for the wiring.

## The `ragbits evaluate run` CLI

```bash
ragbits evaluate run \
  --dataloader-factory-path my_eval.factories:build_dataloader \
  --target-factory-path     my_eval.factories:build_pipeline \
  --metrics-factory-path    my_eval.factories:build_metrics
```

Each `*-factory-path` points to a Python callable that returns the corresponding component. Factories are the preferred form when components are complex — they encapsulate wiring logic that would be awkward in a flat dict. If all three flags are omitted, the CLI reads project config (ragbits' own config file, typically `pyproject.toml`'s `[tool.ragbits]` section).

For the common case (single script, small config), sticking with `Evaluator.run_from_config(config)` in `evaluate.py` is simpler and more portable.

## Related Reference Files

- `references/checklists/new-document-search.md` — retrieval benchmarking setup
- `references/checklists/new-question-answer.md` — LLM-as-judge QA scoring
- `references/checklists/new-custom-pipeline.md` — subclassing `EvaluationPipeline`
- `references/checklists/new-custom-metric.md` — subclassing `Metric`
- `references/checklists/new-dataset-generator.md` — synthetic dataset generation
- `references/checklists/new-optimizer.md` — Optuna hyperparameter sweeps
