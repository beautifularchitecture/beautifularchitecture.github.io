+++
date = '2026-07-05T10:00:15+10:00'
draft = false
title = 'Braintrust: Production Eval History, Dashboards and CI Gates'
tags = ['Braintrust', 'Evals', 'LLM', 'Testing', 'CI', 'Dashboards']
summary = "Braintrust bridges the gap between local eval scripts and production monitoring — dataset management, commit-linked eval history, regression dashboards, and human annotation queues."
+++

## What Braintrust Provides

The [eval pyramid](../../ai-outputs-eval/) has a gap between "I run evals locally" and "I catch regressions before they ship." Local scripts give you a terminal table and a JSON baseline file. They do not give you historical dashboards, commit-linked comparisons, or team-wide visibility.

Braintrust fills that gap. It is a platform (with an open-source scoring library) that records every eval run, links it to the git commit that triggered it, and surfaces regressions in a web UI before they merge. It also provides human annotation queues so reviewers can score outputs that the automated judge is unsure about.

## When to Add Braintrust

| Stage          | Setup                                 | When to upgrade                                      |
| -------------- | ------------------------------------- | ---------------------------------------------------- |
| Solo developer | `eval-baseline.json` + `--gate` in CI | You have >1 person looking at eval results           |
| Small team     | Braintrust free tier                  | You want dashboards without maintaining infra        |
| Large team     | Braintrust paid tier                  | You need audit trails, SLAs, and human review queues |

The companion repo starts with the minimal pattern — `eval-baseline.json` checked into git, `--gate` in CI. Braintrust is what you add when you outgrow that: when you need to see whether this week's scores are higher than last month's, or when a team member asks "did the model swap regress anything I should know about?"

## Core Concepts

### Datasets

A dataset in Braintrust is a versioned collection of eval examples. Each example has an input, expected output (optional), and metadata. Datasets are the managed version of a golden dataset:

```python
from braintrust import init_dataset

dataset = init_dataset(
    project="my-app",
    name="golden-evals",
    description="Golden dataset for regression testing",
)

dataset.insert([
    {
        "input": {"prompt": "What is the capital of France?"},
        "expected": "Paris",
        "metadata": {"category": "geography", "difficulty": "easy"},
    },
    {
        "input": {"prompt": "Write a Python Fibonacci function."},
        "expected": None,  # open-ended: no exact expected output
        "metadata": {"category": "code", "difficulty": "medium"},
    },
])
```

Datasets are versioned — you can roll back if a dataset change introduces bad cases.

### Custom Scorers

Scorers are Python functions that take an output, expected value, and optional metadata, and return a numeric score:

```python
from braintrust import Eval

def contains_scorer(output, expected, metadata=None):
    """1.0 if output contains the expected substring."""
    if expected is None:
        return 1.0  # no expected output for open-ended tasks
    return 1.0 if expected.lower() in output.lower() else 0.0

def json_validity_scorer(output, expected, metadata=None):
    """1.0 if output is valid JSON."""
    import json
    try:
        json.loads(output)
        return 1.0
    except json.JSONDecodeError:
        return 0.0

def word_limit_scorer(output, expected, metadata=None):
    """1.0 if output is within word limit from metadata."""
    limit = (metadata or {}).get("max_words", 200)
    return 1.0 if len(output.split()) <= limit else 0.0
```

### Running an Eval

The `Eval` function ties datasets, task, and scorers together:

```python
def task(input_data):
    """The function under evaluation. Braintrust calls this for each dataset row."""
    from src.llm import openrouter_chat_model
    model = openrouter_chat_model(temperature=0.0)
    response = model.invoke(input_data["prompt"])
    return response.content.strip()


results = Eval(
    "my-app",
    data=dataset,          # the dataset defined above
    task=task,             # the function to evaluate
    scores=[contains_scorer, json_validity_scorer, word_limit_scorer],
    metadata={"model": "claude-sonnet-4", "prompt_version": "v3"},
)
```

Braintrust runs every dataset row through the task function, applies every scorer, and uploads the results to the platform. The output includes:

- Mean and variance per scorer
- Per-row scores (drill down into which cases regressed)
- A link to the dashboard showing this run in context

### Commit-Linked History

Every eval run is automatically associated with the git commit SHA, branch name, and any tags you provide. The dashboard shows:

- **Timeline**: scores over time, annotated with git commits
- **Comparisons**: side-by-side diff between any two runs (e.g. before and after a model swap)
- **Regression alerts**: highlighted in red when a score drops below the previous run or a configured threshold

This is the feature that makes Braintrust more useful than a local baseline file. With `eval-baseline.json`, you compare against one fixed point. With Braintrust, you see trends: "our helpfulness score has been declining over the last 3 deployments and we did not notice because each individual run stayed above baseline."

## CI Gate

Braintrust gates work the same way as the `--gate` pattern, but the baseline is the platform history rather than a local file:

```python
# ci_gate.py
from braintrust import Eval

results = Eval("my-app", data=dataset, task=task, scores=[scorer])
if not results.passed():
    print("❌ Regression detected — blocking merge")
    exit(1)
```

In CI:

```yaml
- run: uv run python ci_gate.py
  env:
    BRAINTRUST_API_KEY: ${{ secrets.BRAINTRUST_API_KEY }}
```

Braintrust compares the current run against the previous run on the same dataset (or a configured baseline). If any score dropped beyond the threshold, the gate fails.

## Human Annotation Queues

Braintrust has a built-in human review workflow. Rows where the automated judge score is uncertain (near the threshold) or where multiple scorers disagree are surfaced in an annotation queue. Reviewers score them manually, and those human scores become the ground truth for calibrating the automated scorers.

This directly implements Layer 6 of the eval pyramid (human-in-the-loop) without needing a separate tool like Label Studio.

## Customizing Scorers

Braintrust scorers are sandboxed Python — you can call any model, run any computation, or fetch external data:

```python
def faithfulness_scorer(output, expected, metadata=None):
    """Use an LLM judge to score faithfulness."""
    from src.llm import openrouter_chat_model
    model = openrouter_chat_model(temperature=0.0)
    judge_prompt = f"""Score faithfulness 0.0–1.0.

Output: {output}
Context: {expected}

Score:"""
    response = model.invoke(judge_prompt)
    import re
    match = re.search(r"(\d\.?\d*)", response.content)
    return float(match.group(1)) if match else 0.0
```

Because scorers are Python, you can reuse your existing `src/eval.py` metrics directly.

## Braintrust vs. Local Baseline

| Aspect          | `eval-baseline.json` + `--gate`    | Braintrust                                                      |
| --------------- | ---------------------------------- | --------------------------------------------------------------- |
| Setup           | A JSON file and 10 lines of Python | Platform account + API key                                      |
| History         | One point (the recorded baseline)  | Full timeline across all runs                                   |
| Comparisons     | Before vs. baseline only           | Any two runs, any date range                                    |
| Team visibility | PR comments only                   | Web dashboards + Slack notifications                            |
| Human review    | Manual process                     | Built-in annotation queues                                      |
| Cost            | Free                               | Free tier available; paid for history retention + team features |
| When to use     | Solo dev or early stage            | Team with >1 reviewer, or when you need trend analysis          |

## Companion Repo

The companion repo includes [`scripts/example_braintrust.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/tools/example_braintrust.py) with a working Braintrust eval integration.

## Further Reading

- [Braintrust documentation](https://www.braintrust.dev/docs)
- [Testing LLM Outputs: The Eval Pyramid](../../ai-outputs-eval/)
- [DeepEval: CI-Gated Typed Metrics for LLM Outputs](../deepeval/)
- [Promptfoo: Model Comparison and Red-Teaming](../promptfoo/)
- [pytest for LLM Evaluation](../pytest/)
