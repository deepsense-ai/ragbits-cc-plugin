# Custom Pipeline Checklist

Quality gates for evaluating a pipeline that isn't document-search or question-answer — e.g., a custom agent, a tool-using chain, a classifier, a multi-step orchestrator. Subclass `EvaluationPipeline` to tell `Evaluator` how to run your code against each dataset sample.

## When to reach for this

Built-in pipelines cover the two most common shapes. Use a custom pipeline when:

- The thing under test takes inputs or produces outputs the built-ins don't understand
- You need to invoke multiple LLM calls, tool calls, or post-processing steps in sequence per sample
- The pipeline is a library you already own and you want to wrap it rather than reimplement inside a config

If it's a document-search or QA variant with a slightly different component (say, a custom reranker), stay with the built-in pipeline and just swap the component inside its config — that's what the factory pattern exists for. Custom pipelines are for structurally different workloads.

## Imports

```python
from ragbits.evaluate.pipelines.base import EvaluationPipeline
```

Add whatever your pipeline depends on — `ragbits.agents`, `ragbits.core.llms`, etc.

## Minimal Custom Pipeline

Write it in `pipeline.py` alongside `evaluate.py`:

```python
"""Custom pipeline that {one-line description of what it does}."""

from dataclasses import dataclass

from ragbits.core.llms import LiteLLM
from ragbits.evaluate.pipelines.base import EvaluationPipeline


@dataclass
class Sample:
    """A single input drawn from the dataloader."""
    question: str
    reference: str


@dataclass
class Result:
    """What the pipeline produces for each sample."""
    question: str
    answer: str
    reference: str


class MyPipeline(EvaluationPipeline[Sample, Result]):
    """Runs the thing under test against each dataloader sample."""

    def __init__(self, model_name: str = "gpt-4o-mini") -> None:
        super().__init__()
        self.llm = LiteLLM(model_name=model_name)

    async def __call__(self, sample: Sample) -> Result:
        response = await self.llm.generate(
            [{"role": "user", "content": sample.question}]
        )
        return Result(
            question=sample.question,
            answer=response,
            reference=sample.reference,
        )
```

### What's important

- `EvaluationPipeline` is generic over the input (`Sample`) and output (`Result`) types. Those dataclasses become the contract metrics see.
- `__call__` is async and runs **once per sample**. The evaluator handles concurrency and batching on top.
- Keep this class focused — pipeline setup goes in `__init__`, per-sample work in `__call__`. If `__call__` grows, factor helper methods; don't do lazy setup inside it.

## Wiring into `evaluate.py`

```python
config = {
    "evaluation": {
        "dataloader": {
            "type": "ragbits.evaluate.dataloaders.question_answer:QuestionAnswerDataLoader",
            "config": {
                "source": {
                    "type": "ragbits.core.sources:HuggingFaceSource",
                    "config": {"path": "...", "split": "train"},
                },
            },
        },
        "pipeline": {
            "type": "{app_name_snake}.pipeline:MyPipeline",
            "config": {"model_name": "{model}"},
        },
        "metrics": {
            # Use a built-in metric or pair with a custom one — see new-custom-metric.md
        },
    }
}
```

The `type` string is the same `module:Class` form as built-ins. Python's import machinery finds it because the project is installed via `pip install -e .`.

## Matching the Sample Shape to the Dataloader

The dataloader decides what each `Sample` looks like. When you reuse `QuestionAnswerDataLoader`, the fields are already fixed (question / answer / context); your dataclass just mirrors them. When you pair a custom pipeline with a custom dataloader, both sides evolve together — keep the dataclasses in one module so changes stay coordinated.

## Common Pitfalls

- **Blocking I/O inside `__call__`** — wrap synchronous HTTP/DB calls in `run_in_executor` or an async client, otherwise the evaluator's concurrency doesn't help.
- **Global state** — the evaluator may instantiate your pipeline more than once. Don't rely on module-level singletons for anything the pipeline mutates.
- **Exception handling** — let exceptions bubble up; the evaluator records failures per-sample and summarizes them. Swallowing errors hides real regressions.

## Validation Checklist

- [ ] `pipeline.py` lives next to `evaluate.py` and the project installs via `pip install -e .`
- [ ] `MyPipeline` extends `EvaluationPipeline[InputType, OutputType]` with concrete dataclass types
- [ ] `__call__` is `async def`, runs per sample, and returns the declared output type
- [ ] No blocking I/O inside `__call__` (use async clients or `run_in_executor`)
- [ ] The config's `pipeline.type` is `{app_name_snake}.pipeline:MyPipeline` — matching the import path
- [ ] At least one metric understands the pipeline's output type — either by reusing a built-in that speaks QA/DocSearch shapes, or by pairing with a custom metric (see `new-custom-metric.md`)
