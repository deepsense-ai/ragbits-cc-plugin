# Custom Metric Checklist

Quality gates for writing a scoring function that isn't covered by the built-in precision/recall/F1 or LLM-as-judge metrics. Use this when you need exact-match scoring, regex matches, BLEU/ROUGE on specific fields, a domain-specific rubric, or an LLM judge with a custom prompt.

## When to reach for this

Stay with built-ins when they fit — `DocumentSearchPrecisionRecallF1` with a `RougeChunkMatch` strategy covers most retrieval cases, and the four QA judge metrics cover most answer-quality needs. Write a custom metric when:

- The scoring rule is objective and cheap (string equality, regex, set overlap, numeric tolerance) — no reason to pay for an LLM judge
- You want an LLM judge with a bespoke prompt that encodes domain rules the built-in judges don't know
- You need a metric that combines multiple fields in a non-standard way (e.g., "correct AND under 200 words AND cites at least one source")

## Imports

```python
from ragbits.evaluate.metrics.base import Metric
```

## Minimal Custom Metric

Write it in `metric.py`:

```python
"""Custom metric that {one-line description of what it scores}."""

from ragbits.evaluate.metrics.base import Metric

from {app_name_snake}.pipeline import Result  # the dataclass your pipeline emits


class ExactAnswerMatch(Metric[Result]):
    """Fraction of samples where the predicted answer equals the reference (case-insensitive)."""

    def __init__(self, case_sensitive: bool = False) -> None:
        super().__init__()
        self.case_sensitive = case_sensitive

    async def compute(self, results: list[Result]) -> dict[str, float]:
        if not results:
            return {"exact_match": 0.0}

        hits = 0
        for r in results:
            predicted = r.answer if self.case_sensitive else r.answer.lower()
            expected = r.reference if self.case_sensitive else r.reference.lower()
            if predicted.strip() == expected.strip():
                hits += 1
        return {"exact_match": hits / len(results)}
```

### What's important

- `Metric[ResultT]` is generic over the pipeline's output type — the metric can only run against a pipeline whose `Result` it understands.
- `compute` takes the **full list** of per-sample results and returns a `dict[str, float]`. A single metric class can emit multiple keys — that's how `DocumentSearchPrecisionRecallF1` returns precision, recall, and f1 together.
- Keep per-sample work inside the loop; return aggregated numbers.

## LLM-Judge Custom Metric

When the scoring rule needs reasoning, instantiate an LLM in the metric and grade each sample:

```python
from ragbits.core.llms import LiteLLM


class ConcisenessScore(Metric[Result]):
    """LLM judge scoring how concise each answer is (0.0–1.0)."""

    JUDGE_PROMPT = (
        "Grade the answer on conciseness from 0.0 (rambling) to 1.0 (perfectly tight). "
        "Respond with only a number.\n\n"
        "Question: {question}\nAnswer: {answer}"
    )

    def __init__(self, judge_model: str = "gpt-4o-mini") -> None:
        super().__init__()
        self.judge = LiteLLM(model_name=judge_model)

    async def compute(self, results: list[Result]) -> dict[str, float]:
        if not results:
            return {"conciseness": 0.0}

        total = 0.0
        for r in results:
            response = await self.judge.generate(
                [{"role": "user", "content": self.JUDGE_PROMPT.format(
                    question=r.question, answer=r.answer,
                )}]
            )
            try:
                total += float(response.strip())
            except ValueError:
                pass  # count unparseable responses as 0
        return {"conciseness": total / len(results)}
```

For high throughput, use `asyncio.gather` inside `compute` — the built-in QA judge metrics do exactly that.

## Wiring into `evaluate.py`

```python
"metrics": {
    "exact_match": {
        "type": "{app_name_snake}.metric:ExactAnswerMatch",
        "config": {"case_sensitive": False},
    },
    "conciseness": {
        "type": "{app_name_snake}.metric:ConcisenessScore",
        "config": {"judge_model": "gpt-4o-mini"},
    },
}
```

Metric names in the dict (`exact_match`, `conciseness`) are what appears in `results.metrics` — make them descriptive.

## Common Pitfalls

- **Returning a single float** — `compute` must return `dict[str, float]`. Even one score needs a key.
- **Judge answering with free text** — always ask the judge to respond with only a number or a fixed token, and parse defensively. Unparseable responses should count as 0 or be skipped with a warning, never crash the whole run.
- **Mixing metric types on incompatible pipelines** — a metric generic over `Result` won't run against a pipeline that emits `DocumentSearchResult`. Keep the pair aligned.

## Validation Checklist

- [ ] `metric.py` lives next to `evaluate.py`
- [ ] Metric extends `Metric[ResultT]` with the same `ResultT` the paired pipeline emits
- [ ] `compute` is `async def` and returns `dict[str, float]`
- [ ] Keys in the returned dict are descriptive — they become the metric names in `results.metrics`
- [ ] LLM-judge metrics defensively parse judge output (try/except around `float(...)`, or a regex match)
- [ ] `evaluate.py`'s `metrics` dict references the metric by `{app_name_snake}.metric:ClassName`
- [ ] When high throughput matters, judge calls go through `asyncio.gather` rather than sequential awaits
