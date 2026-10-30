+++
date = '2026-07-05T10:00:25+10:00'
draft = false
title = 'ToolCallCheck: Mock Your Tool Server and Assert on Trajectories'
tags = ['ToolCallCheck', 'Evals', 'LLM', 'Testing', 'Python', 'Agents', 'MCP']
summary = "Mock MCP server that runs fully offline, records every tool call, and lets you assert on tool selection, argument correctness, call order, and structural invariants — no LLM call needed."
+++

## What ToolCallCheck Does

Most agent failures are not in the final output — they are in the tool calls. The model says the right thing but calls the wrong tool, or calls the right tool with hallucinated arguments, or blunders through 11 steps when 3 would do. These failures are invisible to output-only evals and expensive to catch with an LLM judge.

ToolCallCheck is a mock MCP server that sits in place of your real tool server during tests. It records every tool call the agent makes, and after the run you assert on the recorded trajectory — which tools, what arguments, in what order, and whether structural invariants held. No model call, no network, fully deterministic.

It fills Layers 1 and 3 of the [eval pyramid](../../ai-outputs-eval/):

- **Layer 1 (Deterministic)** — "was `read_file` called with the right path?" and "did the agent call tools in the expected order?"
- **Layer 3 (Invariant)** — "was `process_refund` called more than once?" and "did the agent ever call a destructive tool like `delete`?"

## The Core Pattern

```python
server = MockMCPServer()
server.responses["lookup_order"] = {"success": True, "order": {"id": "ord_123"}}
server.responses["process_refund"] = {"success": True, "refund_id": "rf_456"}

agent = create_agent(tools=server)
agent.run("Refund order 123")

server.assert_called_once_with("lookup_order", order_id="ord_123")
server.assert_called_once_with("process_refund", order_id="ord_123")
server.assert_at_most_once("process_refund")   # invariant
server.assert_called_in_order(["lookup_order", "process_refund"])
```

No model call. No network. The test either passes or fails based on what the agent actually _did_, not what the output _said_.

## Assertion Reference

| Assertion                               | Layer | What it checks                                                 |
| --------------------------------------- | ----- | -------------------------------------------------------------- |
| `assert_called(name)`                   | 1     | Tool was called at least once                                  |
| `assert_called(name, times=N)`          | 1     | Tool was called exactly N times                                |
| `assert_not_called(name)`               | 1     | Tool was never called                                          |
| `assert_called_once_with(name, **args)` | 1     | Tool called once with matching args                            |
| `assert_called_in_order([...])`         | 1     | Tools called in expected sequence                              |
| `assert_step_count(lo, hi)`             | 1     | Total tool calls within range                                  |
| `assert_no_destructive_calls()`         | 3     | No destructive tools (delete, destroy, deploy, drop, shutdown) |
| `assert_at_most_once(name)`             | 3     | Tool called 0 or 1 times (idempotency invariant)               |

## Invariants Catch the Expensive Failures

The most common agent bugs are not "wrong answer" — they are "called the wrong tool" or "called the right tool twice." These are cheap to catch with a mock server and expensive to catch in production.

**Refund twice** — an agent that calls `process_refund` twice for the same order because it retried without checking idempotency. ToolCallCheck catches it with `assert_at_most_once("process_refund")`.

**Destructive action** — an agent given a "read config" task that decides to call `delete` or `deploy`. Caught with `assert_no_destructive_calls()`.

**Wrong argument** — an agent calls `write_file` with the right path but the wrong content, or calls `send_email` without a recipient. Caught with `assert_called_once_with("write_file", path="output.py")`.

**Wandering trajectory** — an agent takes 12 steps to do a 4-step task. Caught with `assert_step_count(3, 6)`.

## Example: Refund Flow

```python
def test_refund_flow():
    server = MockMCPServer()
    server.responses["lookup_order"] = {"success": True}
    server.responses["process_refund"] = {"success": True}
    server.responses["send_notification"] = {"success": True}

    result = run_agent(server)

    assert "Refund completed" in result
    server.assert_called_once_with("lookup_order", order_id="ord_123")
    server.assert_called_once_with("process_refund", order_id="ord_123")
    server.assert_at_most_once("process_refund")
    server.assert_called_in_order([
        "lookup_order", "process_refund", "send_notification",
    ])
```

If the agent calls `process_refund` twice (maybe a retry loop without checking if the first call succeeded), `assert_at_most_once` catches it. If it calls `send_notification` before `process_refund` updates the database, `assert_called_in_order` catches it.

## Example: Safety Invariant

```python
def test_no_destructive_actions():
    server = MockMCPServer()
    server.responses["read_file"] = {"success": True, "content": "key: val"}

    run_agent(server)  # agent decides to delete the file

    server.assert_no_destructive_calls()  # FAILS — delete was called
```

A `PreToolUse` hook would block the call at runtime. ToolCallCheck catches it at test time — you know the agent _tried_ to call delete even if the guard hook blocked the actual side effect.

## Integrating With pytest

ToolCallCheck is pytest-native. Every assertion raises `AssertionError`, so tests compose naturally:

```python
import pytest
from example_toolcallcheck import MockMCPServer, scenario_refund_flow

def test_refund_idempotent():
    server = MockMCPServer()
    server.responses["lookup_order"] = {"success": True}
    server.responses["process_refund"] = {"success": True}
    server.responses["send_notification"] = {"success": True}

    scenario_refund_flow(server)

    server.assert_at_most_once("process_refund")
    server.assert_called_in_order(["lookup_order", "process_refund"])
    server.assert_step_count(2, 4)
```

Run it the same way as any other test:

```bash
pytest tests/test_trajectory.py -v
```

## Companion Repo

The companion repo includes [`scripts/tools/example_toolcallcheck.py`](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/tools/example_toolcallcheck.py) with a complete `MockMCPServer` implementation, scenarios (refund flow, destructive guard, search-then-write), and test functions demonstrating every assertion.

## ToolCallCheck vs. Trajectory Eval

Both check tool-call trajectories, but at different levels:

| Dimension            | ToolCallCheck                                           | Trajectory Eval                                      |
| -------------------- | ------------------------------------------------------- | ---------------------------------------------------- |
| Runs offline         | ✅ Yes — mocks the tool server                          | ✅ Yes — scores pre-recorded trajectories            |
| Checks specific args | ✅ `assert_called_once_with(name, **args)`              | ✅ `score_argument_correctness(traj, expected_args)` |
| Checks call order    | ✅ `assert_called_in_order([...])`                      | ✅ `score_tool_order(traj, expected_order)`          |
| Enforces invariants  | ✅ `assert_at_most_once`, `assert_no_destructive_calls` | ❌ Manual only                                       |
| Mocks the server     | ✅ Intercepts live agent runs                           | ❌ Requires pre-recorded data                        |
| Scoring granularity  | Binary pass/fail per assertion                          | Numeric 0.0–1.0 per criterion                        |

Use ToolCallCheck for unit tests of individual tool-call behaviours. Use Trajectory Eval for full regression scoring of agent trajectories.

## Further Reading

- [Testing LLM Outputs: The Eval Pyramid](../../ai-outputs-eval/)
- [Trajectory Evaluation in the Companion Repo](https://github.com/beautifularchitecture/baba-yaga/blob/main/scripts/trajectory/trajectory_eval.py)
- [pytest for LLM Evaluation](../pytest/)
- [hypothesis: Property-Based Testing for LLM Outputs](../hypothesis/)
