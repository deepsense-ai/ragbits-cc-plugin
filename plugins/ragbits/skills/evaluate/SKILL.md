---
name: evaluate
description: Scaffold a new ragbits evaluation project that measures RAG, agent, or QA quality using the ragbits-evaluate framework. Triggers whenever the user wants to create, build, or set up an evaluation harness, benchmark, metrics suite, LLM-as-judge scoring rig, retrieval precision/recall/F1 report, or hyperparameter sweep for a ragbits pipeline — including synthetic dataset generation, LLM-judge quality scoring, document-search retrieval benchmarks, and Optuna-based optimization. Use even for "evaluate my RAG pipeline", "benchmark how well my retrieval works", "set up LLM-as-judge scoring", "compare two embedding models on precision/recall", or "tune my vector store hyperparameters".
---

# Create a Ragbits Evaluation Project

Scaffold a complete, immediately runnable ragbits evaluation harness. Generate the full template first, install dependencies, then let the user point it at their real data and pipeline.

## Parse Arguments

Parse `$ARGUMENTS`:
- **app-name** (first positional, default: `my-eval`) — project directory name
- **description** (remaining free-form text, optional) — what is being evaluated; drives the target and metric selection
- **--target TARGET** (default: `document-search`) — evaluation target: `document-search`, `question-answer`, or `custom`
- **--judge-model MODEL** (default: `gpt-4o-mini`) — LLM used by judge-style metrics (QA correctness, faithfulness, relevance, consistency)
- **--embedding-model MODEL** (default: `text-embedding-3-small`) — used by the document-search target's evaluation pipeline
- **--features FEATURES** (comma-separated, optional) — any of `optimize`, `dataset-generator`, `custom-metric`, `custom-pipeline`

If `$ARGUMENTS` is empty and there is no prior context describing what to build, ask one message:

> "What should the evaluation project be called, and what are you measuring? (e.g. `retrieval-bench Compare two embedding models on our FAQ corpus`)"

Generate immediately after receiving an answer. Map natural-language cues to features and target without asking follow-ups:

- "precision", "recall", "F1", "retrieval quality", "chunks" → `--target document-search`
- "answer quality", "faithfulness", "groundedness", "LLM judge" → `--target question-answer`
- "my pipeline is unusual", "custom metric", "not covered by the built-ins" → `--target custom` (usually with `custom-metric` and/or `custom-pipeline` features)
- "tune", "sweep", "hyperparameter", "find the best config" → `optimize`
- "I don't have a test set yet", "generate synthetic data", "make me a dataset" → `dataset-generator`

For deeper domain questions (dataset provenance, metric selection, judge-model cost), see `references/interview-checklist.md`.

## Project Structure

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    └── evaluate.py           # Evaluator.run_from_config entry point
    └── optimize.py           # (optional — when `optimize` feature)
    └── generate.py           # (optional — when `dataset-generator` feature)
    └── pipeline.py           # (optional — when `custom-pipeline` feature)
    └── metric.py             # (optional — when `custom-metric` feature)
```

`{app_name_snake}` = app-name converted to snake_case (hyphens → underscores). Always write an empty `__init__.py` so the directory is a valid Python package.

## Generate Files

### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits-evaluate",
    "ragbits-core",
]
```

Append based on target and features:
- `--target document-search` (any) → add `"ragbits-document-search"`
- When the pipeline stores vectors in chroma/qdrant → add `"ragbits-core[chroma]"` or `"ragbits-core[qdrant]"`
- `optimize` feature → `ragbits-evaluate` already bundles Optuna; no extra needed unless a custom sampler is pulled in

### evaluate.py

Canonical pattern. `Evaluator.run_from_config(config)` is the single entry point. `config` is a nested `{"type": "...", "config": {...}}` factory dict — every component (dataloader, pipeline, metric, vector store, embedder) is referenced by its fully-qualified import path, not instantiated directly. That decoupling is what lets the optimizer and CLI swap components without touching the runner.

```python
"""
Ragbits evaluation: {one-line description}.
"""

import asyncio
import logging

from ragbits.evaluate.evaluator import Evaluator

logging.getLogger("LiteLLM").setLevel(logging.ERROR)
logging.getLogger("httpx").setLevel(logging.ERROR)


config = {
    "evaluation": {
        "dataloader": {
            {dataloader_block}
        },
        "pipeline": {
            {pipeline_block}
        },
        "metrics": {
            {metrics_block}
        },
    }
}


async def evaluate() -> None:
    print("Starting evaluation...")
    results = await Evaluator.run_from_config(config)

    print("\nMetrics:")
    for key, value in results.metrics.items():
        print(f"  {key}: {value}")

    print("\nTime performance:")
    print(f"  Total time:         {results.time_perf.total_time_in_seconds:.2f} s")
    print(f"  Samples per second: {results.time_perf.samples_per_second:.2f}")
    print(f"  Latency:            {results.time_perf.latency_in_seconds:.3f} s")


if __name__ == "__main__":
    asyncio.run(evaluate())
```

The `{dataloader_block}` / `{pipeline_block}` / `{metrics_block}` bodies come from the target-specific checklist:
- `--target document-search` → read `references/checklists/new-document-search.md`
- `--target question-answer` → read `references/checklists/new-question-answer.md`
- `--target custom` → read `references/checklists/new-custom-pipeline.md` and/or `new-custom-metric.md`

Always wire the **dataloader source** and **pipeline's test documents source** to placeholders the user can replace — `HuggingFaceSource` with `deepsense-ai/synthetic-rag-dataset_v1.0` is the canonical demo because it's real, small, and public.

### README.md

Include:
- What is being evaluated (from the description)
- Prerequisites (`OPENAI_API_KEY` for the embedder and/or judge model; whatever the target pipeline needs)
- Install: `pip install -e .`
- Run the evaluation: `python -m {app_name_snake}.evaluate` **or** `ragbits evaluate run --dataloader-factory-path ... --target-factory-path ... --metrics-factory-path ...`
- (When `optimize`) Run the sweep: `python -m {app_name_snake}.optimize`
- (When `dataset-generator`) Generate synthetic data: `python -m {app_name_snake}.generate`
- How to swap the dataloader source (HuggingFace dataset path, local file path, etc.)
- How to interpret the printed metrics
- Pointers to the per-component checklists

Always use **dots** in Python module paths (`my_eval.evaluate`), not slashes.

## Component Extensions

Read the relevant checklist before generating. Generate without asking follow-up questions.

| User mentions / flag                     | Read                                                    | What to add                                           |
|------------------------------------------|---------------------------------------------------------|-------------------------------------------------------|
| `--target document-search`               | `references/checklists/new-document-search.md`          | `DocumentSearchDataLoader` + `DocumentSearchPipeline` + precision/recall/F1 metric |
| `--target question-answer`               | `references/checklists/new-question-answer.md`          | QA dataloader + QA pipeline + LLM-as-judge metrics    |
| `--target custom` / `custom-pipeline`    | `references/checklists/new-custom-pipeline.md`          | `pipeline.py` subclassing `EvaluationPipeline`        |
| `custom-metric`                          | `references/checklists/new-custom-metric.md`            | `metric.py` subclassing `Metric`                      |
| `dataset-generator`                      | `references/checklists/new-dataset-generator.md`        | `generate.py` using `DatasetGenerationPipeline`       |
| `optimize`                               | `references/checklists/new-optimizer.md`                | `optimize.py` using `Optimizer.run_from_config`       |

For advanced patterns (results structure, time-performance fields, the ragbits `evaluate run` CLI, how factories resolve `type:` strings) see `references/evaluate-spec.md`.

## Install Dependencies

After generating all files, detect if working inside a local ragbits source repo:

```bash
find . -maxdepth 6 -name "ragbits-core" -type d 2>/dev/null | head -1
```

**If found (developing ragbits from source):**
```bash
pip install -e {path_to_repo}/packages/ragbits-core \
            -e {path_to_repo}/packages/ragbits-evaluate
```

Add matching local packages per target/feature:
- `--target document-search` → also `-e {path_to_repo}/packages/ragbits-document-search`

**Otherwise:**
```bash
cd {app-name} && pip install -e .
```

Wait for installation to complete. If it fails, show the error and suggest a fix.

## Summary

After files are generated and dependencies are installed, print a summary matching the selected shape:

```
Created {app-name}/
  {app_name_snake}/evaluate.py   — Evaluator.run_from_config entry point
  {optional extra files per feature}
  pyproject.toml                 — project + dependencies
  README.md                      — setup, run, how to read metrics

Target:   {document-search | question-answer | custom}
Features: {list of enabled features, or "none"}

Run:
  cd {app-name}
  python -m {app_name_snake}.evaluate

{If optimize:}  Run sweep:          python -m {app_name_snake}.optimize
{If dataset-gen:} Generate dataset: python -m {app_name_snake}.generate

Next steps:
  • Replace the demo HuggingFace dataset with your own source in evaluate.py
  • Swap the embedding / vector-store / pipeline config for your real components
  • Tune thresholds (score_threshold, k, matching strategy) once the baseline prints
  • Add more metrics: see references/checklists/new-{document-search,question-answer}.md
```

Mention the LLM judge cost caveat when `question-answer` metrics are used: every sample becomes an LLM call against `--judge-model`, which scales linearly with the dataset size.
