+++
date = '2026-07-05T10:00:05+10:00'
draft = false
title = 'pytest for LLM Evaluation: Parametrized Tests, Fixtures, and CI Gates'
tags = ['pytest', 'Evals', 'LLM', 'Testing', 'Python', 'CI']
summary = "pytest is the runner that ties the eval pyramid together — parametrized golden datasets, fixtures for model connections, invariant checks, and sampling loops. No extra framework needed."
+++

## Why pytest for LLM Evals

Most teams adopt a purpose-built eval framework (DeepEval, Promptfoo, Braintrust) and let their test runner fall out of practice. That is backwards. The framework handles scoring and dashboards. The test runner handles the discipline of "run before merge, fail on regression" — and pytest is already the standard Python runner for that job.

pytest gives you three patterns that map directly to LLM eval needs:

| Traditional pytest concept      | LLM eval equivalent                                 |
| ------------------------------- | --------------------------------------------------- |
| `@pytest.mark.parametrize`      | A golden dataset of input-output cases              |
| Fixtures (`@pytest.fixture`)    | Model connections, API clients, seed outputs        |
| `yield`-based fixtures          | Post-test invariant checks (no secrets, valid JSON) |
| `pytest-repeat` or manual loops | Statistical sampling across N runs                  |

You do not need a separate eval runner. If you already use pytest for unit tests, adding LLM evals is adding new test files, not a new toolchain. This fits Layer 1 (deterministic), Layer 3 (invariant), and Layer 5 (sampling) of the [eval pyramid](../../ai-outputs-eval/).

## Parametrized Golden Datasets

A golden dataset is a list of (prompt, expected-checks) pairs. pytest's `@parametrize` runs each pair through the same test function:

```python
import pytest
import json

GOLDEN_CASES = [
    pytest.param(
        "What is the capital of France? Answer in one word.",
        {"must_contain": ["paris"], "max_words": 5},
        id="capital-france",
    ),
    pytest.param(
        "Write a list comprehension for squaring even numbers 0 to 20.",
        {"must_contain": ["**2", "range", "if"], "must_not_contain": ["import"]},
        id="list-comprehension",
    ),
    pytest.param(
        'Return valid JSON: {"name": "Alice", "age": 30} with age incremented.',
        {"expects_valid_json": True},
        id="json-output",
    ),
]


def score_contains(output, words):
    return all(w in output.lower() for w in words)

def score_excludes(output, words):
    return not any(w in output.lower() for w in words)

def score_max_words(output, limit):
    return len(output.split()) <= limit

def score_valid_json(output):
    try:
        json.loads(output)
        return True
    except json.JSONDecodeError:
        return False


@pytest.mark.parametrize("prompt,checks", GOLDEN_CASES)
def test_golden_case(prompt, checks, model):
    response = model.invoke(prompt)
    output = response.content.strip()
    failures = []

    for word in checks.get("must_contain", []):
        if not score_contains(output, [word]):
            failures.append(f"missing '{word}'")
    for word in checks.get("must_not_contain", []):
        if not score_excludes(output, [word]):
            failures.append(f"contains banned '{word}'")
    for limit in [checks.get("max_words", 0)]:
        if limit and not score_max_words(output, limit):
            failures.append(f"exceeds {limit} words")
    if checks.get("expects_valid_json") and not score_valid_json(output):
        failures.append("invalid JSON")

    assert not failures, f"{failures}\n---\n{output[:200]}"
```

Each case runs as a separate test with a descriptive ID. When a case fails, you know exactly which prompt regressed.

## Fixtures for Model Connections

The `model` fixture in the test above is injected by pytest:

```python
@pytest.fixture(scope="session")
def model():
    """Model connection, reused across all tests in the session."""
    from src.llm import openrouter_chat_model
    return openrouter_chat_model(temperature=0.0)
```

`scope="session"` is important — you do not want to reconnect the model for every test case. One connection per test run.

For testing different models or configurations, use parametrized fixtures:

```python
@pytest.fixture(params=["openrouter", "groq", "bedrock"])
def model(request):
    if request.param == "openrouter":
        from src.llm import openrouter_chat_model
        return openrouter_chat_model(temperature=0.0)
    elif request.param == "groq":
        from src.llm import groq_chat_model
        return groq_chat_model(temperature=0.0)
```

Now every test runs against all three providers automatically. You see which provider fails which case.

## Post-Test Invariant Fixtures

Use `yield` fixtures to run invariant checks after every test — without adding an assert to every test function:

```python
@pytest.fixture
def invariant_checks():
    """Capture outputs and check invariants after each test."""
    outputs = []
    yield outputs
    for output in outputs:
        assert "sk-" not in output, "Secret key leaked"
        assert len(output) > 0, "Empty output"
        assert len(output.split()) <= 500, "Output too long"


def test_capital_france(model, invariant_checks):
    response = model.invoke("What is the capital of France?")
    output = response.content.strip()
    invariant_checks.append(output)
    assert "Paris" in output
```

The invariant fixture runs _after_ the test body. If the test passes but an invariant fails, the test fails — and you know the invariant, not the specific assertion, caught the regression.

## Statistical Sampling

A single run at temperature > 0 tells you almost nothing. Run each case N times and assert on the distribution:

```python
import statistics

N = 5

@pytest.mark.parametrize("prompt,checks", GOLDEN_CASES)
def test_with_sampling(prompt, checks, model):
    outputs = [model.invoke(prompt).content.strip() for _ in range(N)]

    pass_count = 0
    for output in outputs:
        # Simplified: all checks must pass
        all_pass = all(
            score_contains(output, checks.get("must_contain", [])) and
            score_excludes(output, checks.get("must_not_contain", []))
        )
        if all_pass:
            pass_count += 1

    pass_rate = pass_count / N
    assert pass_rate >= 0.6, f"Pass rate too low: {pass_rate:.0%} ({pass_count}/{N})"
```

Report the distribution in the test output:

```python
    scores = [score_case(output, checks) for output in outputs]
    print(f"\n  mean={statistics.mean(scores):.3f}  stdev={statistics.stdev(scores):.3f}  pass-rate={pass_rate:.0%}")
```

## CI Integration

The eval suite runs like any other pytest run:

```yaml
# .github/workflows/eval.yml
name: LLM Eval Suite
on: [pull_request]
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install uv && uv sync
      - run: uv run pytest tests/evals/ -v --tb=short
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

Add `--gate` logic by comparing against a baseline file:

```python
def test_baseline_gate(request):
    baseline = json.loads(Path("eval-baseline.json").read_text())
    current = run_suite()
    for case in current:
        prev = baseline.get(case["id"], 0.0)
        assert case["mean"] >= prev, (
            f"{case['id']}: {case['mean']:.3f} < baseline {prev:.3f}"
        )
```

## Combining With Other Tools

| Tool           | How pytest integrates                                                                        |
| -------------- | -------------------------------------------------------------------------------------------- |
| **DeepEval**   | `deepeval` metrics are pytest-compatible; `deepeval.assert_test()` works inside pytest tests |
| **hypothesis** | `@given` decorator works inside pytest tests alongside `@parametrize`                        |
| **Braintrust** | Wrap `braintrust.Eval` in a pytest fixture; results post to the platform                     |
| **Promptfoo**  | pytest triggers `promptfoo eval` via subprocess; results parsed and asserted in Python       |

The unifying pattern: pytest is the runner, each tool is a scorer you call inside a test.

## Companion Repo

The companion repo includes [`scripts/example_pytest.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/tools/example_pytest.py) with a complete parametrized golden dataset, invariant fixtures, and sampling.

## Further Reading

- [pytest documentation](https://docs.pytest.org/)
- [Testing LLM Outputs: The Eval Pyramid](../../ai-outputs-eval/)
- [hypothesis: Property-Based Testing for LLM Outputs](../hypothesis/)
- [DeepEval: CI-Gated Typed Metrics for LLM Outputs](../deepeval/)
