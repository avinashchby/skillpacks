---
name: ai-eval-harness
description: >
  Creates structured evaluation harnesses for LLM outputs: generates test case suites,
  defines accuracy and quality metrics, measures latency and token cost, and produces
  A/B comparison reports between prompt versions or models. Supports automated scoring
  via LLM-as-judge and deterministic assertions. Trigger phrases: "eval harness",
  "evaluate llm", "benchmark prompt", "test llm output", "compare prompts", "ab test
  prompt", "llm evaluation", "measure prompt quality", "score llm responses",
  "regression test prompt", "prompt benchmark".
version: 1.0.0
---

# LLM Eval Harness

## When to Use

Activate this skill when the user:
- Wants to measure whether a prompt change improved or degraded output quality
- Needs to compare two models (e.g., GPT-4o vs Claude 3.5 Sonnet) on the same task
- Asks to generate test cases for an existing prompt or workflow
- Wants to track accuracy, latency, or cost metrics across prompt versions
- Is building a CI pipeline that gates on LLM output quality

Do NOT activate for unit testing of non-LLM code, or for one-off manual prompt testing.

---

## Instructions

### Step 1 — Define the evaluation scope

Before writing any code, establish:

1. **Task type** — Classification, extraction, summarization, generation, QA, code generation, reasoning
2. **What "correct" means** — Exact match, semantic similarity, rubric-based, or human judgment
3. **Metrics needed**:
   - Accuracy / F1 (for classification/extraction)
   - BLEU / ROUGE (for summarization — but flag their limitations)
   - LLM-as-judge score (for subjective quality)
   - Latency (p50, p95 in milliseconds)
   - Cost (input + output tokens × model price)
4. **A/B setup** — Are we comparing prompt versions, models, or both?
5. **Dataset** — Existing golden set, or does the user need help generating test cases?

State all assumptions before generating code.

### Step 2 — Generate test cases

If the user has no test cases, generate them systematically. A robust test suite covers:

**Category distribution for classification tasks:**
- Happy path: clear, prototypical examples of each class (60% of cases)
- Boundary cases: inputs that sit between two classes (20%)
- Edge cases: empty input, very short input, all-caps, non-English (10%)
- Adversarial: inputs designed to trigger misclassification (10%)

**For generation/summarization tasks:**
- 5-10 representative inputs from the actual production distribution
- 2-3 inputs with unusual length (very short, very long)
- 2-3 inputs with missing or ambiguous information

**Test case format:**
```python
from pydantic import BaseModel
from typing import Any

class TestCase(BaseModel):
    id: str
    input: dict[str, Any]          # matches the prompt's input variables
    expected_output: str | None    # None for LLM-judge-only evaluation
    expected_label: str | None     # for classification tasks
    tags: list[str] = []           # "happy_path", "edge_case", "adversarial"
    notes: str = ""                # human-readable description of what this tests
```

**Golden set file format (JSONL, one case per line):**
```jsonl
{"id": "tc_001", "input": {"text": "Cancel my subscription immediately"}, "expected_label": "cancellation", "tags": ["happy_path"]}
{"id": "tc_002", "input": {"text": ""}, "expected_label": "invalid", "tags": ["edge_case"]}
```

### Step 3 — Define the scoring functions

**Exact match (classification, structured extraction):**
```python
def score_exact_match(predicted: str, expected: str) -> float:
    return 1.0 if predicted.strip().lower() == expected.strip().lower() else 0.0
```

**JSON field match (structured output):**
```python
import json

def score_json_fields(predicted: str, expected: dict, required_fields: list[str]) -> float:
    """Score based on how many required fields are correctly extracted."""
    try:
        parsed = json.loads(predicted)
    except json.JSONDecodeError:
        return 0.0
    correct = sum(
        1 for field in required_fields
        if parsed.get(field) == expected.get(field)
    )
    return correct / len(required_fields)
```

**LLM-as-judge (generation quality, coherence, faithfulness):**
```python
JUDGE_PROMPT = """
You are evaluating an AI assistant's response to a task.

Task: {task_description}
Input: {input}
Response: {response}
{reference_section}

Score the response on a scale of 1-5 for each criterion:
- Accuracy (1=factually wrong, 5=fully correct)
- Completeness (1=missing key info, 5=covers everything needed)
- Format compliance (1=wrong format, 5=exact format requested)

Output strict JSON:
{{"accuracy": <1-5>, "completeness": <1-5>, "format_compliance": <1-5>, "reasoning": "<one sentence>"}}
"""

async def llm_judge_score(
    task_description: str,
    input_text: str,
    response: str,
    reference: str | None,
    judge_llm,
) -> dict:
    reference_section = f"Reference answer: {reference}" if reference else ""
    prompt = JUDGE_PROMPT.format(
        task_description=task_description,
        input=input_text,
        response=response,
        reference_section=reference_section,
    )
    raw = await judge_llm.ainvoke(prompt)
    return json.loads(raw.content)
```

**Use a different model as judge than the model under test.** If testing GPT-4o, use
Claude as judge. This avoids self-serving bias where a model rates its own outputs higher.

### Step 4 — Build the harness runner

```python
import asyncio
import time
from dataclasses import dataclass, field

@dataclass
class EvalResult:
    case_id: str
    input: dict
    predicted: str
    scores: dict[str, float]
    latency_ms: float
    input_tokens: int
    output_tokens: int
    cost_usd: float
    error: str | None = None

async def run_case(
    case: TestCase,
    chain,               # LangChain runnable or any async callable
    scorer,              # async fn(predicted, case) → dict[str, float]
    model_pricing: dict, # {"input_per_1k": 0.00015, "output_per_1k": 0.0006}
) -> EvalResult:
    start = time.perf_counter()
    try:
        result = await chain.ainvoke(case.input)
        latency_ms = (time.perf_counter() - start) * 1000
        predicted = result.content if hasattr(result, "content") else str(result)
        usage = getattr(result, "usage_metadata", None)
        input_tokens = usage.input_tokens if usage else 0
        output_tokens = usage.output_tokens if usage else 0
        cost = (
            input_tokens / 1000 * model_pricing["input_per_1k"]
            + output_tokens / 1000 * model_pricing["output_per_1k"]
        )
        scores = await scorer(predicted, case)
        return EvalResult(
            case_id=case.id,
            input=case.input,
            predicted=predicted,
            scores=scores,
            latency_ms=latency_ms,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            cost_usd=cost,
        )
    except Exception as e:
        return EvalResult(
            case_id=case.id,
            input=case.input,
            predicted="",
            scores={},
            latency_ms=0,
            input_tokens=0,
            output_tokens=0,
            cost_usd=0,
            error=str(e),
        )

async def run_eval(
    cases: list[TestCase],
    chain,
    scorer,
    model_pricing: dict,
    concurrency: int = 5,
) -> list[EvalResult]:
    """Run all test cases with bounded concurrency."""
    sem = asyncio.Semaphore(concurrency)
    async def bounded(case):
        async with sem:
            return await run_case(case, chain, scorer, model_pricing)
    return await asyncio.gather(*[bounded(c) for c in cases])
```

### Step 5 — Generate the comparison report

When running A/B evals (two prompt versions or two models):

```python
import statistics

def comparison_report(
    results_a: list[EvalResult],
    results_b: list[EvalResult],
    label_a: str = "Variant A",
    label_b: str = "Variant B",
) -> str:
    def agg(results: list[EvalResult]) -> dict:
        valid = [r for r in results if r.error is None]
        all_scores = {k: [r.scores[k] for r in valid if k in r.scores]
                      for k in (valid[0].scores if valid else {})}
        return {
            "n": len(results),
            "errors": len(results) - len(valid),
            "avg_scores": {k: statistics.mean(v) for k, v in all_scores.items()},
            "p50_latency_ms": statistics.median(r.latency_ms for r in valid) if valid else 0,
            "p95_latency_ms": sorted(r.latency_ms for r in valid)[int(len(valid)*0.95)] if valid else 0,
            "total_cost_usd": sum(r.cost_usd for r in results),
            "avg_cost_per_call": statistics.mean(r.cost_usd for r in valid) if valid else 0,
        }

    agg_a, agg_b = agg(results_a), agg(results_b)
    lines = [
        f"## Eval Report: {label_a} vs {label_b}\n",
        f"| Metric               | {label_a:<20} | {label_b:<20} | Delta      |",
        f"|----------------------|{'-'*22}|{'-'*22}|------------|",
    ]
    for metric, val_a in agg_a["avg_scores"].items():
        val_b = agg_b["avg_scores"].get(metric, 0)
        delta = val_b - val_a
        delta_str = f"+{delta:.3f}" if delta >= 0 else f"{delta:.3f}"
        lines.append(f"| {metric:<20} | {val_a:<22.3f} | {val_b:<22.3f} | {delta_str:<10} |")
    lines += [
        f"| p50 latency (ms)     | {agg_a['p50_latency_ms']:<22.0f} | {agg_b['p50_latency_ms']:<22.0f} | {agg_b['p50_latency_ms']-agg_a['p50_latency_ms']:+.0f}       |",
        f"| p95 latency (ms)     | {agg_a['p95_latency_ms']:<22.0f} | {agg_b['p95_latency_ms']:<22.0f} | {agg_b['p95_latency_ms']-agg_a['p95_latency_ms']:+.0f}       |",
        f"| avg cost/call (USD)  | {agg_a['avg_cost_per_call']:<22.6f} | {agg_b['avg_cost_per_call']:<22.6f} | {agg_b['avg_cost_per_call']-agg_a['avg_cost_per_call']:+.6f} |",
        f"| total cost (USD)     | {agg_a['total_cost_usd']:<22.4f} | {agg_b['total_cost_usd']:<22.4f} |            |",
        f"| errors               | {agg_a['errors']:<22} | {agg_b['errors']:<22} |            |",
    ]
    return "\n".join(lines)
```

### Step 6 — Project structure

```
eval-harness/
├── pyproject.toml
├── src/
│   └── eval/
│       ├── __init__.py
│       ├── runner.py          # run_eval, run_case
│       ├── scorers.py         # exact_match, json_fields, llm_judge
│       ├── reporter.py        # comparison_report, export to CSV/HTML
│       └── types.py           # TestCase, EvalResult, ModelPricing
├── datasets/
│   └── <task-name>.jsonl      # golden test sets (version-controlled)
├── results/
│   └── <date>_<variant>.json  # raw results (gitignored or archived)
└── scripts/
    └── run_eval.py            # CLI entry point
```

---

## Examples

### Example 1 — Evaluating a Classification Prompt

**Task:** Support ticket classification (billing / technical / account / general)

**Test dataset (datasets/ticket_classification.jsonl):**
```jsonl
{"id": "tc_001", "input": {"ticket": "My invoice is wrong"}, "expected_label": "billing", "tags": ["happy_path"]}
{"id": "tc_002", "input": {"ticket": "App crashes on iOS 17"}, "expected_label": "technical", "tags": ["happy_path"]}
{"id": "tc_003", "input": {"ticket": "I can't log in and my payment failed"}, "expected_label": "billing", "tags": ["boundary"]}
{"id": "tc_004", "input": {"ticket": ""}, "expected_label": "invalid", "tags": ["edge_case"]}
```

**Run comparison between two prompt versions:**
```python
results_v1 = await run_eval(cases, chain_v1, scorer, PRICING_GPT4O_MINI)
results_v2 = await run_eval(cases, chain_v2, scorer, PRICING_GPT4O_MINI)
print(comparison_report(results_v1, results_v2, "Prompt v1", "Prompt v2"))
```

**Sample output:**
```
## Eval Report: Prompt v1 vs Prompt v2

| Metric               | Prompt v1             | Prompt v2             | Delta      |
|----------------------|-----------------------|-----------------------|------------|
| accuracy             | 0.720                 | 0.880                 | +0.160     |
| p50 latency (ms)     | 812                   | 634                   | -178       |
| avg cost/call (USD)  | 0.000042              | 0.000031              | -0.000011  |
```

Verdict: v2 is more accurate (+16%), cheaper (-26%), and faster (-22%). Ship v2.

---

### Example 2 — Evaluating RAG Answer Faithfulness

**Metrics needed:** Faithfulness (did the answer come from context?) + Completeness

```python
RAG_JUDGE_PROMPT = """
Context provided to the AI:
{context}

Question: {question}
AI Answer: {answer}

Evaluate:
1. Faithfulness: Is every claim in the answer supported by the context? (1-5)
2. Completeness: Does the answer address all aspects of the question the context can support? (1-5)
3. Hallucination: Did the AI add any information NOT in the context? (yes/no)

Output JSON: {{"faithfulness": <1-5>, "completeness": <1-5>, "hallucination": <bool>, "reasoning": "<one sentence>"}}
"""
```

**Minimum acceptable thresholds for production:**
- Faithfulness mean ≥ 4.0
- Hallucination rate ≤ 5%
- Completeness mean ≥ 3.5

---

### Example 3 — CI Integration (GitHub Actions)

**scripts/run_eval.py** with a pass/fail threshold:
```python
import asyncio, sys, json
from pathlib import Path
from eval.runner import run_eval
from eval.reporter import comparison_report

PASS_THRESHOLD = {"accuracy": 0.85}

async def main():
    cases = load_jsonl(Path("datasets/ticket_classification.jsonl"))
    results = await run_eval(cases, build_chain(), build_scorer(), PRICING)
    valid = [r for r in results if not r.error]
    avg_accuracy = sum(r.scores.get("accuracy", 0) for r in valid) / len(valid)
    print(f"accuracy: {avg_accuracy:.3f} (threshold: {PASS_THRESHOLD['accuracy']})")
    if avg_accuracy < PASS_THRESHOLD["accuracy"]:
        print("FAIL: accuracy below threshold")
        sys.exit(1)
    print("PASS")

asyncio.run(main())
```

**.github/workflows/eval.yml:**
```yaml
name: Prompt Eval
on: [pull_request]
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: uv sync
      - run: uv run python scripts/run_eval.py
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

---

## Anti-patterns

1. **Using BLEU or ROUGE as primary quality metrics for LLM output.** BLEU/ROUGE measure
   n-gram overlap against a reference. An LLM answer can be semantically correct while
   sharing zero n-grams with the reference. Use LLM-as-judge or semantic similarity
   (cosine similarity of embeddings) for free-form generation tasks. Reserve BLEU/ROUGE
   for translation where n-gram fidelity actually matters.

2. **Using the same model as both the system under test and the judge.** GPT-4o judging
   GPT-4o outputs will systematically rate them higher than a neutral judge would. Always
   use a different model family as the judge (e.g., Claude judging GPT, or GPT judging
   Claude). This is the single largest source of inflated eval scores.

3. **Evaluating on fewer than 30 test cases and drawing conclusions.** A 10-case eval
   where the model gets 8 correct has a confidence interval so wide (+/- 25%) it is
   statistically meaningless. Aim for at least 50 cases for initial evals; 200+ for
   production gates. If your golden set is too small, use the LLM to generate
   additional synthetic cases from the real distribution.

4. **Not versioning the test dataset alongside the prompt.** If you update the golden set
   at the same time as the prompt, you can no longer tell whether an accuracy improvement
   came from the prompt change or the easier test cases. Treat `datasets/*.jsonl` as
   append-only; create a new dataset file when the task definition fundamentally changes.

5. **Optimizing for average score while ignoring error rate.** A system that scores 0.90
   average accuracy but has 5% parse errors (where the output is not valid JSON and
   cannot be used) is not production-ready. Always track and gate on `error_rate` as a
   separate metric with a hard maximum (e.g., error_rate ≤ 1% before shipping).
