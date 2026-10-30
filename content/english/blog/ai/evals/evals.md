+++
date = '2025-07-02T10:25:00+10:00'
draft = false
title = 'Testing LLM Outputs: Evals for Models, Agents, and Skills'
tags = ['Claude', 'LLM', 'Testing', 'Evals', 'Agents']
summary = "LLM outputs are non-deterministic, which breaks the assumptions most of our testing tooling is built on. This document covers how to actually test model outputs, agent trajectories, and Claude Code-style skills — the layers of evaluation, the current 2026 tool landscape, and a working CI harness you can copy."
+++

![Class diagram of the LLM evaluation toolkit](../img/evals-class-diagram.png)

[Read Book here](https://drive.google.com/file/d/1ZcckmgQj2yYNYjJkC2ym5B0n9wobpUwO/view?usp=drive_link)

## Class Reference

Every box in the diagram above, and what it does. The toolkit is a Python harness (`evalkit`) built up over the course of the book — a dataset to run, scorers to judge with, and a runner to tie them together. Everything below is either one of those three, or a supporting piece: caching, cost accounting, regression gating, human labelling and production monitoring.

### Core types

| Class         | What it does                                                                                                                                                                                                                                                                                                                                                        |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ScoreType`   | Enum classifying what a scorer emits:<br/>**`binary`** [pass/fail, resolves to 1.0 or 0.0]<br/>**`numeric`** [continuous score on a 0.0 to 1.0 scale]<br/>**`likert`** [ordinal 1 to 5 rating, from terrible to excellent. Ordinal, so averaging it as if it were a ratio is not meaningful]<br/>**`categorical`** [un-ordered labels such as good, bad or neutral] |
| `EvalCase`    | A single test case: the input, an optional expected output, plus metadata and tags.                                                                                                                                                                                                                                                                                 |
| `EvalResult`  | Outcome of one case: output, per-scorer scores, pass flag, latency and error.                                                                                                                                                                                                                                                                                       |
| `EvalReport`  | Aggregate of a run: totals, pass rate, average scores and average latency.                                                                                                                                                                                                                                                                                          |
| `EvalConfig`  | Global session settings: directories, provider and model, concurrency, thresholds.                                                                                                                                                                                                                                                                                  |
| `Dataset`     | Named collection of `EvalCase` with load, filter, sample and serialize helpers.                                                                                                                                                                                                                                                                                     |
| `ScoreResult` | What a single scorer returns: normalized score, pass flag, reason and raw value.                                                                                                                                                                                                                                                                                    |

### Scorers

| Class                   | What it does                                                                            |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `Scorer`                | Abstract base every scorer implements: `name`, `score_type`, `threshold` and `score()`. |
| `ExactMatch`            | 1.0 if the output matches the expected answer exactly.                                  |
| `Contains`              | 1.0 if the output contains the expected substring.                                      |
| `RegexMatch`            | 1.0 if the output matches a supplied regular expression.                                |
| `AllOf`                 | Composite scorer: passes only when every child scorer passes.                           |
| `AnyOf`                 | Composite scorer: passes when at least one child scorer passes.                         |
| `ContainsAll`           | Checks that every required phrase appears, with optional per-phrase weights.            |
| `ContainsAny`           | Checks that at least one phrase from a set appears.                                     |
| `JsonMatch`             | Deep-compares parsed JSON against the expected value, ignoring key order.               |
| `LevenshteinSimilarity` | Edit-distance similarity: 1.0 minus the normalized distance.                            |
| `JsonSchemaValid`       | Validates the output against a JSON Schema.                                             |
| `EmbeddingSimilarity`   | Cosine similarity of embeddings, with a batched variant for throughput.                 |
| `LengthRange`           | Passes when the output length falls within the configured bounds.                       |
| `NoForbiddenContent`    | Fails if the output contains any banned string or pattern.                              |
| `PythonSyntaxValid`     | Extracts Python code blocks and checks that they compile.                               |
| `SqlSyntaxValid`        | Lightweight structural check on an SQL statement.                                       |
| `ContextRelevance`      | An LLM judges each retrieved chunk against the query, then aggregates.                  |
| `Faithfulness`          | Extracts claims from the answer and verifies each against the retrieved context.        |
| `AnswerCompleteness`    | Checks the answer addresses every part of the question.                                 |
| `ContextRecall`         | Measures whether retrieval surfaced the ground-truth contexts.                          |
| `ToolCallCorrectness`   | Compares actual tool calls to expected ones using longest-common-subsequence overlap.   |
| `TrajectoryEfficiency`  | Judges whether the path taken was efficient rather than wasteful.                       |
| `MultiTurnCoherence`    | Checks the conversation stays coherent and on-topic across turns.                       |
| `TaskCompletion`        | Judges whether the agent actually finished the assigned task.                           |
| `CachedScorer`          | Decorator that memoizes another scorer through a file-backed cache.                     |
| `LangfuseScorer`        | Wraps a scorer and mirrors its score onto a Langfuse trace.                             |
| `ScorerCache`           | SHA-256 keyed on-disk cache with hit and miss rate tracking.                            |

### LLM-as-judge and calibration

| Class                   | What it does                                                                |
| ----------------------- | --------------------------------------------------------------------------- |
| `JudgeCriterion`        | One rubric line: name, description, weight and the 1-to-5 scale text.       |
| `JudgeConfig`           | Judge settings: model, temperature, max tokens, criteria and pass score.    |
| `CalibrationCase`       | A case carrying both a human score and a judge score, for agreement checks. |
| `LLMJudge`              | Scores an output against a weighted rubric with a structured LLM call.      |
| `LengthNormalizedJudge` | Wraps a judge and penalizes answers far longer than the reference.          |
| `ConsensusJudge`        | Runs several judges and takes the median score for robustness.              |

### Guardrails

| Class                | What it does                                                              |
| -------------------- | ------------------------------------------------------------------------- |
| `Guardrail`          | Base synchronous output check returning an allow, block or modify action. |
| `ProfanityGuardrail` | Blocks or masks profanity found in the output.                            |
| `LengthGuardrail`    | Blocks or truncates the output once it passes a length limit.             |
| `PIIGuardrail`       | Detects and redacts emails, phone numbers, SSNs and similar patterns.     |
| `GuardrailResult`    | Outcome of one guardrail: action taken, reason, latency and output.       |
| `GuardrailPipeline`  | Runs guardrails in sequence, threading modified output through each.      |
| `PipelineResult`     | Final output, blocked flag, per-guardrail results and total latency.      |

### Runners

| Class              | What it does                                                                 |
| ------------------ | ---------------------------------------------------------------------------- |
| `Runner`           | Executes the system over a dataset with every scorer and returns a report.   |
| `CostAwareRunner`  | Runner that accrues token cost and stops once a budget is exhausted.         |
| `TieredRunner`     | Runs cheap scorers first and skips slow ones on clear failures.              |
| `ConcurrentRunner` | Async runner using a semaphore to issue concurrent LLM scorer calls.         |
| `CostEntry`        | One accounted call: model, input and output tokens, estimated cost, case id. |
| `CostReport`       | Aggregate cost and token totals, broken down by scorer and by model.         |

### Datasets and versioning

| Class                | What it does                                                               |
| -------------------- | -------------------------------------------------------------------------- |
| `DatasetVersion`     | Version metadata: name, case count, checksum, creation date and changelog. |
| `DatasetRegistry`    | Stores dataset versions with checksums and a queryable history.            |
| `SyntheticGenerator` | Generates LLM-written test cases at a chosen difficulty level.             |
| `ValidationIssue`    | One dataset quality problem: severity, case index and message.             |

### Regression and prompt versioning

| Class                    | What it does                                                                |
| ------------------------ | --------------------------------------------------------------------------- |
| `RegressionResult`       | Baseline diff: regression flag, metric deltas, new failures and new passes. |
| `RegressionDetector`     | Saves a golden baseline and compares current reports against it.            |
| `PromptComparisonResult` | Side-by-side A/B outcome for two prompt variants over one dataset.          |
| `PromptVersion`          | One prompt revision with author, changelog and measured pass rate.          |
| `PromptRegistry`         | Versioned prompt store with history and the current revision.               |

### RAG

| Class     | What it does                                                                   |
| --------- | ------------------------------------------------------------------------------ |
| `RAGCase` | A RAG eval case: query, retrieved contexts, generated answer and ground truth. |

### Agents

| Class             | What it does                                                                 |
| ----------------- | ---------------------------------------------------------------------------- |
| `ToolCall`        | A single tool invocation: name, arguments, result, latency and error.        |
| `AgentStep`       | One trajectory step: a thought, a tool call, a tool result or a response.    |
| `AgentTrajectory` | A full agent run: steps, final answer, tokens, latency and accessor helpers. |

### Human annotation

| Class             | What it does                                                                   |
| ----------------- | ------------------------------------------------------------------------------ |
| `Annotation`      | A human label on a case: score, label, reasoning and who produced it.          |
| `AnnotationTask`  | One case plus its surrounding context, queued for human scoring.               |
| `AnnotationBatch` | A batch of annotation tasks with status and a completion rate.                 |
| `AgreementReport` | Inter-annotator metrics: exact and within-one rates, spread and Cohen's kappa. |

### Production monitoring

| Class               | What it does                                                       |
| ------------------- | ------------------------------------------------------------------ |
| `SampledRequest`    | A sampled production request together with the scores it received. |
| `ProductionSampler` | Deterministically samples traffic and scores it off the hot path.  |
| `QualityWindow`     | Scores and pass rate over a time window, with alert state.         |
| `QualityMonitor`    | Aggregates windows and raises an alert when the pass rate drops.   |

### Integrations and workflows

| Class                     | What it does                                                                 |
| ------------------------- | ---------------------------------------------------------------------------- |
| `EvalKitToDeepEvalMetric` | Adapter exposing an evalkit `Scorer` as a DeepEval metric.                   |
| `ExperimentRecord`        | One logged experiment: model, prompt version, pass rate and regression flag. |
| `EDDWorkflow`             | The eval-driven development loop: run, compare, record and pick a best.      |
| `SupportBot`              | Case-study support agent with tools, iterated to 99% reliability.            |

## Part 1: Why This Is a Different Testing Problem

- **Traditional software testing** is based on **determinism** of systems - `assertEqual(f(x), y)` works because `f` is deterministic.
- The moment you put an **LLM in the equation, this assumption breaks**. Ask Claude the same question twice, at the same temperature and you can get two answers that are both "correct" but not identical — different wording, different tool-call order, different length.
- Ask an **agent to complete a multi-step task** and you get a **trajectory**, not a single output: a chain of reasoning, tool calls and intermediate states that can diverge wildly between runs while still arriving at a valid result.
- You're not testing for equality anymore (`unittest`/`pytest` in their classic form) — you're testing for **membership in an acceptable set**, scored probabilistically. That reframing is what "evals" are.

In Summary: Three things compound the problem:

1. **Stochasticity is layered.**
   - Stochasticity is the quality of lacking a predictable pattern, where outcomes are governed by probability rather than deterministic rules.
   - The model call is non-deterministic, and if you use another LLM to _grade_ the output (LLM-as-judge), the grader is non-deterministic too.Stack enough randomness and your test suite becomes noise unless you control for it.
2. **Agents multiply the surface area.**
   - A single-turn LLM call has one input and one output to score.
   - An agent has N tool calls, each of which can fail, retry, hallucinate a tool name or call the right tool with the wrong arguments — and two completely different trajectories can both be "correct."
3. **Correctness is often a spectrum, not a boolean.**
   - A summary can be accurate but too long. A generated skill can succeed but use twice the tokens it needed to. You need scored metrics, not pass/fail, for most of what matters.

## Part 2: The Layers of LLM Output Testing

No single technique covers everything. In practice you build a pyramid, and the shape of that pyramid matters more than any individual tool you pick.

### Pyramid

```
L6          X        Human review (spot-check samples)
L5         XXX       Statistical sampling (N runs at temp > 0)
L4        XXXXX      Golden dataset (curated specs, regression gate)
L2       XXXXXXX     Model-graded (semantic judgment, per-PR only)
L1+L3   XXXXXXXXX    Deterministic + invariants
```

Each layer up has fewer X's (fewer checks), costs more per check, and runs less frequently. e.g. The pile-up at the base catches the 80% that can be expressed as code assertions. The tip catches what slips through — but you need that tip to be narrow, because expensive checks at scale become noise.

### Relationship.

```mermaid
flowchart BT
    L6["Layer 6 — Human-in-the-Loop"]
    L5["Layer 5 — Statistical Sampling"]
    L4["Layer 4 — Golden Datasets &amp; Regression Tracking"]
    L3["Layer 3 — Property-Based / Invariant Testing"]
    L2["Layer 2 — Model-Graded (LLM-as-judge)"]
    L1["Layer 1 — Deterministic / Structural Checks<br/>~(60-80% of eval axes)"]

    L6 -->|depends on| L4
    L5 -->|depends on| L4
    L4 -->|each golden case runs| L1
    L4 -->|each golden case runs| L2
```

### Layer 1 — Deterministic / Structural Checks (cheapest, do these first)

- Anything that _can_ be checked without another model call. These are fast, free and have zero grader-side noise.
- The generation and the check are separate steps — the model call produces the output (non-deterministic), and then the check inspects it with plain code (deterministic).

Examples:

- Output parses as valid JSON / matches a schema - e.g. for a prompt that produces json output
- Required fields are present and correctly typed - e.g. word 'Paris' is capital of France.
- Code compiles / lints / passes type checks - e.g. generated python code for fibonacci.
- The agent called the tool it was supposed to call, with the arguments it was supposed to pass
- Output length within bounds e.g. for a summarization task
- No banned strings e.g. PII patterns (Email + Phone number, SSN, TFN), forbidden phrases
- Regex / substring matches for known-good or known-bad patterns e.g. _secret detection:_ re.search(r"sk-[A-Za-z0-9]{20,}", output)
- Latency and token-cost thresholds

If you can express the check as code, do it. This layer should be the majority of your suite

> **Few references:** [FutureAGI (Feb 2026)](https://futureagi.com/blog/deterministic-llm-evaluation-metrics-2026/) reports deterministic checks catch _30 to 60 percent of failures before any LLM judge fires_. The [G-Eval production guide (Mar 2026)](https://futureagi.com/blog/g-eval-definitive-guide-2026/) recommends routing every response through a deterministic floor (schema, regex, length, banned phrases) before any LLM judge call, dropping the judge bill 80–90% without losing detection rate. Both sources converge on the same finding: 60–80% of the scoring criteria in a typical eval suite can be implemented as plain code rather than requiring an LLM judge. The consensus: deterministic checks are the cheapest, fastest, and most reliable layer — they should be the base of every eval pyramid.

[runnable example defines the same four scorers (`score_contains`, `score_excludes`, `score_max_words`, `score_valid_json`), applies them to a 5-case golden dataset, and prints results.](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/01-deterministic.py#L22-L39)

Each scorer is a plain Python function — no LLM, no API call, no judge model. They are composable: a single golden case can combine `score_contains` + `score_max_words` + `score_excludes` and the overall score is simply the mean of the individual checks.

The tools differ by language, but the categories are the same. The following table maps common Layer 1 checks to practical technologies:

| Category                          | Python                                                                                                                                                                                         | Node.js / TypeScript                                                                                                                  | Java                                                                                                                                                   |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Test runner                       | [`pytest`](../pytest/) or `unittest` — runs deterministic assertions, fixtures and parametrized cases in CI.                                                                                   | `Vitest` or `Jest` — runs unit tests, assertions, mocks and snapshots for JavaScript or TypeScript.                                   | `JUnit 5` — runs unit tests, assertions, parameterized tests and extensions through Maven or Gradle.                                                   |
| Schema and data validation        | [`pydantic`](../pydantic/) or `jsonschema` — validates parsed model output against typed Python models or JSON Schema.                                                                         | `Zod`, `Ajv` or `TypeBox` — validates JSON at runtime and can connect schemas to TypeScript types.                                    | Jackson plus `jakarta.validation` or a JSON Schema validator — parses JSON and checks fields, formats and constraints.                                 |
| Static analysis and type checking | `mypy` or `pyright` — static type checkers that inspect Python without executing it and catch incompatible arguments, returns and assignments. `ruff` adds fast syntax, lint and style checks. | `tsc --noEmit` — checks TypeScript types without emitting JavaScript. `ESLint` catches common errors and enforces code rules.         | `javac` or the Maven/Gradle compiler — catches Java syntax and compile-time type errors. `Checkstyle` and `SpotBugs` add source and bytecode analysis. |
| Property-based testing            | [`hypothesis`](../hypothesis/) — generates many inputs, including edge cases, and shrinks failures to smaller examples.                                                                        | `fast-check` — generates and shrinks values for property-based tests and works with Vitest, Jest or other runners.                    | `jqwik` — generates inputs and action sequences for JUnit Platform properties, then shrinks failing cases.                                             |
| Agent and tool-call checks        | [`toolcallcheck`](../toolcallcheck/) or a mock MCP server — records tool calls and checks names, arguments, order and forbidden actions.                                                       | Mock tool handlers with `Vitest`, `Jest`, `MSW` or `nock` — verifies requests, arguments and response handling without live services. | `Mockito`, `WireMock` or a custom fake server — verifies collaborators, HTTP calls, arguments and failure recovery offline.                            |
| Simple output invariants          | Plain `assert`, regular expressions and standard-library parsers — checks length, banned strings, secret patterns and valid JSON.                                                              | `expect`, regular expressions and `JSON.parse` — checks required text, forbidden patterns, output bounds and parseability.            | JUnit assertions, regular expressions and Jackson — checks required fields, forbidden patterns, output bounds and parseability.                        |

These tools check the generated artifact after the model produces it. For example, an eval can ask a model to generate a Python script, run `mypy` and `ruff` against the saved script, then execute it only if those checks pass. The same workflow applies to generated TypeScript with `tsc --noEmit` or generated Java with the project compiler and static-analysis tools. SQL and configuration files need language-specific parsers, linters or validators rather than Python, Node.js or Java type checkers.

### Layer 2 — Model-Graded Evaluation (LLM-as-judge)

For anything semantic (involves understanding the meaning) e.g. correctness against a rubric, tone, faithfulness to source material, whether a response actually answers the question — you use another LLM call to score the output. [Code example](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/02-model-graded.py#L17-L30):

The same output is then scored on three axes: is it _correct_ (factually accurate), _helpful_ (directly addresses the question), and _concise_ (avoids unnecessary verbosity). The judge prompt, the scoring rubric, and the output format are all explicit — no room for vibes.

Things that make this reliable instead of vibes-based:

- **Use a rubric, not a vague instruction.** instead of asking "Is this good?" - "Score against these five specific criteria" is a good one.
- **Calibrate the judge against human labels.** Periodically have a human score a sample and check the judge agrees e.g.
  - sample random 20 cases weekly and compare;
  - after editing the judge prompt or rubric (the judge is an LLM call — changing its prompt changes its scoring behavior, so re-validate against human labels);
  - when golden dataset scores drift unexpectedly without a corresponding change to the generator.
  - If the judge disagrees with the human on more than ~10% of samples, fix the rubric before trusting the automated score.
- **Watch for judge biases**: Randomize order, control for length, and occasionally cross-check with a different model as judge. e.g. of Biases:
  - position bias (favoring the first option in a comparison)
  - length bias (favoring longer answers)
  - self-preference bias (a model judge favoring outputs from its own family).
- **Never let this be your only signal.** Model-graded evals add noise on top of the agent's own noise. Deterministic checks are your ground truth; LLM-as-judge fills the gap deterministic checks can't reach.

Tools that implement this:

- **[`deepeval`](../deepeval/) (`GEval`, `FaithfulnessMetric`, `HallucinationMetric`, `AnswerRelevancyMetric`)** — pre-built model-graded scorers that call an LLM judge internally and return a numeric score you can assert against; no need to write your own judge prompt for common patterns.
- **[`promptfoo`](../promptfoo/)** — runs model outputs through comparison-based scoring or adversarial test cases; strongest for catching regressions when swapping prompts or models, with built-in red-teaming vectors.
- **[`braintrust`](../braintrust/)** — custom sandboxed Python scorers where you define the judge logic (call any model, run any calculation); scores are wired into CI gates and regression dashboards automatically.
- **`langsmith`** — annotation queues let humans review model-graded scores and correct them; closed-loop feedback refines the judge prompt over time.
- **`langfuse`** — model-as-judge evaluation built into the tracing platform; useful when you already use Langfuse for production monitoring and want eval without a second service.
- **`ragas`** — faithfulness, answer relevancy, and context precision metrics specifically for RAG pipelines; uses LLM calls internally but scoped to the retrieval-grounded evaluation domain.

G-Eval (or GPT Based Evaluation or chain-of-thought-based scoring with a scoring function) is the most common pattern for structuring judge prompts well — it asks the judge to reason step by step against explicit criteria before emitting a score, which measurably improves judge-human agreement over a bare "rate 1-5" prompt.

### Layer 3 — Property-Based / Invariant Testing

[Code example](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/03-property-based.py#L27-L62).

Instead of checking a specific output, check properties that must hold across _all_ outputs regardless of variance:

- The agent never calls a destructive action (delete, deploy, send) without an explicit confirmation step
- Idempotency: calling the same tool twice with the same arguments doesn't double-charge, double-send, or duplicate a record
- No response exceeds a hard token/length ceiling
- Every generated SQL query is read-only when the task doesn't require writes
- A generated skill never writes outside its declared working directory

This is where a lot of practical agent _safety_ testing actually lives — you're not testing quality, you're testing that the guardrails hold no matter what path the agent takes to get there.

> The implementation looks the same as Layer 1 (regex, file checks, no LLM), but the _target_ differs. A deterministic check answers "given _this specific input_, does the output pass?" — it runs against golden dataset cases and tests for expected behavior. A property-based invariant answers "does this property hold for _every possible input_?" — it uses random/edge-case input generation (e.g. Hypothesis) to stress-test the boundary. Example: a deterministic check says "for this pet-store spec, every path has a route file." A property-based invariant says "for _any_ spec, the skill never writes files outside allowed directories." One verifies correctness for a known case; the other verifies safety across unknowns.

Tools that implement this:

- **`hypothesis`** — property-based testing framework; you declare invariants ("output never contains X") and it generates inputs to try to violate them, including edge cases you wouldn't think to write manually.
- **[`toolcallcheck`](../toolcallcheck/)** — assert structural constraints on tool calls (e.g. "never call the delete tool" or "always pass a confirmation flag"); fails the test if the agent's trajectory violates the constraint, regardless of output content.
- **Custom `PreToolUse` / `PermissionDenied` hooks** — in `claude-code` or `opencode`, these hooks fire before every tool call and can block it deterministically — you enforce invariants at the agent runtime level, not just in tests.
- **`pytest` with invariant fixtures** — write a fixture that runs post-test (e.g. `yield` + assert pattern); every test in the suite automatically checks the invariant without duplicating the assertion.
- **`deepeval` (`BiasMetric`, `ToxicityMetric`)** — model-graded safety metrics that score output against fairness and toxicity rubrics; useful when the invariant is semantic rather than structural.
- **`promptfoo` red-teaming suite** — 500+ adversarial input vectors that probe whether your invariant holds under malicious or unusual inputs; automated probing, not manual test writing.

### Layer 4 — Golden Datasets and Regression Tracking

- [Code examples](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/04-golden-dataset.py)
- [Full implementation](https://github.com/beautifularchitecture/baba-yaga/blob/main/src/eval.py).
- Curate a set of real inputs — pulled from actual usage rather than invented — and snapshot how your system scores against them over time.
- here you are asserting the **eval score on that set doesn't regress (go down)** when you change a prompt, swap a model version or edit a skill definition.
- The discipline that matters here: **score your current production system before setting an acceptance bar.** Don't invent a target in a vacuum — measure the baseline, then gate merges on "no regression below baseline" rather than an arbitrary absolute number. E.g. _"tool-selection success: current baseline 88%, acceptance bar 95%, anything below baseline blocks deploy."_

Tools that implement this:

**`deepeval` (`Dataset`)**

- **What it does:** curate input-output pairs as a typed `LLMTestCase` dataset and run evals against the whole set as a batch. Includes synthetic data generation from your existing docs to expand coverage without manual curation. Built-in scorers cover G-Eval, hallucination, faithfulness, answer relevancy, and 50+ others.
- **Best for:** staying in pytest, CI gating without external infra, teams that want synthetic coverage without writing every test case by hand.
- **Downside:** built-in scorers are opinionated — non-standard checks are harder to wire in. No dashboard or historical tracking beyond CI logs.

**`promptfoo`**

- **What it does:** YAML-defined test case datasets with expected outputs or rubric criteria. Built-in regression comparison across runs shows you score deltas for every prompt or model change — run from the terminal, see "model A: 0.91, model B: 0.85" before you commit. Strongest open-source red-teaming suite (500+ adversarial input vectors).
- **Best for:** prompt iteration, model comparison, adversarial testing.
- **Downside:** no database, no web UI, no cross-run history tracked by default. Results live in local JSON — regression tracking across weeks is manual unless you build a wrapper.

**`braintrust`**

- **What it does:** dataset-first eval platform. Every eval run is recorded and automatically linked to the git commit that triggered it. Web dashboards flag any score drop compared to the previous commit or baseline — you don't diff JSON files by hand. Custom sandboxed Python scorers let you call any model or run any calculation as the judge.
- **Best for:** teams that need persistent eval history, commit-linked dashboards, and CI gating at scale.
- **Downside:** requires a platform account and setup. Heavier than a local pytest script. Overkill for a solo developer running evals once a week.

**`langsmith`**

- **What it does:** dataset management with versioning and comparison views. If you're already tracing production traffic through LangSmith, those traces auto-become eval datasets — you're not writing test cases from scratch. Side-by-side score breakdowns across model versions, prompt variants, or dataset revisions in the web UI.
- **Best for:** teams already on LangChain/LangGraph who don't want a second platform.
- **Downside:** locked into the LangChain ecosystem. Platform pricing can get expensive at scale. Not worth adopting LangChain just for the eval layer.

**`pytest-snapshot`**

- **What it does:** golden-file plugin for pytest. Captures exact outputs as snapshot files and asserts they haven't changed — on failure it shows a diff so you can review and accept the change.
- **Best for:** structural outputs (configs, schemas, generated code) where byte-for-byte equality is meaningful.
- **Downside:** only works for deterministic structural output. Useless for semantic evaluation — asserting exact output equality on LLM responses defeats the whole purpose of non-deterministic scoring.

**Custom YAML gates**

- **What it does:** define your baseline and acceptance threshold in a version-controlled YAML file checked into the repo. A one-page CI script checks "does the current score stay above the saved baseline?" before every merge. No platform dependency, works with any eval scorer.
- **Best for:** minimal setups where the discipline of "nothing below our measured baseline merges" is the goal — the companion repo does exactly this with `eval-baseline.json` in git and `--gate` in CI.
- **Downside:** no web UI, no dashboards, no cross-team visibility. You build the runner yourself — but the total surface fits in a single file.

> regression is an overloaded term:
> Statistical regression (ML) — modeling relationships between variables to predict a continuous value (e.g. house prices from sq footage, bedrooms). "Regression" in this sense is about fitting a function.
> Regression testing / score regression (eval) — a change causing performance to degrade relative to a known baseline. The "regress" here is literal: things went backward.

### Layer 5 — Statistical Sampling

- [code example](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/05-statistical-sampling.py#L66-L78)
- A single run tells you almost nothing at temperature > 0. Run each test case N times (5–20 is typical) and look at the **distribution** of scores, not one output:
- Report pass-rate ("87% of runs satisfied the rubric") instead of pass/fail
- Track variance, not just mean — a task with 90% mean but huge variance is riskier than 85% with low variance
- This matters disproportionately for agents, since errors compound across turns — a 95%-reliable single tool call becomes a 60%-reliable five-step trajectory (0.95⁵ ≈ 0.77, and it gets worse fast as steps increase and per-step reliability drops)

Tools that implement this:

- **`pytest` + `statistics`** — wrap any test in a parametrized loop over N seeds and aggregate scores with the built-in `statistics` module (mean, stdev, pass-rate); no extra dependency, works in any CI as shown in Part 8.
- **`deepeval`** — confidence intervals and statistical significance reporting built into its metrics; when you run N times it reports the distribution, not just the average, and can gate on the lower bound of the confidence interval.
- **`hypothesis`** — controls sample counts and random seeds as part of its property-based testing engine; useful when you want to combine invariant testing with sampling (e.g. "run this invariant 20 times across different random seeds").
- **`pytest-repeat`** — minimal plugin that re-runs a test N times; good for quick iteration before moving to a more structured sampling approach.
- **`promptfoo`** — `repeat: N` in the test config runs each case N times and aggregates pass-rates, latencies, and scores across runs in the output report.
- **Custom CI gates** — scripts that compare pass-rate distributions between the feature branch and main; block if the lower bound of the confidence interval drops below the baseline, regardless of the mean.

### Layer 6 — Human-in-the-Loop

- [Code Example](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/06-human-review.py#L64-L83)
- [exports eval results to CSV for annotation, then reports disagreements between human and automated scores](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/layers/06-human-review.py#L23-L32).
- No rubric anticipates every edge case. Sample production traffic (5–10% is a common starting point) for manual review and feed disagreements between human and judge scores back into refining the rubric. This is also where you catch the failure modes that are only obvious to a domain expert — a technically well-formed answer that's subtly wrong in a way no automated check would flag.

Tools that implement this:

- **`langsmith`** — annotation queues for collecting human feedback at the trace or thread level; reviewers score individual LLM calls or entire agent runs, and disagreement with the automated judge feeds back into rubric refinement.
- **`braintrust`** — human annotation UI with reviewer assignment and audit trails; scores from reviewers are automatically compared against model-graded scores, and dashboards surface the disagreement rate so you know which rubrics need fixing.
- **`langfuse`** — manual scoring workflows where a human can annotate traces after the fact; integrated with the same UI used for production monitoring, so reviewers see the full context of the agent run.
- **`label-studio`** — general-purpose annotation platform; configure any custom review pipeline (e.g. "review 10% of outputs flagged as low-confidence by the automated judge") with your own scoring rubric UI.
- **`arize-phoenix`** — production trace sampling (e.g. 5% of all agent runs) surfaced for human-in-the-loop scoring; uses OpenTelemetry so sampling config works across any instrumented stack without platform-specific wiring.

## Enterprise Minimal Tool Set

If you need to cover all 6 layers in an enterprise setting with the smallest surface area, this is the practical minimum:

Runnable examples for each tool are in the companion repo at `scripts/tools/` (linked in the tables below). All are open-source except Braintrust — you can defer it by checking baseline JSON into git, adding it later when you need dashboards and team-wide visibility.

### Full Tool Features Matrix

| Tool                                   | Deterministic | LLM judge | Property-based | Golden dataset | Sampling | Human review | Red-teaming | Mock offline | Synthetic data |
| -------------------------------------- | ------------- | --------- | -------------- | -------------- | -------- | ------------ | ----------- | ------------ | -------------- |
| **[pytest](../pytest/)**               | ✅            | ❌        | ✅             | ❌             | ✅       | ❌           | ❌          | ❌           | ❌             |
| **[DeepEval](../deepeval/)**           | ✅            | ✅        | ✅             | ✅             | ✅       | ❌           | ❌          | ❌           | ✅             |
| **[Promptfoo](../promptfoo/)**         | ❌            | ✅        | ✅             | ✅             | ✅       | ❌           | ✅          | ❌           | ❌             |
| **[Braintrust](../braintrust/)**       | ❌            | ✅        | ❌             | ✅             | ❌       | ✅           | ❌          | ❌           | ❌             |
| **[hypothesis](../hypothesis/)**       | ✅            | ❌        | ✅             | ❌             | ✅       | ❌           | ❌          | ❌           | ❌             |
| **[toolcallcheck](../toolcallcheck/)** | ✅            | ❌        | ✅             | ❌             | ❌       | ❌           | ❌          | ✅           | ❌             |
| **langsmith**                          | ❌            | ✅        | ❌             | ✅             | ❌       | ✅           | ❌          | ❌           | ❌             |
| **langfuse**                           | ❌            | ✅        | ❌             | ❌             | ❌       | ✅           | ❌          | ❌           | ❌             |
| **ragas**                              | ❌            | ✅        | ❌             | ❌             | ❌       | ❌           | ❌          | ❌           | ❌             |
| **jsonschema / pydantic**              | ✅            | ❌        | ❌             | ❌             | ❌       | ❌           | ❌          | ❌           | ❌             |
| **mypy / pyright / ruff**              | ✅            | ❌        | ❌             | ❌             | ❌       | ❌           | ❌          | ❌           | ❌             |
| **label-studio**                       | ❌            | ❌        | ❌             | ❌             | ❌       | ✅           | ❌          | ❌           | ❌             |
| **arize-phoenix**                      | ❌            | ❌        | ❌             | ❌             | ❌       | ✅           | ❌          | ❌           | ❌             |
| **pytest-snapshot**                    | ✅            | ❌        | ❌             | ❌             | ❌       | ❌           | ❌          | ❌           | ❌             |
| **pytest-repeat**                      | ❌            | ❌        | ❌             | ❌             | ✅       | ❌           | ❌          | ❌           | ❌             |

## Part 3: Evaluating Agents Specifically — Trajectory, Not Just Output

- [Code Example](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/trajectory/trajectory_eval.py).
- Agent evaluation is a strictly harder problem than single-call LLM evaluation, because the thing you're scoring isn't a string — it's a **trajectory**: the full sequence of reasoning steps, tool calls, and intermediate states. LangChain's framing is a useful mental model here: scoring only the final answer is like grading an exam by the final grade alone; trajectory evaluation is grading by the working shown at every step.

What trajectory evaluation actually checks:

- **Tool selection correctness** — did it call the right tool for the sub-task, not just _a_ tool
- **Argument correctness** — right tool, wrong arguments still fails
- **Step efficiency** — did it take 3 steps or 11 to do the same job (token cost and latency both suffer from wandering)
- **Recovery behavior** — when a tool call fails, does the agent retry sensibly, fall back, or ask the user, versus looping or hallucinating a workaround
- **Path validity** — two different trajectories can both be correct (there's rarely one "right" path), so scoring needs to allow for legitimate divergence rather than exact-path matching

Frameworks like [`agentevals`](https://docs.langchain.com/oss/python/langchain/test/evals) (LangChain's ecosystem) and Strands Evals read traces natively if you're already building on those frameworks — check what your agent framework ships before adding a separate dependency, since many now include a first-party eval subpackage.

## Part 4: Testing Claude Code Skills Specifically

Skills are a narrower, more testable case than open-ended agents, because a skill has a declared contract: given this trigger condition, it should read this SKILL.md, follow this procedure, and produce this class of output. That contract is exactly wh at you eval against.

A practical skill-testing checklist:

1. **Trigger accuracy** — does the skill activate on the inputs it should and _not_ activate on inputs it shouldn't (false positives are as costly as false negatives (front matter - `disable-model-invocation: false` based on invocation by intend) — an overtriggered skill burns context and can derail an otherwise fine response)
2. **Procedure adherence** — did the model actually follow the steps documented in the skill, or ignore them and improvise
3. **Output contract** — if the skill promises a specific file type, directory location, or format, verify that mechanically (file exists, correct extension, correct location, opens without error)
4. **Boundary respect** — a skill that touches the filesystem should never write outside its declared scope; test this as a hard invariant, not a soft check
5. **Token efficiency** — skills that reference other files or run multi-step procedures should be checked for whether they're pulling in more context than the task needs

> For skills, deterministic checks carry even more of the weight than in general agent testing, because the contract is narrower and more mechanically verifiable — "did it produce a valid .docx with a table of contents" is a much cheaper and more reliable check than asking a judge model "is this a good Word document."

### Practical Testing by Platform

The six-layer pyramid applies regardless of what you're building — but the _tool_ you use to implement each layer changes depending on whether you're testing a Claude Code skill, a hook function, or a Copilot instructions file. Here is the mapping per platform.

**Claude Code skills (`~/.claude/skills/<name>/SKILL.md`)**

A skill has a narrow contract: trigger condition, documented procedure, output promise. That contract is your eval spec, and deterministic checks cover most of it.

| Layer              | How to test it                                                                                                                                                                                                                            |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 — Deterministic  | Create a test repo, run the skill via `claude` in headless mode, assert the output file exists at the right path with the right format. Check tool-call order against the skill's documented steps with `toolcallcheck` or a custom mock. |
| 2 — Model-graded   | Only needed if the skill produces open-ended output (e.g. "write a summary"). Use the same LLM-as-judge pattern as `eval.py` — call the model with a rubric, score the output.                                                            |
| 3 — Invariants     | A `PreToolUse` hook that blocks writes outside the skill's scope _is_ the test — add a test that the hook fires when it should and doesn't when it shouldn't.                                                                             |
| 4 — Golden dataset | Add each skill's output contract checks to `eval.py` as new golden cases. If a skill change breaks the contract, `--gate` catches it before merge.                                                                                        |
| 5 — Sampling       | Run the skill N times. If the output is deterministic (file path, format), one run is enough. If the skill calls an LLM internally, sample 3–5 runs.                                                                                      |
| 6 — Human review   | Review baseline changes in PRs — if a skill change drops a score, a human decides whether the new behaviour is acceptable before updating the baseline.                                                                                   |

**Claude Code hooks (`~/.claude/hooks/<name>`)**

Hooks (`PreToolUse`, `PostToolUse`, `PermissionDenied`) are deterministic Python or TypeScript functions. They take a context object and return a decision. No LLM is involved — test them like regular unit tests.

| Layer             | How to test it                                                                                                                                                                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 — Deterministic | Mock the tool-call context, call the hook, assert the return value (allow / block / transform). No model call needed. Example: a `PreToolUse` hook that blocks `rm -rf` — unit test passes it a `rm -rf /` command and asserts `PermissionDenied`. |
| 2–6               | Not applicable. Hooks are pure logic. If they call an external API or LLM (they shouldn't — that defeats the purpose of a synchronous guard), those calls become integration tests under Layer 1 with a mocked network.                            |

**System-prompt overlays (`AGENTS.md`, `copilot-instructions.md`, `.github/instructions/`)**

Files like `AGENTS.md` and `copilot-instructions.md` are system-prompt overlays applied on every task — they change the model's default behaviour globally. If one is wrong, it degrades everything, which makes regression gating more important here than anywhere else.

| Layer                               | How to test it                                                                                                                                                                                                                                                                                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1 — Deterministic                   | Add a golden case that checks the overlay file itself — does it parse? are required sections present? does it reference tools or skills that actually exist?                                                                                                                                                                                     |
| 2–4 — Model-graded + golden dataset | Run your existing golden dataset twice: once with the overlay prepended to each prompt, once without (the baseline). Compare the scores. If adding the overlay degrades accuracy on cases that shouldn't be affected, gate the change. This catches things like an `AGENTS.md` that accidentally suppresses tool-calling for cases that need it. |
| 5 — Sampling                        | Same as any golden dataset run — 3–5 samples per case with `--gate` against the no-overlay baseline.                                                                                                                                                                                                                                             |
| 6 — Human review                    | Review the diff to the overlay file in PRs. If it's hard to tell whether a change improves or harms output, that's a signal the golden dataset needs a new case covering that scenario.                                                                                                                                                          |

The tactic is identical for `AGENTS.md`, `copilot-instructions.md`, and any file that injects instructions at the model level: golden dataset with/without the overlay, `--gate` on any regression.

**Custom agents (`~/.claude/agents/`, `.github/agents/`, OpenCode agents, etc.)**

An agent has three testable surfaces: the _instructions_ (system prompt), the _tool definitions_ (what tools it can call), and the _trajectory_ (how it chains them together). Testing only the final output misses most agent-specific failure modes — a model can say the right thing but call the wrong tool, or call the right tool with hallucinated arguments.

| Layer              | How to test it                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 — Deterministic  | Mock the tool server (MCP or custom). Assert the agent called the right tools with the right arguments in the right order — regardless of what the LLM output actually _said_. `toolcallcheck` does this fully offline, no model call. Also check invariants: "never calls the delete tool" enforced via `PreToolUse` hook. Assert step efficiency — did it take 3 tool calls or 11 to do the same job? |
| 2 — Model-graded   | For open-ended agent output (summaries, decisions, generated code), use an LLM judge with a rubric. Score the _final output_ but also score intermediate tool-call results — did each sub-step produce something valid before the next one started?                                                                                                                                                     |
| 3 — Property-based | Feed the agent edge-case inputs (empty input, malformed data, adversarial prompts). Assert invariants hold across all of them: no destructive tool calls, no PII leaked, no writes outside the working directory. Run these with `hypothesis` to generate the edge cases automatically.                                                                                                                 |
| 4 — Golden dataset | Same `eval.py` pattern. Each golden case is a full task — e.g. "generate a report from this template with these data files". The score is a composite of tool-call correctness + final output quality + step efficiency. Run N times, record baseline, `--gate` on regression.                                                                                                                          |
| 5 — Sampling       | Agent trajectories are where non-determinism compounds hardest — a 95%-reliable single tool call becomes 60% reliable across 10 steps (0.95¹⁰ ≈ 0.60). Run each golden case 5–10 times and gate on the lower bound of the confidence interval, not the mean.                                                                                                                                            |
| 6 — Human review   | Agent failures are harder to diagnose than single-call failures. Log every trajectory (tool calls + reasoning + timestamps) to a file per run. A human reviews regressions by replaying the trajectory, not guessing what went wrong.                                                                                                                                                                   |

The critical insight for agents: Layer 1 (tool-call correctness) is where most agent bugs live, not Layer 2 (output quality). A mock tool server that records and validates every tool call catches the majority of agent failures without ever calling a model as judge.

The unifying thread across all platforms: the same `eval.py` + `eval-baseline.json` + `--gate` pattern works for everything described in this post. The golden dataset covers the model-facing surface; deterministic `assert` calls in unit tests cover the logic surface; mock tool servers cover trajectory correctness; the baseline file and CI gate tie them together into a regression-proof workflow.

## Part 5: Measuring Effectiveness from Session History

The eval pyramid tells you whether quality _regressed_. It does not tell you whether productivity _improved_. For that you need a before-and-after measurement on real usage data, not synthetic golden cases.

### What session logs contain

Claude Code writes session transcripts as JSONL files (one JSON object per line) to `~/.claude/projects/<url-encoded-project-path>/<session-id>.jsonl`. Each line represents a message in the conversation with role, content blocks, and metadata:

- **Role** — `"user"`, `"assistant"`, or `"system"`
- **Content blocks** — text, `tool_use` (name + input), `tool_result` (content + optional `is_error`)
- **Model metadata** — model version, usage (tokens), stop reason

```jsonl
{"role": "user", "content": [{"type": "text", "text": "generate a compliance report"}]}
{"role": "assistant", "content": [{"type": "text", "text": "I'll help with that."}, {"type": "tool_use", "name": "read_file", "input": {"path": "template.md"}}]}
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_abc", "content": "---\ntitle: Report\n---", "is_error": false}]}
```

### Before-and-after comparison

Take a baseline window (e.g. 2 weeks before introducing a skill) and a measurement window (2 weeks after). Extract the same metrics from both across all session files in `~/.claude/projects/*/`:

```bash
# Total tool calls across all sessions — measures automation depth
find ~/.claude/projects -name '*.jsonl' -exec sh -c '
  jq -s "[.[] | .content // [] | map(select(.type == \"tool_use\")) | length] | add" "$1"
' _ {} \; | awk '{sum+=$1; count++} END {print "Avg tool calls/session:", sum/count}'

# Sessions with errors — measures reliability
find ~/.claude/projects -name '*.jsonl' -exec sh -c '
  jq -s "[.[] | .content // [] | map(select(.type == \"tool_result\" and .is_error)) | length > 0] | any" "$1"
' _ {} \; | awk '/true/ {errors++} END {print "Error rate:", errors/NR}'

# Total tokens used (from last assistant message per session)
find ~/.claude/projects -name '*.jsonl' -exec sh -c '
  jq -s "[.[] | select(.role == \"assistant\") | .usage // {} | .output_tokens // 0] | add" "$1"
' _ {} \; | awk '{sum+=$1} END {print "Total tokens:", sum}'
```

The report structure for each measurement:

| Metric                  | Before (no skill) | After (with skill) | Delta |
| ----------------------- | ----------------- | ------------------ | ----- |
| Avg task duration       | 4m 12s            | 2m 08s             | −49%  |
| Avg tool calls per task | 8.3               | 4.1                | −51%  |
| Error rate              | 12%               | 4%                 | −67%  |
| First-attempt pass rate | 0.60              | 0.88               | +47%  |
| Tokens per task         | 8 200             | 4 500              | −45%  |

The last row (first-attempt pass rate) is the direct link back to the eval pyramid — it is the same score your golden dataset measures, now computed on production traffic instead of synthetic cases.

### Automated via a session-analysis script

The companion repo includes [`scripts/sessions/analyze_sessions.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/sessions/analyze_sessions.py) that walks `~/.claude/projects/` for JSONL transcripts and produces the metrics above. Run it against your logs:

```bash
uv run python scripts/analyze_sessions.py ~/.claude/projects/ --report
```

## Part 6: Roundtrip Consistency — Code ↔ Docs

If you have engineering docs that describe what code should do, and code that implements it, the two should be consistent. The eval pattern applies in both directions — and in a loop that checks they stay in sync.

### The roundtrip pipeline

```
engineering.md  ──→  generate code  ──→  score generated code against actual codebase
     ↑                                              ↓
     └── score against original doc  ←─  generate doc  ←─┘
```

Each arrow is a deterministic check:

- **Doc → Code**: does the generated code compile? do function signatures match the doc's API spec? are all documented interfaces present?
- **Code → Doc**: does the generated doc mention every public function? are parameter types and return types correct? do examples match the actual code behaviour?
- **Roundtrip**: doc → code → doc produces a document that scores above the consistency threshold against the original. If the roundtrip degrades, either the model lost information or the original doc was ambiguous.

### Golden dataset

Each engineering doc becomes a golden case with three sub-scores:

```python
ROUNDTRIP_DATASET = [
    {
        "id": "api-authentication",
        "doc_path": "docs/design/auth.md",
        "code_path": "src/auth/",
        "check_compile": True,
        "check_signatures": ["login()", "verify_token()", "refresh()"],
        "check_exports": ["authenticate", "AuthError"],
    },
]
```

The scorers are deterministic — no LLM judge needed for the structural layer:

- `code_compile_score(path)` — 1.0 if the generated code passes `mypy` or `ruff`, 0.0 otherwise
- `signature_match_score(generated, expected)` — 1.0 if all required function names appear with the right parameter count
- `doc_roundtrip_score(original, generated)` — 1.0 if the regenerated doc covers all sections of the original (checked with substring matching on section headers)

### CI gate

Same `eval-baseline.json` pattern. Record the roundtrip scores on your current codebase. After any change (refactor, new feature, model swap), re-run and `--gate`. If the roundtrip drops below baseline, either the code and docs have drifted apart or the model can no longer maintain consistency at the expected level.

This completes the loop: the eval pyramid guards against quality regression, session history measures productivity gains, and roundtrip consistency ensures code and docs stay in sync across both directions.

The open-source evaluation ecosystem has matured quickly, and a fairly clear division of labor has emerged. The pattern most engineering-led teams converge on: **one lightweight framework for CI/CD gating, paired with one platform for production tracing, human annotation, and regression dashboards.** Running just one or the other tends to leave a gap.

| Tool                             | Type                               | Best for                                                                            | Notes                                                                                                                                                                                           |
| -------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **[Promptfoo](../promptfoo/)**   | OSS CLI, YAML-driven               | Model/prompt comparison, red-teaming                                                | Strongest open-source red-teaming suite (500+ adversarial vectors); acquired into OpenAI's Frontier infrastructure in March 2026, OSS package stays MIT                                         |
| **[DeepEval](../deepeval/)**     | OSS, Python/pytest-native          | CI-gated evals with typed metrics                                                   | 50+ built-in metrics (G-Eval, hallucination, faithfulness, contextual recall); agent-specific metrics for tool correctness and task completion; synthetic dataset generation from your own docs |
| **RAGAS**                        | OSS, Python                        | RAG pipeline scoring specifically                                                   | Research-backed retrieval + generation metrics; narrower scope than DeepEval/Promptfoo by design                                                                                                |
| **[Braintrust](../braintrust/)** | Platform (OSS scorer lib + hosted) | Full lifecycle: tracing → eval → CI gates → dashboards                              | Dataset-first, model-agnostic; sandboxed custom Python scorers; strongest option when you need eval scores wired into actual release gating, not just local test runs                           |
| **LangSmith**                    | Platform                           | Annotation workflows, production monitoring for LangChain-based agents              | Tightest fit if you're already on LangChain/LangGraph                                                                                                                                           |
| **Arize Phoenix**                | OSS, OpenTelemetry-native          | Teams already instrumented with OTel who want tracing + eval in one open-core stack | Best portability story since it rides on standard OTel traces                                                                                                                                   |
| **OpenAI Evals**                 | OSS, registry-based                | Reproducible, benchmark-style evals                                                 | Closest analog to running a published academic benchmark against your own system                                                                                                                |
| **Langfuse**                     | OSS + managed cloud                | Self-hostable tracing/eval                                                          | Self-hosting adds infra overhead; eval depth is lighter than the full platforms above                                                                                                           |

Practical pairing that shows up repeatedly across current guidance: **DeepEval for CI-gated, pytest-style evals** + **Braintrust or LangSmith for production traceability, human annotation, and regression dashboards**. For a Python/`uv`-first, terminal-centric workflow, DeepEval is the lower-friction starting point since it drops straight into an existing pytest suite and CI pipeline without standing up a separate service.

## Part 8: A Working CI Harness

### Deterministic layer with pytest + DeepEval

```python
# test_skill_output.py
import json
from deepeval import assert_test
from deepeval.metrics import GEval, TaskCompletionMetric
from deepeval.test_case import LLMTestCase

def test_pdf_report_generates_valid_file(tmp_path):
    """Deterministic check — no LLM call needed."""
    result = run_skill("generate-compliance-report", output_dir=tmp_path)
    output_file = tmp_path / "report.pdf"

    assert output_file.exists()
    assert output_file.stat().st_size > 1024          # not an empty/broken PDF
    assert result.tool_calls_used <= 6                 # step-efficiency bound
    assert "delete" not in [c.tool_name for c in result.tool_calls]  # invariant

def test_summary_accuracy_llm_judge():
    """Model-graded layer — used only where deterministic checks can't reach."""
    test_case = LLMTestCase(
        input="Summarize the Q2 anomaly detection findings.",
        actual_output=run_skill("summarize-report").text,
        expected_output=REFERENCE_SUMMARY,
    )
    correctness = GEval(
        name="Correctness",
        criteria="Does the summary accurately reflect the reference without inventing figures?",
        evaluation_params=["input", "actual_output", "expected_output"],
        threshold=0.8,
    )
    assert_test(test_case, [correctness])
```

Run it exactly like any other suite: `uv run pytest test_skill_output.py -v`. Wire it into CI so any PR touching a skill definition or prompt re-runs the suite; a failed eval blocks the merge, same as a failed unit test.

### Sampling for non-determinism

```python
def run_with_sampling(task_fn, n=10, threshold=0.85):
    """Run N times, report pass-rate instead of a single pass/fail."""
    results = [task_fn() for _ in range(n)]
    pass_rate = sum(r.passed for r in results) / n
    return {
        "pass_rate": pass_rate,
        "gate_passed": pass_rate >= threshold,
        "variance": statistics.stdev([r.score for r in results]),
    }
```

### Regression gating against a baseline, not an arbitrary number

```yaml
# ci-eval-gate.yaml
gates:
  - name: tool-selection-accuracy
    baseline: 0.88 # measured from current production, not guessed
    acceptance: 0.95
    block_on: below_baseline # don't merge if this PR regresses the baseline
  - name: hallucination-rate
    baseline: 0.03
    acceptance: 0.02
    block_on: below_baseline
```

## Part 9: Common Pitfalls

- **Trusting a single run.** Non-determinism means one green test tells you almost nothing about reliability. Sample.
- **Using LLM-as-judge as your only signal.** It's a second stochastic system layered on the first. Anchor with deterministic checks wherever the check _can_ be deterministic.
- **Setting acceptance bars without a baseline.** An arbitrary "95% accuracy" target means nothing if you never measured what production is doing today.
- **Grading final output only, for agents.** A trajectory that hallucinates a tool call halfway through and then "recovers" to a correct-looking answer is not a healthy trajectory even if the final string passes. Score the path.
- **Not controlling judge bias.** Position bias and length bias are well-documented and easy to accidentally bake into a comparison-style judge prompt. Randomize ordering; don't reward verbosity by default.
- **Skipping human review entirely.** Automated evals catch what the rubric anticipated. They don't catch what it didn't.
- **Ignoring cost compounding in agents.** Per-step reliability multiplies across a trajectory — a system that looks fine on single-tool-call benchmarks can degrade sharply once chained into a 5–10 step agent.

## Part 10: Putting It Together

A reasonable target mix, consistent with what's converged across current practice: **roughly 60% deterministic checks, 30% model-graded (LLM-as-judge), 10% human-in-the-loop**, running on every PR that touches prompts, skills, or agent logic, gated against a measured baseline rather than an invented threshold.

The uncomfortable truth is that you can't fully eliminate the non-determinism — you're not trying to. You're building enough layered signal that when something does drift, you catch it before it ships, and when you can't automate the judgment call, you know exactly which 10% needs a human to actually look at it.

## Part 11 — Worked Example: Evaluating a Skill End-to-End

This section walks through a single concrete skill across all eval layers: given an OpenAPI 3 spec, scaffold a full Next.js API — App Router route handlers, TypeScript types, Zod validation schemas, and a typed fetch client.

The skill definition is at [`.github/skills/nextjs-api-from-spec/SKILL.md`](https://github.com/beautifularchitecture/baba-yaga/blob/main/.github/skills/nextjs-api-from-spec/SKILL.md) in the companion repo, with the eval code at [`tests/eval_skill.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/.github/skills/nextjs-api-from-spec/tests/eval_skill.py). Read the SKILL.md first — every claim in it maps to an eval check below.

### The skill contract (extracted from SKILL.md)

The skill promises to produce code with these properties:

| Claim                                        | Where it's stated in SKILL.md                    |
| -------------------------------------------- | ------------------------------------------------ |
| Valid TypeScript (`tsc --noEmit`)            | Output section                                   |
| ESM `import`/`export` syntax                 | Output section                                   |
| Zod schemas matching TypeScript types        | Procedure step 3                                 |
| Route handler per path + operation           | Procedure step 4                                 |
| Typed fetch client functions per operationId | Procedure step 5                                 |
| JSDoc on every handler                       | Procedure step 4                                 |
| No `any`, prefer `unknown`                   | Output section                                   |
| No eval / require / hardcoded secrets        | Output section                                   |
| All external input validated with Zod        | Output section                                   |
| Barrel exports (`*/index.ts`)                | Procedure step 6                                 |
| Reads `baseUrl` from env var or `/api`       | Procedure step 5                                 |
| Error class with status + body               | Example output                                   |
| No `process.env` in handlers                 | (invariant — implied by "config module" pattern) |
| All responses use `NextResponse.json()`      | (invariant — App Router convention)              |
| Files within allowed directories only        | (invariant — structural constraint)              |

### The pyramid

Not all checks are equal. The pyramid shows the _shape_ of the eval suite — wide, cheap base tapering to narrow, expensive tip:

```
L6          X        Human review (spot-check samples)
L5         XXX       Statistical sampling (N runs at temp > 0)
L4        XXXXX      Golden dataset (curated specs, regression gate)
L2       XXXXXXX     Model-graded (semantic judgment, per-PR only)
L1+L3   XXXXXXXXX    Deterministic + invariants (15 checks, every commit)
```

Volume ↑ · Cost ↑ · Frequency ↓

Each layer up has fewer X's (fewer checks), costs more per check, and runs less frequently. The pile-up at the base catches the 80% that can be expressed as code assertions. The tip catches what slips through — but you need that tip to be narrow, because expensive checks at scale become noise.

The golden dataset (L4) isn't a separate check type — it's _the same L1–L2 checks_ run against curated inputs with a regression gate. Sampling (L5) repeats the same suite N times. Human review (L6) compares automated scores against a domain expert. L3 (property-based invariants) doesn't appear as a separate layer because invariants are just more deterministic checks — they belong in L1. The six layers in Part 2 remain valid as a taxonomy, but in practice you organize them by _shape_: three check types (L1, L2, and everything above is process around them).

### Layer 1 — Deterministic + invariants (15 checks, $0)

Every claim that can be expressed as a code assertion, including invariants that must hold regardless of input:

| #   | Check                        | What it verifies                                 | Implementation in eval_skill.py                                                       |
| --- | ---------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------- |
| 1   | `tsc --noEmit`               | Generated directory is valid TypeScript          | `check_tsc_no_emit()` — writes files to temp dir, runs `npx tsc --noEmit`             |
| 2   | ESM imports                  | No `require()` anywhere                          | `check_esm_imports()` — string search                                                 |
| 3   | Route file per path          | Each OpenAPI path → `app/api/.../route.ts`       | `check_route_file_per_path()` — parse spec, glob generated files                      |
| 4   | Export matches method        | `GET` endpoint → `export async function GET`     | `check_named_export_matches_method()` — regex per route file                          |
| 5   | JSDoc on handlers            | Every exported function has `/** ... */`         | `check_jsdoc_on_handlers()` — regex for JSDoc preceding each export                   |
| 6   | Zod schema per type          | For each schema in spec, a `{Name}Schema` exists | `check_zod_schema_per_type()` — extract schema names from spec, grep generated code   |
| 7   | No `any`                     | Uses `unknown` instead of `any`                  | `check_no_any()` — regex with false-positive guards                                   |
| 8   | No banned patterns           | No `eval(`, `require(`, hardcoded secrets        | `check_no_eval_require_secrets()` — regex for secret-like strings                     |
| 9   | Input validation             | Route handlers call `.parse()` or `.safeParse()` | `check_all_input_validated()` — check every route file uses Zod parsing               |
| 10  | Barrel exports               | `types/index.ts`, `schemas/index.ts`             | `check_barrel_exports()` — file existence check                                       |
| 11  | Client per operation         | For every `operationId`, a typed client function | `check_client_function_per_operation()` — extract operationIds, find matching exports |
| 12  | Client error handling        | Throws `ApiError` on non-2xx                     | `check_client_throws_api_error()` — search for `throw new ApiError`                   |
| 13  | Base URL from env            | Client reads `NEXT_PUBLIC_API_URL`               | `check_base_url_from_env()` — string search                                           |
| 14  | No `process.env` in handlers | Config not scattered across route files          | `check_no_process_env_in_handlers()` — grep route files for env access                |
| 15  | `NextResponse.json()`        | No bare `new Response()` in handlers             | `check_wraps_in_next_response()` — grep for disallowed patterns                       |

**15 checks**, all deterministic, all zero-cost, all noise-free. This is the wide base — code asserts that run on every commit.

Invariants (#14, #15) used to be a separate "Layer 3" in earlier drafts. They aren't a different thing — they're just as deterministic as the other 13. Calling them a separate layer overcomplicates the picture without adding insight. Put them in L1.

### Layer 2 — Model-graded (3 criteria, per-PR)

Three things code checks can't verify:

1. **Schema correctness.** A `z.string()` on a field that should be `z.number()` passes check #6 (schema exists), but it's semantically wrong. Only an LLM judge can spot this.
2. **Handler logic.** A `GET /pets/{petId}` handler that ignores `petId` passes #3 (file exists), #4 (export matches), #14 (no env), but returns the wrong data.
3. **Client ergonomics.** A generated client that requires manual URL construction passes #11 (function exists) but defeats the purpose of typed clients.

```python
JUDGE_PROMPT = """Score the generated Next.js API code 0.0–1.0 on:
- schema_correctness: do Zod types match the actual OpenAPI schema semantics?
- handler_logic: does each route handler correctly use its params and body?
- client_ergonomics: is the fetch client pleasant to use with typed params?

OpenAPI spec title: {spec_title}
Generated files: {file_listing}"""
```

Run this only on PRs that change the skill prompt — not on every commit. At ~$0.01 per judge call, the cost doesn't matter on its own, but the _noise_ does — you don't want a stochastic judge gate-keeping every diff. Reserve it for intentional prompt changes.

### Layer 4 — Golden dataset (regression gate)

The golden dataset is the same L1 + L2 checks, but run against curated inputs with a baseline and gate.

Four specs in `tests/`:

| Case             | Why it's there                                                                                           |
| ---------------- | -------------------------------------------------------------------------------------------------------- |
| `pet-store.yaml` | Flat CRUD, 6 operations, nested schemas, multipart upload, enums, nullables — covers the happy path      |
| `ecommerce.yaml` | 9 operations, pagination, allOf composition, deeply nested schemas, 20+ query params — stresses handling |
| `minimal.yaml`   | Single endpoint, no named schemas, no params — edge case the skill should handle gracefully              |
| `malformed.yaml` | Invalid YAML — skill should reject with an error rather than hallucinate code                            |

Run with:

```bash
# Record baseline (run against all specs, save scores)
uv run python .github/skills/nextjs-api-from-spec/tests/eval_skill.py --baseline

# After a skill prompt change, gate against baseline
uv run python .github/skills/nextjs-api-from-spec/tests/eval_skill.py --gate
```

A regression detection:

```
  PASS    pet-store.yaml          0.987  (was 1.000, Δ -0.013)
  REGRESS ecommerce.yaml          0.857  (was 0.943, Δ -0.086)
```

The gate blocks the merge. The baseline (`eval-baseline.json`) is checked into the repo so every developer uses the same reference.

### Layer 5 — Statistical sampling

A single run tells you nothing. Run the suite multiple times and look at the distribution:

```bash
uv run python .github/skills/nextjs-api-from-spec/tests/eval_skill.py \
  --runs 10 --temperature 0.2
```

Output:

```
  Spec                        Overall   Stdev  Pass-rate    N
  pet-store.yaml                0.987   0.012      1.000   10
  ecommerce.yaml                0.871   0.184      0.800   10
  minimal.yaml                  1.000   0.000      1.000   10
  malformed.yaml                1.000   0.000      1.000   10
```

Gate on pass rate, not mean score. A single bad run pulling the mean down is noise; a pass rate below 80% across 10 runs is a real problem. The `ecommerce` case with stdev=0.184 is fragile — that's where you investigate.

### Layer 6 — Human review

The automated suite catches what the rubric anticipated. It does not catch what it didn't. Sample generated outputs periodically and export for human review:

```bash
uv run python .github/skills/nextjs-api-from-spec/tests/eval_skill.py --export-human
```

This produces `human-review.csv` with columns: `spec, l1_overall, l2_overall, overall, human_score, human_note`. Fill in `human_score` manually, then compare with the automated score. Disagreements above 0.3 reveal rubric gaps — "this passes all code checks but uses the old `pages/api` pattern," or "Zod schema is correct but uses `z.string().email()` while the spec says `format: uri`."

Refine the L1 checks or L2 judge prompt to close the gap, then re-run. The goal isn't to eliminate disagreements — some judgments genuinely need a human — but to shrink the gap over time as you encode the patterns your reviewers keep finding.

### What this demonstrates

The pyramid isn't about which layer number a check belongs to. It's about the _shape_ of signal: a wide, cheap, deterministic base that catches 80% of failures on every commit, a narrow semantic judge for the PRs that matter, and process layers (dataset, sampling, humans) that don't add new check types but make the existing ones regression-trackable and statistically sound.

For this skill:

- **15 checks in L1** — deterministic, $0, every commit
- **3 criteria in L2** — semantic, ~$0.01, per-PR only
- **L4-L6 are process**, not check types — they apply L1+L2 across curated inputs, multiple runs, and human review

The same decomposition works for any skill: read the SKILL.md, extract every claim, classify each as "can I write a code assertion for this?" (L1) or "do I need a judgment call?" (L2), then wrap them in a golden dataset with baselines, sampling, and human spot-checks. The source code at `tests/eval_skill.py` implements all four layers in a single file you can adapt to your own skill.

## Open Questions

- **Coverage for skills and agents.** Code coverage answers "which lines of code did my tests exercise?" There is no equivalent for skills — no automated way to say "my golden dataset covers 80% of my agents' trigger conditions and procedure steps." How would you define skill coverage, and can it be computed without manual auditing?
