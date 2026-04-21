# Dataset Generator Checklist

Quality gates for generating a synthetic evaluation dataset when no labeled test set exists. `DatasetGenerationPipeline` reads raw documents and produces question/answer (or retrieval-style) samples you can evaluate against.

## When to reach for this

- The user has a document corpus but no labeled queries → generate synthetic questions + expected chunks
- Bootstrapping a new RAG project — synthetic data provides a useful day-one baseline, even if a human-curated set supersedes it later
- Stress-testing a pipeline by generating adversarial or edge-case queries

Synthetic datasets are a starting point, not a substitute. Human-reviewed samples catch the kinds of mistakes an LLM generator also makes — validate a 30–50 sample slice before trusting the full synthetic set.

## Dependency

Already included via `ragbits-evaluate` — no extra install needed.

## Imports

```python
from ragbits.evaluate.dataset_generator import DatasetGenerationPipeline
```

## Minimal `generate.py`

```python
"""
Synthesize an evaluation dataset from raw documents.
"""

import asyncio

from ragbits.evaluate.dataset_generator import DatasetGenerationPipeline


config = {
    "source": {
        "type": "ragbits.core.sources:HuggingFaceSource",
        "config": {
            "path": "{source_dataset_path}",
            "split": "train[:50]",
        },
    },
    "llm": {
        "type": "ragbits.core.llms:LiteLLM",
        "config": {"model_name": "{generator_model}"},
    },
    "n_samples_per_document": 3,
    "output_path": "synthetic_dataset.jsonl",
}


async def generate() -> None:
    pipeline = DatasetGenerationPipeline.from_config(config)
    result = await pipeline.run()
    print(f"Generated {result.n_samples} samples → {config['output_path']}")


if __name__ == "__main__":
    asyncio.run(generate())
```

The exact config keys (`n_samples_per_document`, `output_path`, judge model selection) may vary slightly across ragbits versions — inspect `DatasetGenerationPipeline` in the installed package when a field doesn't exist, or fall back to the pipeline's `__init__` kwargs.

## What to Generate

- **Questions only** — for use with `QuestionAnswerDataLoader` when you have ground-truth answers elsewhere
- **Question + reference answer pairs** — complete QA datasets
- **Question + ground-truth chunks** — for use with `DocumentSearchDataLoader`, measuring retrieval quality

The pipeline typically picks a mode based on config. When the output mode is ambiguous, default to question + answer + source-chunk, which works with both dataloaders.

## Cost Considerations

Synthesis is an LLM call per generated sample. For `n_samples_per_document=3` over 100 documents, that's 300 calls — budget accordingly. `gpt-4o-mini` usually produces reasonable questions; use a stronger model only when the corpus is technical enough that weaker generators miss nuance.

## Reviewing the Output

After generation, sample and inspect:

```python
import json

with open("synthetic_dataset.jsonl") as f:
    for i, line in enumerate(f):
        if i >= 10:
            break
        sample = json.loads(line)
        print(sample)
```

Look for:
- Questions the corpus genuinely answers (not hallucinated questions)
- A spread across different documents (not all from the first three)
- Ground-truth answers faithful to the source chunks

When ~15–20% of sampled pairs look wrong, throw the dataset out and tune the generator config (more documents, stronger generator model, smaller batches). It's cheaper than shipping bad numbers.

## Using the Generated Dataset in `evaluate.py`

Point your dataloader at the generated file:

```python
"dataloader": {
    "type": "ragbits.evaluate.dataloaders.question_answer:QuestionAnswerDataLoader",
    "config": {
        "source": {
            "type": "ragbits.core.sources:LocalFileSource",
            "config": {"path": "synthetic_dataset.jsonl"},
        },
    },
},
```

`LocalFileSource` handles JSONL and CSV out of the box. For HuggingFace upload workflows, push the generated file as a dataset and reference it via `HuggingFaceSource` — that makes the dataset versioned and shareable with the team.

## Validation Checklist

- [ ] `generate.py` lives next to `evaluate.py`
- [ ] Source dataset path exists and is accessible
- [ ] Generator model is specified and the env var (`OPENAI_API_KEY` etc.) is documented in the README
- [ ] `n_samples_per_document` (or the equivalent size knob) is set to something sane — budget before running
- [ ] Output path is `.jsonl` or `.csv`, matching what `LocalFileSource` can read
- [ ] README tells the user to sample-inspect the output before trusting it
- [ ] The generated file is gitignored when large; small samples can be checked in for reproducibility
