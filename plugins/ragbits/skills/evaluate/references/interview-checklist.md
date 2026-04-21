# Interview Checklist

Domain-specific questions that help produce a well-shaped evaluation project.
Use these when the user provides minimal context. Combine into one message — do not ask one at a time.

## Core (always relevant)

- **Project name**: Directory/module name? (snake_case preferred)
- **What's being evaluated**: The retrieval step? The final answer? Both? A custom pipeline? (drives `--target`)
- **Test data**: Is there a labeled dataset already, or does one need to be generated?

## Target selection

- **Document search** (retrieval quality):
  - Do you care about precision/recall/F1 only, or also rank-aware metrics (MRR, NDCG)?
  - What counts as a correct retrieval — exact chunk match, ROUGE overlap with a ground-truth chunk, or an LLM-judge decision?

- **Question answer** (answer quality):
  - Which judge dimensions matter — correctness (vs. ground truth), faithfulness (grounded in context), relevance (matches question), consistency (style)?
  - Judge model — `gpt-4o-mini` is the default; switch to `gpt-4o` / Claude only if judge precision is the blocker
  - Dataset size — every QA sample costs one judge LLM call per metric, so a 1k-sample set × 3 metrics = 3k calls

- **Custom** (neither built-in fits):
  - What does the pipeline take as input and produce as output?
  - What's the scoring function — exact match, embedding cosine, BLEU/ROUGE, a custom LLM prompt, a hand-written rule?

## Dataset

- Where does the test set live? (HuggingFace Hub path, local CSV/JSON, internal database)
- How many samples? (affects judge model cost)
- Is there a held-out vs. dev split, or just one set?
- If no dataset exists yet → `dataset-generator` feature. Which documents should the synthetic set be derived from?

## Pipeline under test

- Point at an existing ragbits pipeline, or define a new one here?
- Which embedder, vector store, reranker, rephraser, etc. are being compared?
- Are there baseline + variant(s), or just a single config?

## Optimization (→ `optimize` feature)

- Which parameters should be swept? (k, score_threshold, embedding dimensions, chunk size, etc.)
- How many trials can the budget afford? (`n_trials` × per-trial evaluation cost = total)
- Objective — maximize F1? Maximize recall subject to a precision floor?

## Reporting

- Where should results go — console print, JSON file, dashboard, CI gate?
- Should this run on every PR, nightly, or on-demand?
- Pass/fail thresholds for regression checks?

## Deployment

- Local dev only, or part of a CI pipeline?
- API keys required (`OPENAI_API_KEY`, `HF_TOKEN` for private HuggingFace datasets)?
- Rate limits or cost caps to respect?

These feed into the `config` dict in `evaluate.py`. A specific dataset + baseline pipeline + well-chosen metrics beats a generic template, because evaluation is fundamentally comparative — the numbers only mean something relative to a known baseline and a fixed test set.
