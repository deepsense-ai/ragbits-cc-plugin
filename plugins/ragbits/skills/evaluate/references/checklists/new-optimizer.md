# Optimizer Checklist

Quality gates for adding an Optuna-backed hyperparameter sweep on top of the evaluation config. `Optimizer.run_from_config` reads the same factory config, treats fields marked `"optimize": True` / `"range": [...]` as search dimensions, and returns a ranked list of `(config, score)` pairs.

## When to reach for this

- The evaluation already runs and prints a stable metric — now you want to find the best config
- Known knobs: retrieval `k`, `score_threshold`, embedding dimensions, chunk size, reranker model, batch size
- You can afford `n_trials × single_run_cost` in both time and LLM spend

Skip when the evaluation is already at target — optimizers find local maxima, not architectural improvements. If moving from BM25 to dense retrieval doubled the score, don't sweep `k`; investigate the architectural win first.

## Dependency

Optuna ships inside `ragbits-evaluate` — no extra install needed. Custom samplers or remote storage back-ends may need `optuna` extras; add on demand.

## Imports

```python
from pathlib import Path

from ragbits.evaluate.optimizer import Optimizer
from ragbits.evaluate.utils import log_optimization_to_file
```

## Minimal `optimize.py`

```python
"""
Optuna-driven hyperparameter sweep over the evaluation config.
"""

from pathlib import Path

from ragbits.evaluate.optimizer import Optimizer
from ragbits.evaluate.utils import log_optimization_to_file


config = {
    "optimizer": {
        "direction": "maximize",
        "n_trials": 5,
        "max_retries_for_trial": 1,
    },
    "evaluation": {
        # Copy the dataloader + metrics from evaluate.py verbatim.
        "dataloader": { ... },
        "pipeline": {
            "type": "ragbits.evaluate.pipelines.document_search:DocumentSearchPipeline",
            "config": {
                "vector_store": {
                    "type": "ragbits.core.vector_stores.chroma:ChromaVectorStore",
                    "config": {
                        "client": {"type": "EphemeralClient"},
                        "index_name": "sweep",
                        "distance_method": "l2",
                        "default_options": {
                            "k": {"optimize": True, "range": [1, 10]},
                            "score_threshold": {"optimize": True, "range": [-1.5, -0.5]},
                        },
                        "embedder": {
                            "type": "ragbits.core.embeddings.dense:LiteLLMEmbedder",
                            "config": {"model_name": "text-embedding-3-small"},
                        },
                    },
                },
                # ingest_strategy, parser_router, source stay as-is
            },
        },
        "metrics": { ... },
    },
}


def main() -> None:
    configs_with_scores = Optimizer.run_from_config(config)
    output_dir = log_optimization_to_file(configs_with_scores, Path("optimization"))
    print(f"Ranked trials written to: {output_dir}")


if __name__ == "__main__":
    main()
```

## Marking a Parameter for Sweeping

Replace the literal value with a dict:

```python
# Before
"k": 3

# After — sweep integer k ∈ [1, 10]
"k": {"optimize": True, "range": [1, 10]}
```

Integer / float ranges use `[min, max]`. Categorical dimensions use `{"optimize": True, "choices": ["option1", "option2"]}`. Nested fields work at any depth.

## Objective Score

The optimizer maximizes (or minimizes — set `direction` accordingly) the **first metric** in the `metrics` dict by default. Put the metric you care about first, or pin it explicitly in the `optimizer` block when that's supported.

For multi-metric optimization, reduce to a single scalar before sweeping (e.g., a custom metric that returns `0.7 * f1 + 0.3 * latency_penalty`). Optuna's multi-objective mode exists but complicates result interpretation — most teams get more mileage from a carefully-weighted scalar.

## Budgeting

- **Time**: `n_trials × per-trial evaluation wall time`. A 30s evaluation × 20 trials = 10 min. A 10 min evaluation × 20 trials = over 3 hours.
- **LLM cost**: if the evaluation uses LLM-as-judge metrics, multiply by `n_trials`. 1k sample × 3 judges × 20 trials = 60k judge calls.
- **Retries**: `max_retries_for_trial=1` means each failed trial retries once. Bump it when external APIs are flaky, leave at 1 for deterministic runs.

Start with `n_trials=5` as a smoke test. Once the mechanism works, step up to 20–50 for real optimization. Optuna's TPE sampler finds good regions in the first dozen trials — beyond 50 trials, returns diminish unless the search space is high-dimensional.

## Reading the Output

`log_optimization_to_file` writes:
- `optimization/{timestamp}/trials.json` — ranked list of `(config, score)` pairs
- Per-trial artifacts (logs, intermediate metrics)

Inspect the top-3 trial configs manually — sometimes the winner wins by 0.1% on a noisy metric and picks a config with a 2× latency cost. Look at the full score-vs-cost frontier, not just the max score.

## Promoting the Winner to `evaluate.py`

Once a good config is found, copy its values back into `evaluate.py` — strip the `{"optimize": ..., "range": ...}` dicts and leave literals. That pins the production config and makes it legible.

## Validation Checklist

- [ ] `optimize.py` lives next to `evaluate.py` and shares the same dataloader + metrics
- [ ] `optimizer.direction` matches the metric — `maximize` for F1/recall, `minimize` for latency/loss
- [ ] At least one field is marked `"optimize": True` — otherwise the run collapses to a single trial
- [ ] `n_trials` is sized against the budget; README surfaces the expected time + cost
- [ ] Output directory is gitignored (optimization artifacts can be large)
- [ ] The first metric in the `metrics` dict is the one being optimized (unless the objective is pinned explicitly)
- [ ] README documents how to promote a winning config back into `evaluate.py`
