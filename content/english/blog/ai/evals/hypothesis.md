+++
date = '2026-07-05T10:02:00+10:00'
draft = false
title = 'hypothesis: Property-Based Testing for LLM Outputs'
tags = ['hypothesis', 'Property-Based Testing', 'Evals', 'LLM', 'Testing', 'Python']
summary = "Property-based testing with hypothesis is a natural fit for non-deterministic LLM output: instead of asserting a specific answer, you assert invariants that must hold across all valid outputs regardless of phrasing."
+++

## Why Property-Based Testing Exists

Traditional unit testing says: "given input X, the function must return output Y." That is _example-based_ testing. It works when the function is deterministic — same input always produces same output. LLMs break that assumption. Ask the same model the same question twice and you get two differently-worded but equally correct answers. Example-based tests either over-match (failing on valid paraphrases) or under-match (asserting only a substring and missing the semantic regression).

Property-based testing inverts the approach. You do not assert a specific output. You assert **properties** that must hold for _any_ valid output:

- The output never contains secrets
- The output is valid JSON when JSON was requested
- The output stays within a word limit
- The output never contains banned phrases
- The output includes all required sections

Then you feed the system a wide range of inputs — including edge cases you would not think to write by hand — and verify that every output satisfies those properties. The framework ([hypothesis](https://hypothesis.readthedocs.io/)) generates the inputs automatically. It finds the edge cases that violate your invariants, shrinks them to minimal failing examples, and reports them back.

This is Layer 3 of the [eval pyramid](../../ai-outputs-eval/) — property-based / invariant testing.

## Core Concepts

hypothesis has three pieces you need to understand:

### `@given` — the decorator that generates inputs

Instead of writing a test with a fixed input, you declare the _shape_ of valid inputs and let hypothesis generate concrete values:

```python
from hypothesis import given, strategies as st

@given(st.text(min_size=1, max_size=200))
def test_output_never_contains_secrets(topic: str):
    output = model.invoke(topic)
    assert "sk-" not in output
```

hypothesis will run this test with dozens of automatically generated strings: empty strings, Unicode, whitespace-only, very long strings, strings with special characters — whatever might trigger a failure.

### `strategies` — the building blocks for describing valid inputs

Strategies are composable. You combine them to describe exactly what kind of inputs your system should handle:

```python
# A user query: non-empty text, up to 500 chars
st.text(min_size=1, max_size=500)

# A choice between prompt styles
st.sampled_from(["concise", "detailed", "json"])

# A positive integer (e.g. max_tokens setting)
st.integers(min_value=1, max_value=4096)

# Composed: a topic and a style
st.tuples(
    st.text(min_size=1, max_size=200),
    st.sampled_from(["concise", "detailed"]),
)
```

### Invariants — the properties that must always hold

This is where you encode your eval criteria. Each invariant is a function that takes an output and returns a boolean:

```python
def invariant_no_secrets(output: str) -> bool:
    import re
    patterns = [
        r"sk-[A-Za-z0-9]{20,}",       # OpenAI keys
        r"AKIA[0-9A-Z]{16}",           # AWS access keys
        r"-----BEGIN (RSA |EC )?PRIVATE KEY-----",
    ]
    return not any(re.search(p, output) for p in patterns)

def invariant_valid_json(output: str) -> bool:
    import json
    try:
        json.loads(output)
        return True
    except json.JSONDecodeError:
        return False

def invariant_within_word_limit(output: str, limit: int = 200) -> bool:
    return len(output.split()) <= limit
```

## Practical LLM Invariants

What invariants should you check? Here is a toolkit organised by what they catch.

### Safety and compliance

```python
def invariant_no_pii(output: str) -> bool:
    """No email addresses, phone numbers, or credit cards."""
    import re
    patterns = [
        r"\b[\w\.-]+@[\w\.-]+\.\w+\b",           # email
        r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",        # US phone
        r"\b\d{4}[ -]?\d{4}[ -]?\d{4}[ -]?\d{4}\b",  # credit card
    ]
    return not any(re.search(p, output) for p in patterns)

def invariant_no_banned_phrases(output: str) -> bool:
    banned = ["I cannot help with that", "As an AI", "I don't have access"]
    return not any(phrase.lower() in output.lower() for phrase in banned)
```

### Structural integrity

```python
def invariant_valid_json(output: str) -> bool:
    import json
    try:
        json.loads(output)
        return True
    except (json.JSONDecodeError, ValueError):
        return False

def invariant_required_sections(output: str, sections: list[str]) -> bool:
    return all(s.lower() in output.lower() for s in sections)

def invariant_max_depth(output: str, max_depth: int = 6) -> bool:
    """Check nesting depth of JSON or brackets."""
    depth = 0
    max_d = 0
    for ch in output:
        if ch in ("{", "["):
            depth += 1
            max_d = max(max_d, depth)
        elif ch in ("}", "]"):
            depth -= 1
    return max_d <= max_depth
```

### Performance budgets

```python
def invariant_token_budget(output: str, max_tokens: int = 500) -> bool:
    """Approximate: 1 token ≈ 4 chars for English."""
    return len(output) // 4 <= max_tokens

def invariant_no_repeated_text(output: str, min_repeat: int = 20) -> bool:
    """Check for model looping / repetition."""
    words = output.split()
    for i in range(len(words) - min_repeat):
        chunk = " ".join(words[i:i+min_repeat])
        if chunk in output[i+min_repeat:]:
            return False
    return True
```

## Writing a Hypothesis Test for LLM Output

A complete test file that checks invariants across randomly generated inputs:

```python
"""test_llm_invariants.py"""

import re
import json
from hypothesis import given, strategies as st, settings

from src.llm import openrouter_chat_model

model = openrouter_chat_model(temperature=0.5)


# ── Invariants ──────────────────────────────────────────────

def invariant_no_secrets(output: str) -> bool:
    patterns = [r"sk-[A-Za-z0-9]{20,}", r"AKIA[0-9A-Z]{16}"]
    return not any(re.search(p, output) for p in patterns)

def invariant_valid_json_when_requested(output: str) -> bool:
    stripped = output.strip()
    if stripped.startswith(("{", "[")):
        try:
            json.loads(stripped)
            return True
        except json.JSONDecodeError:
            return False
    return True

def invariant_within_word_limit(output: str, limit: int = 150) -> bool:
    return len(output.split()) <= limit


# ── Tests ───────────────────────────────────────────────────

@given(
    topic=st.text(min_size=1, max_size=100).filter(lambda t: t.strip() != ""),
    style=st.sampled_from(["concise", "detailed", "json"]),
)
@settings(max_examples=20)
def test_output_invariants(topic: str, style: str):
    style_prompts = {
        "concise": "Answer in 3 words or fewer: ",
        "detailed": "Explain briefly: ",
        "json": 'Return valid JSON with key "answer": ',
    }
    prompt = style_prompts[style] + topic
    response = model.invoke(prompt)
    output = response.content.strip()

    assert invariant_no_secrets(output), f"Secret leaked for topic={topic!r}"
    assert invariant_within_word_limit(output), f"Too long for topic={topic!r}"
    assert invariant_valid_json_when_requested(output), f"Invalid JSON for topic={topic!r}"
```

Run it:

```bash
uv run pytest test_llm_invariants.py -v
```

hypothesis will run 20 iterations with different generated inputs. If any iteration violates an invariant, hypothesis reports the minimal failing input and the call stack.

## Combining With the Golden Dataset

Property-based tests and golden datasets complement each other. The golden dataset (Layer 4) checks specific known cases — "does the model still answer capital-of-France correctly?" Property-based tests (Layer 3) check properties that must hold across _any_ input — "does the model ever leak a secret?"

Both should run in CI. The golden dataset gates on score regression. The property-based tests gate on invariant violations (which are always failures, not scores).

## Running in CI

```yaml
# .github/workflows/invariants.yml
name: Invariant tests
on: [pull_request]
jobs:
  invariants:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: |
          pip install uv
          uv sync
      - name: Run invariant tests
        run: uv run pytest tests/ -k invariant -v
        env:
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

## Common Pitfalls

**Too many examples.** `max_examples=20` is enough to catch most invariant violations. 100+ examples per test starts to cost real money in API calls. Start low, increase only if you suspect a rare edge case.

**Flaky invariants.** If your invariant depends on exact wording (e.g. "output must contain 'approved'") and the model sometimes uses synonyms, the invariant is too tight. Prefer semantic invariants (regex patterns, JSON validation, length bounds) over exact substring matches.

**Ignoring shrink output.** When hypothesis finds a failure, it _shrinks_ the input to the smallest example that still fails. Read the shrink output carefully — it tells you exactly which edge case triggers the invariant violation. That is more valuable than the failure itself.

**Temperature zero is not deterministic.** Even at temperature zero, most hosted models are not bit-reproducible. Batch inference, routing, and sampler implementation details change across requests. Run property-based tests at temperature > 0 to stress-test invariants under realistic variance.

## Companion Repo

The companion repo includes [`scripts/layers/03-property-based.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/03-property-based.py) and [`scripts/example_hypothesis.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/tools/example_hypothesis.py) with runnable examples you can adapt.

## Further Reading

- [hypothesis documentation](https://hypothesis.readthedocs.io/)
- [Testing LLM Outputs: The Eval Pyramid](../../ai-outputs-eval/)
- [DeepEval: CI-Gated Typed Metrics for LLM Outputs](../deepeval/)
