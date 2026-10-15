+++
date = '2026-10-14T12:10:00+10:00'
draft = false
title = 'AI-DLC Developer Reference: Changing the Framework Itself'
tags = ['AI-DLC', 'AI', 'Agentic', 'Software Engineering', 'Methodology', 'Developer Reference', 'AWS', 'Orchestration']
summary = 'The AI-DLC Developer Reference covers the code behind the methodology — the orchestrator, hooks, CLI tools, the stage-graph compile pipeline, the audit taxonomy and the test suite — and the boundary where a data edit becomes a code change.'
+++

## Introduction

The [Harness Engineer Guide](https://github.com/awslabs/aidlc-workflows/blob/main/docs/harness-engineering/00-overview.md) changes AI-DLC by editing data: stages, agents, scopes, rules, sensors and knowledge. The [Developer Reference](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/00-overview.md) is the other half. It documents the code that reads that data, and it is written for contributors changing AI-DLC itself.

- **Audience:** framework contributors, not users and not harness engineers.
- **Scope:** the orchestrator, hooks, CLI tools, the stage-graph compile pipeline, the audit taxonomy and the test suite.
- **Repository:** [github.com/awslabs/aidlc-workflows](https://github.com/awslabs/aidlc-workflows)
- **Published documentation:** [awslabs.github.io/aidlc-workflows](https://awslabs.github.io/aidlc-workflows/)

## The boundary: when a change is code

The split between the two guides is data versus code. Use these points to decide which side a change is on.

- **Data change** — a Markdown file with YAML frontmatter, or a JSON config. Harness engineering. No TypeScript.
- **Code change** — any `.ts` file. Developer Reference.
- A change is code the moment it touches any of these:
  - The orchestrator's routing or state machine (`core/tools/aidlc-orchestrate.ts`).
  - A hook.
  - A CLI tool.
  - The stage-graph compile pipeline.
  - The audit event taxonomy.
  - The test suite.
- Adding a stage or an agent touches the workflow graph but does not change the code that reads the graph. That is harness work.
- Changing how the graph is compiled, or adding a new audit event, changes the code that reads the data. That is developer work.

## Path conventions

The reference names files by one of four conventions, and the convention tells you whether you may edit the file.

- **`core/…`** — the hand-authored, harness-neutral source of truth. This is where you edit. `core/tools/aidlc-orchestrate.ts` and `core/aidlc-common/stages/` are examples.
- **`dist/<harness>/…`** — an ignored local generated copy for source development and manual testing. Materialized by `bun scripts/package.ts`. Never hand-edited and never committed.
- **`dist-release/<harness>/…`** — the generated native projection. Also ignored and never hand-edited.
- **`<harness-dir>/…`** (for example `.claude/`) — the runtime location inside an installed project, where commands run and the framework reads and writes during a workflow.

A bare `.claude/` path in the reference means the runtime location for the Claude harness specifically. The same file is authored in `core/` and shipped to each harness's own directory.

## The architecture at a glance

AI-DLC is a hybrid execution model: some stages run inline, where the conductor loads the agent persona and executes in conversation, and others delegate to subagents that run autonomously and return structured summaries.

Five layers carry the framework:

- **Rules** — the space memory layer, reached through the `rules/aidlc.md` import stub: the org, team, project and phase rule files, the Guard Policy section, and the guardrail sections. Confirmed human corrections are written back here.
- **Agents** — fourteen flat agent files: eleven domain-expert personas, two review-only agents and the composer. Authored core personas carry `disallowedTools: Task`.
- **Knowledge** — two tiers: `aidlc-shared/` for cross-agent reference including the canonical audit event registry, and `aidlc-<agent>-agent/` for per-agent methodology.
- **Skills** — the orchestrator entry point (`SKILL.md`, its `composer.md` annex and the protocol modules) plus the 33 stage files across the five phase directories.
- **Hooks** — the framework hooks for audit emission, session lifecycle, state sync, state validation, subagent tracking and statusline rendering. All are prefixed `aidlc-*.ts`.

## Configuration layers

Contributors placing a new concern (a rule, a methodology fact, a sensor binding) route it along two orthogonal axes:

- **Axis 1 — who authors it.** Framework-authored content ships with the distribution and is the same for every project. Team-authored content is specific to one project and persists across workflows.
- **Axis 2 — when it is consumed.** Continuously loaded harness configuration is read at session start and available to every stage. A per-workflow artifact is produced by a stage as output and read by later stages as input.

- Crossing the axes gives four quadrants. Three are populated: framework-authored config, team-authored config and team-authored per-workflow artifacts.
- The fourth quadrant, framework-authored per-workflow artifacts, is empty by design. Such outputs would have to ship with the distribution, which makes them harness config, not workflow output.
- Framework-authored config ships from upstream and is treated as immutable in your project. Add your practices to `team.md` or `project.md` instead of mutating framework defaults.

## Plane architecture

AI-DLC separates its concerns into three planes, borrowing the discipline from networking.

- **Management plane** — configuration in and observability out, at a human cadence.
- **Control plane** — the source files that decide how work is shaped: rule files and sensor manifests. Topology cadence.
- **Data plane** — the state lifecycle and forwarding, at the speed of a running workflow.
- The compile boundary joins the planes: authored source compiles into the artifacts the data plane reads.

## The engine and the conductor

The orchestration architecture splits one concern into two.

- The **engine** (`core/tools/aidlc-orchestrate.ts`, shipped to `<harness-dir>/tools/`) owns between-stage routing: scope resolution, the flag-precedence ladder, jump-direction computation, resume and init guards, stage sequencing, gate status and workflow completion.
- The **conductor** (`skills/aidlc/SKILL.md`) owns execution quality inside the move the engine named: framing the persona, asking good questions, keeping the stage diary, the intra-stage Keep/Modify/Redo loop and surfacing judgment to the human at gates.
- The engine is deterministic code by design. Routing is the determinism concern, so it lives in a tool, never in LLM prose.

The engine is a Bun CLI with exactly six subcommands:

| Subcommand   | Role                                                                                             | Mutates state? |
| ------------ | ------------------------------------------------------------------------------------------------ | -------------- |
| `next`       | Read workflow state and the compiled stage graph, resolve scope and position, emit one directive | No             |
| `report`     | Commit the transition after the conductor acted on a directive                                   | Yes            |
| `team-board` | Render the read-only Team Construction board                                                     | No             |
| `wait`       | Bounded, read-only wait for dispatched work                                                      | No             |
| `park`       | Pause an active workflow at a clean inter-stage boundary                                         | Yes            |
| `continue`   | Internal steering transport                                                                      | Yes            |

- Every directive is validated against the frozen contract in `aidlc-directive.ts` before it is printed.
- A malformed directive exits non-zero rather than emitting a lie the conductor would act on.
- `next` is idempotent: when the active-directive marker already records the exact directive for the state, it returns it verbatim and writes nothing, so re-asking never moves an approval.

## The typed directive contract

`aidlc-directive.ts` defines a discriminated union over eleven directive kinds, keyed on the `kind` field. Each directive carries exactly the fields its kind needs. The engine emits nine kinds today; two are documented placeholders that keep the loop complete-shaped.

| `kind`              | Emitted today?   | What the conductor does                                                      |
| ------------------- | ---------------- | ---------------------------------------------------------------------------- |
| `print`             | Yes              | Do what `directive.message` says: it is authoritative                        |
| `error`             | Yes              | Stop without recovering or retrying; the message is the user-facing error    |
| `done`              | Yes              | Either continue at once or present the completion summary and stop           |
| `parked`            | Yes              | Tell the user the workflow is parked, state how to resume, then stop         |
| `notice`            | Yes              | Print the message verbatim and stop                                          |
| `load-steering`     | Yes              | Apply the stage's rules in parts and continue the loop                       |
| `run-stage`         | Yes              | Run the named stage with its rules, context, ceremony and topology           |
| `ask`               | Yes              | Show the question and its choices, end the turn, then run the chosen command |
| `invoke-swarm`      | Yes              | Fan out the units and run the convergence loop against the swarm referee     |
| `dispatch-subagent` | No (placeholder) | Would run the named stage via a Task call rather than inline                 |
| `present-gate`      | No (placeholder) | Would run the gate ritual as its own directive                               |

## State machines and the audit taxonomy

Three nested state machines drive AI-DLC, and a fourth independent stream records session events. The four streams share the intent's audit trail but are owned by different code paths.

- **Workflow** — is the overall job running or done? Status values are `Running`, `Completed` and `Archived`.
- **Phase** — is this lifecycle phase in progress, verified or skipped because the scope excluded it?
- **Stage** — is the stage being worked on, waiting on the user, being revised after rejection or complete?
- **Session** — events emitted by the harness hooks.

Two rules connect the machines to the audit trail:

- **Every state transition has a tool-owned emitter.** A transition is recorded by its owning mutation path, never duplicated by the conductor.
- **Audit-first atomicity.** Tools emit their audit entries before mutating state. If audit emission fails, the tool throws before touching state, so the audit trail and the state file never disagree.
- The audit taxonomy records 118 event types across the workflow, phase, stage and session streams.
- Keeping the reference tables in sync with the code is enforced by the drift test at `tests/integration/t48-audit-event-emitters.test.ts`. If the document and the code disagree, the test fails.

## Hooks and tools

This implementation uses seventeen TypeScript hook sources. All seventeen are project-wide: one through the statusline key, the other sixteen through the `hooks` block of `settings.json`.

- **Eleven hooks are non-blocking.**
- **Six hooks are flow-altering:** the `Stop` hook keeps the forwarding loop running, the deliver-stage-rules hook attaches exact active-stage rules to subagent briefs, the plan-approval guard refuses premature code-generation dispatches, the reviewer-scope hook refuses sibling-unit reviewer access, the review-freeze hook refuses a reviewed-output write that would void a fresh terminal review receipt before the gate, and the state-transition guard refuses direct lifecycle calls that bypass `aidlc-orchestrate.ts report`.
- The hooks cover audit emission on Write and Edit, session lifecycle on SessionStart and SessionEnd, state sync on TaskUpdate, state validation on PreCompact, subagent tracking on SubagentStop, and statusline rendering.

## Testing

The AI-DLC test suite is entirely TypeScript. Every test is a `t*.test.ts` file run under `bun`, with zero shell test files. This is the platform-invariance guarantee by construction: the same files run identically on macOS, Linux and Windows.

The suite organises into four levels, one directory each under `tests/`:

- **`smoke`** and **`unit`** — the protocol layer: contracts, structure and cross-references.
- **`integration`** — the stage layer: individual stages with stub input, verifying artifacts.
- **`e2e`** — the acceptance layer: full workflows with artifact and experience verification.

The no-flag default and the `--ci` profile both run smoke, unit and integration. The `--release` and `--all` profiles add e2e.

## Supply-chain security

Releases are created by two isolated GitHub Actions workflows in the source repository: `release.yml` for stable tags and `preview-release.yml` for scheduled or manual previews. Both use the repository-provided `GITHUB_TOKEN`.

- Pushing a strict `vX.Y.Z` tag starts the release workflow.
- The first job rejects the release unless the event ref is the pushed tag, the checked-out commit is the tag target, the tag target is contained in `main` and the tag equals `v` plus the version in `core/tools/aidlc-version.ts`.
- Stable publication requires a passing release-purpose Full Suite for the exact tagged commit.

## Commit provenance

The commit provenance subsystem provides a durable reverse lookup from a git commit or diff range to the reviewed units of work that own its changed paths. It has three parts:

- **Committed reviewed-source evidence** — a snapshot of what a reviewer approved, written into the intent record so it travels with every clone.
- **The read-only resolver `aidlc attest resolve`** — attribution, drift classification and a stated trust basis.
- **The `SOURCE_COMMITTED` audit anchor** — enrichment only; the resolver never reads it.

Resolution answers two questions that are routinely conflated:

- **Integrity** — do the bytes that landed equal the bytes some receipt approved? Answered unconditionally, from content alone.
- **Authenticity** — was that receipt produced by a review that actually happened? Not answered by the record layout, because anyone who can write to the repository can write a receipt.

Every report carries a `trust{}` object naming the basis of its own attribution, on a four-rung ladder:

| `trust.level`   | What it means                                                              | What it rules out       |
| --------------- | -------------------------------------------------------------------------- | ----------------------- |
| `informational` | The record was read from the working tree — a local diagnostic             | Nothing                 |
| `reproducible`  | The record was read from a git tree, so the result reproduces in any clone | Non-determinism         |
| `independent`   | Reproducible, and the receipts did not come from the change under test     | Self-approval           |
| `signed`        | Independent, and authority-bearing inputs arrived in a signed commit       | Anonymous record writes |

Two verifier-supplied anchors move a report up the ladder:

- **`--record-ref <ref>`** reads receipts from the named ref's tree instead of the queried commit's, which strips a self-written approval.
- **`--require-trust <level>`** turns the level into a gate: exit 3 unless the achieved level reaches it.

Without either anchor, a report is informational metadata: accurate about integrity and silent about authority.

## Where things live

| Concern                  | Authored path                                                            |
| ------------------------ | ------------------------------------------------------------------------ |
| Orchestration engine     | `core/tools/aidlc-orchestrate.ts`                                        |
| Conductor skill          | `core/skills/aidlc/SKILL.md`                                             |
| Directive contract       | `aidlc-directive.ts`                                                     |
| Stage files              | `core/aidlc-common/stages/<phase>/<slug>.md`                             |
| Agent files              | `core/agents/<slug>-agent.md`                                            |
| Protocol modules         | `core/skills/aidlc/aidlc-common/protocols/`                              |
| Hooks                    | `core/hooks/aidlc-*.ts`                                                  |
| Test suite               | `tests/` (`smoke`, `unit`, `integration`, `e2e`)                         |
| Audit emitter drift test | `tests/integration/t48-audit-event-emitters.test.ts`                     |
| Release workflows        | `.github/workflows/release.yml`, `.github/workflows/preview-release.yml` |
| Version source           | `core/tools/aidlc-version.ts`                                            |

## Contributing

The [Contributing chapter](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/11-contributing.md) documents the development workflow, the utility handler checklist and the documentation policy. Two recipes that look like code but are data are covered there: adding a scope and adding an agent are both file-authored with no TypeScript edits.

## References

- [AI-DLC repository (awslabs/aidlc-workflows)](https://github.com/awslabs/aidlc-workflows)
- [Developer Reference overview](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/00-overview.md)
- [Architecture](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/01-architecture.md)
- [Engine and Skill System](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/17-skill-system.md)
- [State Machine](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/12-state-machine.md)
- [Hooks and Tools](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/06-hooks-and-tools.md)
- [Testing](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/09-testing.md)
- [Harness Engineer Guide](https://github.com/awslabs/aidlc-workflows/blob/main/docs/harness-engineering/00-overview.md)
