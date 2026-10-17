+++
date = '2026-10-17T00:00:00+10:00'
draft = false
title = 'AI-DLC Harness Engineering: A Complete Guide to Shaping the Factory'
tags = ['AI-DLC', 'Agentic AI', 'Harness Engineering', 'Agents', 'Process']
summary = "A clear, practical guide to reshaping AI-DLC without writing code: stages, agents, scopes, rules, sensors, knowledge, swarm construction, porting harnesses, and authoring plugins."
aliases = ['/blog/ai/aidlc/harness-engineering/']
+++

# AI-DLC Harness Engineering: A Complete Guide to Shaping the Factory

> Part of the AI-DLC series. [User Guide](https://welldesignedsystem.github.io/docs/guide/) → build with AI-DLC. **This guide** → reshape how AI-DLC behaves for your team. [Developer Reference](https://welldesignedsystem.github.io/docs/reference/) → change AI-DLC's code.

AI-DLC is a spec-driven development factory that runs out of the box across Claude Code, Kiro CLI, Kiro IDE, Codex CLI, Cursor, opencode, and GitHub Copilot. It ships with 14 agents (11 domain experts, 2 reviewers, 1 composer), 33 stages, 11 scopes, plus rules and sensors.

This guide shows you how to **reshape AI-DLC for your team without writing code**. Change which stages run, add agents, tune scopes, teach standing rules, wire deterministic checks, add team knowledge, control Construction autonomy, port to a new harness, or author a plugin—all declaratively.

## 1. Three Roles, Three Guides

| Guide | Who | What You Change |
|---|---|---|
| [User Guide](https://welldesignedsystem.github.io/docs/guide/) | Building *with* AI-DLC | Nothing in harness configs. Run `/aidlc`, answer gates, review artifacts. |
| **Harness Engineering Guide (this one)** | Shaping *how* AI-DLC behaves | Declarative data: stages, agents, scopes, rules, sensors, knowledge. |
| [Developer Reference](https://welldesignedsystem.github.io/docs/reference/) | Changing AI-DLC *itself* | Code: orchestrator, hooks, CLI tools, graph compiler, tests. |

The rule is simple: **data vs code**. If you edit Markdown with YAML frontmatter or JSON, you're in this guide. If you edit TypeScript, you're in the Developer Reference.

## 2. Core Mental Model: What vs Who

Two primitives drive everything.

- **Stage** = *what* happens. A unit of work that declares consumed/produced artifacts, lead agent, and execution. Stages form the workflow graph.
- **Agent** = *who* does the work. A persona with domain expertise, tool allowlist, and model. Agents are loaded into stages.

Stages name their lead agent. Agents never name stages. This keeps work and workers decoupled—you can reassign by editing a stage, or add an agent without changing workflows until a stage opts in.

### 2.1 How Work Flows

- **Engine** (`core/tools/aidlc-orchestrate.ts`) — Deterministic. Reads `aidlc-state.md` and compiled `stage-graph.json`, picks next step, emits one typed directive. Subcommands: `next`, `continue` (internal), `report`, `park`, `team-board` (read-only), `wait` (bounded read-only).
- **Conductor** (`skills/aidlc/SKILL.md`) — Thin forwarding loop that executes each directive.

Everything else shapes their inputs:

- **Scopes** — Which stages run for a given work type.
- **Rules** — Standing "always do this" decisions applied to every workflow.
- **Sensors** — Deterministic checks bound to stages (advisory or blocking).
- **Knowledge** — Domain/project context agents load before working.

## 3. What You Can Change Without Code

Author in **`core/`** (harness-neutral source). Regenerate per-harness distributions.

| Change | Where | See |
|---|---|---|
| Edit a stage | `core/aidlc-common/stages/<phase>/<slug>.md` | §4 |
| Add a new stage | New file in phase + wire dependencies | §5 |
| Add/modify an agent | `core/agents/<name>-agent.md` | §6 |
| Define/tune a scope | `core/scopes/aidlc-<name>.md` + stage `scopes:` | §7 |
| Teach standing rules | `core/memory/{team,project}.md` (framework baseline) or space memory | §8 |
| Wire a deterministic check | `core/sensors/<id>.md` + stage `sensors:` | §9 |
| Add team domain knowledge | `aidlc/knowledge/<agent>-agent/` (space-level, runtime) | §10 |
| Shape Construction & swarm | `core/memory/` + `units-generation` stage | §11 |
| Port to a new harness | `harness/<name>/` + manifest/emit | §12 |
| Author a reusable plugin | `plugins/<name>/` | §13 |

**Team knowledge is the only exception**: it lives at `aidlc/knowledge/` in your project space and is never overwritten by framework updates.

### 3.1 Naming Rules & Validation

- Stage filename stem must equal frontmatter `slug`. `aidlc-graph compile` rejects mismatches, duplicate slugs, multiple producers of same consumed artifact (names files/consumer).
- Sensor filename/id consistency is compile-time enforced.
- Duplicate scope/agent names are loader errors naming both files. Filename↔name drift is reported by `/aidlc --doctor` as advisory.
- One owner per invariant (approval/transition/presence/plan fingerprint). Same check may appear at multiple boundaries, but duplicating logic is a defect.

### 3.2 Build Model: Author in `core/`, Regenerate

**Never edit `dist/` or `dist-release/`**. They are generated.

```bash
# 1. Edit source in core/ (never dist/)
$EDITOR core/aidlc-common/stages/inception/my-stage.md

# 2. Regenerate all harness projections
bun scripts/package.ts

# 3. Verify determinism (CI guard)
bun scripts/package.ts --check
```

Commit `core/`, `harness/`, `plugins/` only. `--check` builds twice in independent temp roots and byte-compares. When instructed to run `aidlc engine graph compile`, run it against your installed harness projection.

### 3.3 When to Cross Into Developer Reference

Cross over when changing code, not data:

- Orchestrator routing/state or engine/conductor/directive contract ([Orchestrator](https://welldesignedsystem.github.io/docs/reference/03-orchestrator/), [State Machine](https://welldesignedsystem.github.io/docs/reference/12-state-machine/), [Skill System](https://welldesignedsystem.github.io/docs/reference/17-skill-system/))
- Hooks/CLI tools ([Hooks and Tools](https://welldesignedsystem.github.io/docs/reference/06-hooks-and-tools/))
- Graph compile pipeline or audit taxonomy
- Test suite ([Testing](https://welldesignedsystem.github.io/docs/reference/09-testing/))

Adding stages/agents changes graph data—not the code that reads it. Changing *how* the graph compiles is code.

## 4. Anatomy of a Stage

A **stage** is the atomic unit of work. Every phase step is a single Markdown file.

### 4.1 Location

`core/aidlc-common/stages/<phase>/<slug>.md` with phases: `initialization`, `ideation`, `inception`, `construction`, `operation`.

### 4.2 Structure

Stage files have YAML frontmatter (machine-readable) and Markdown body (human-readable). Engine reads frontmatter; conductor/agents read body.

```yaml
---
title: "Stage Display Name"
slug: example-stage
phase: inception
lead_agent: aidlc-product-agent
scopes:
  - feature
  - bugfix
  - enterprise
consumes:
  - artifact.idea
produces:
  - artifact.spec
gate: approval  # or none
sensors:
  - sensor: aidlc-required-sections
    on: gate
    binding: advisory
---
```

Body typically has three compartments: **Purpose**, **Inputs & Outputs**, **Execution** (steps, acceptance criteria, guardrails).

### 4.3 Key Frontmatter

| Field | Purpose | Notes |
|---|---|---|
| `title` | Display name | Shown in dashboards/status |
| `slug` | Machine key | Must match filename stem; unique |
| `phase` | Lifecycle phase | One of the five phases |
| `lead_agent` | Persona that runs it | References `core/agents/<name>-agent.md` |
| `scopes` | Which scopes include it | Controls routing |
| `consumes` | Artifacts read | Forms dependency edges |
| `produces` | Artifacts written | Forms edges |
| `gate` | Approval behavior | `approval` or `none` |
| `sensors` | Bound checks | List with `sensor`, `on`, `binding` |

### 4.4 How It Connects

- **Graph edges** from `consumes`/`produces`. Compiler validates ordering, no duplicate producers of consumed artifacts.
- **Scopes** filter eligibility.
- **Sensors** run on `gate`/`write` with `advisory` or `blocking`.
- **Agents** execute via conductor with stage body instructions.

To change *what* happens, edit the stage file. No code change.

## 5. Adding a Stage

### 5.1 Plan First

1. **Phase** — Pick correct phase.
2. **Artifacts** — Define `consumes` (prereqs) and `produces` (new/updated). Use namespaced like `artifact.plan`, `plan.tasks`.
3. **Lead agent** — Pick existing or add one (§6).
4. **Scopes** — Add to relevant scope slugs.
5. **Gate/sensors** — Need human approval or deterministic checks?
6. **Avoid producer conflicts** — Only one stage produces a consumed artifact.

### 5.2 Author the File

Create `core/aidlc-common/stages/<phase>/<slug>.md`. Match stem to `slug`.

```yaml
---
title: "Risk Review"
slug: risk-review
phase: inception
lead_agent: aidlc-architect-agent
scopes:
  - feature
  - enterprise
consumes:
  - artifact.plan
produces:
  - artifact.risk-review
gate: approval
sensors:
  - sensor: aidlc-required-sections
    on: gate
    binding: advisory
---
```

Write clear body: purpose, inputs/outputs, execution steps, acceptance criteria, guardrails.

### 5.3 Wire & Validate

1. `bun scripts/package.ts` — regenerate
2. `aidlc engine graph compile` — validate slug match, edges, producer uniqueness, sensor refs
3. Verify stage appears in intended scopes (`/aidlc --doctor` or status)
4. Run relevant tests if needed

### 5.4 Tips

- Keep `produces` minimal (only what downstream needs).
- Use stable artifact names.
- Start sensors advisory; block only when unambiguously wrong.
- Write for the specialist agent with concrete acceptance criteria.

## 6. Adding an Agent

An **agent** is a persona: expertise, tool allowlist, model, behavior.

### 6.1 Location

`core/agents/<name>-agent.md`. Convention: `aidlc-<role>-agent.md`.

### 6.2 Structure

```yaml
---
name: aidlc-product-agent
title: "Product Agent"
description: "Translates user intent into clear requirements and product decisions"
tools: ["read", "write", "grep", "glob"]
model: inherit
---
```

Body: role, responsibilities, principles, inputs/outputs, guardrails, collaboration posture.

### 6.3 Key Fields

| Field | Purpose |
|---|---|
| `name` | Machine key; unique; align with filename stem |
| `title` | Display name |
| `description` | Short summary |
| `tools` | Tool allowlist (least privilege) |
| `model` | Usually `inherit` to follow harness defaults |

### 6.4 When to Add

- Need domain expertise not covered (data, security, legal, platform-specific)
- Distinct tool posture/decision style for certain stages
- Specialist reviewer with distinct scope

Avoid adding just to split steps—often a stage edit is cleaner.

### 6.5 Bind to Stages

Set `lead_agent: <your-agent-name>` in target stage(s). Then regenerate + compile.

### 6.6 Design Principles

- **Single domain focus** — One clear expertise.
- **Human judgement first** — Present options/tradeoffs/recommendations; don't guess human intent.
- **Least-privilege tools** — Only what genuinely needed.
- **Determinism boundaries** — Keep checks in sensors/hooks, not agent discretion.
- **Reusable** — Write to work across stages where sensible.

### 6.7 Validation

Filename stem must align with `name`. Duplicates are loader errors naming both. Referencing non-existent `lead_agent` fails graph validation.

## 7. Scopes

A **scope** decides which stages run for a work type. Each scope is a curated execute/skip view plus workflow defaults (depth, test strategy, review cap, ceremonies, guard posture).

### 7.1 Two-Part Definition

1. **Scope file** — `core/scopes/aidlc-<name>.md` defines identity, metadata, defaults.
2. **Stage membership** — Each stage's `scopes:` lists which scopes include it.

### 7.2 Scope File

```yaml
---
name: feature
depth: Standard
keywords: []
description: "Full lifecycle for new features at practical depth"
skeleton: on
runner: true
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---
# feature scope

Description...
```

### 7.3 Key Fields

| Field | Required | Purpose |
|---|---|---|
| `name` | Yes | Scope key |
| `depth` | Yes | `Minimal`, `Standard`, or `Comprehensive` |
| `testStrategy` | No | Overrides test volume; defaults to match depth |
| `review_cap` | No | `adversarial`, `advisory`, or `none`; can only lower |
| `keywords` | No | Hints for freeform `/aidlc <text>` |
| `description` | No | One-liner for `/aidlc --help` |
| `skeleton` | No | `on` opts into walking-skeleton when scope-dependent |
| `existing_code` | No | `true` if targets existing code (affects project detection) |
| `runner` | No | `true` includes in default scope-runner set |
| `freeform_default` | No | `true` nominates fallback; at most one enabled |
| `guard_policy` | No | `strict`, `relaxed`, or `off` (memory can override) |
| `sensors`/`learnings`/`summary_confirmation` | No | Ceremonies on/off (defaults on) |
| `plan_approval` | No | Require approval before code plan generation |
| `collaborators` | No | `off` runs lead-only (empty effective support agents) |

### 7.4 Membership on Stages

```yaml
scopes:
  - feature
  - bugfix
  - enterprise
```

If active scope is listed, stage executes.

### 7.5 Resolution Order

1. Global kill switches (`AIDLC_DISABLE_*`)
2. Valid per-intent flags (`/aidlc --sensors off`, etc.)
3. Memory layer (`org/team/project.md`) — can lock (e.g. `guard_policy: strict`)
4. Scope defaults
5. Built-ins

**Memory strict wins**: `Mode: strict` in memory overrides scope defaults and blocks chat/flag flips naming the source file.

### 7.6 Add or Tune

**Add:** create scope file → add scope name to stages that should run under it → regenerate → `aidlc engine graph compile` → verify.

**Tune:** change defaults in scope file or adjust stage `scopes:` lists. Commit in `core/`.

### 7.7 Validation

- Duplicate `name` across scope files → loader error naming both.
- >1 `freeform_default: true` among enabled → compile error naming claimants.
- Invalid enums → rejected with file/key + allowed values.
- Unknown scope refs in stages don't hard-fail but simply exclude those stages—compile/inspect to be sure.

## 8. Rules and the Learning Loop

**Rules** are standing "how" decisions that apply to every workflow. They encode conventions, constraints, preferences. **Knowledge** (§10) is "what" (context). **Sensors** (§9) are deterministic checks.

### 8.1 Where Rules Live

Runtime space memory:

```
aidlc/spaces/<space>/memory/
├── org.md      # Org/framework defaults
├── team.md     # Team affirmed practices
└── project.md  # Project overrides
```

Framework baseline patterns live in `core/memory/`. Phase guidance in `core/memory/phases/<phase>.md`. Space memory is team/project editable and takes precedence in practice (project > team > org on conflicts).

### 8.2 Authoring Rules

Write clear, actionable Markdown. Prefer concrete over vague.

```markdown
## Coding Standards
- Always use TypeScript strict mode
- Prefer named exports over default exports
- Never log secrets or keys

## Review Expectations
- All public APIs must have docstrings
- Breaking changes require explicit justification
```

### 8.3 Learning Loop: Corrections → Rules or Sensors

1. **Detect** — Human corrects/rejects.
2. **Capture** — What's wrong, why, correct form.
3. **Propose** — Promote to **rule** ("always do") or **sensor** (deterministic check).
4. **Confirm** — Human approves writing to memory or adding sensor.
5. **Apply** — Future runs see it.

This teaches AI-DLC your team's preferences over time without hardcoding.

### 8.4 Guard Policy in Rules

`## Guard Policy` with `Mode: strict|relaxed|off` in memory:

- `strict` — Wins over scope defaults; blocks chat/flag flips (names file).
- `relaxed`/`off` — Replaces scope defaults for that layer.
- One owner per rule; avoid unintentional duplication.

### 8.5 Editing Safely

- Space memory (`aidlc/spaces/.../memory/`) for team/project rules—no rebuild needed.
- Use `project.md` for one-off constraints, `team.md` cross-project.
- If checkable deterministically, consider a **sensor** instead of prose rule.
- Be explicit about "never" vs "avoid" and conflict resolution.

### 8.6 Effects

Space memory edits take effect next step. `core/memory/phases/*.md` is framework source—edit in `core/` and regenerate if shipping changes. Layer precedence: project > team > org.

## 9. Sensors

**Sensors** are deterministic checks bound to stages. They verify objective criteria and return **advisory** or **blocking** results. Run on matching writes or at approval gates.

### 9.1 Location

`core/sensors/<id>.md`

Shipped examples: `aidlc-required-sections`, `aidlc-claim-sources`, `aidlc-upstream-coverage`, `aidlc-traceability`, `aidlc-linter`, `aidlc-type-check`.

### 9.2 Manifest

```yaml
---
id: aidlc-required-sections
title: "Required Sections Check"
description: "Verifies required sections exist in key artifacts"
applies_to: ["artifact.spec", "artifact.plan"]
on: gate
binding: advisory  # or blocking
---
# aidlc-required-sections

## Check Logic
...

## Failure Message
...

## Remediation
...
```

### 9.3 Bind to Stages

```yaml
sensors:
  - sensor: aidlc-required-sections
    on: gate
    binding: advisory
  - sensor: aidlc-linter
    on: write
    binding: blocking
```

- `sensor` — references sensor `id`
- `on` — `gate`, `write`, or other supported events
- `binding` — `advisory` (warn, proceed) or `blocking` (must pass)

### 9.4 Add a New Sensor

1. **Design deterministically** — regex, file presence, counts, command output. No LLM.
2. **Create manifest** — `core/sensors/<id>.md` with frontmatter + logic/fail/remediation.
3. **Bind to stages** — Add to relevant stage(s) only.
4. **Regenerate + compile** — `bun scripts/package.ts` → `aidlc engine graph compile`.
5. **Test** — Verify advisory vs blocking behavior.

### 9.5 Guidelines

- **Deterministic only** — If needs semantic judgment, use rule/gate.
- **Clear failures** — Name exact issue + how to fix (file/section/command).
- **Least blocking** — Start advisory; promote to blocking only if correctness non-negotiable.
- **Scoped binding** — Only where `applies_to` artifacts appear.
- **Idempotent** — Safe to run multiple times.
- **One owner** — Don't duplicate same invariant.

### 9.6 Validation

Filename/id consistency compile-time hard error. Unknown sensor ref in stage fails validation. Invalid `binding`/`on` rejected.

## 10. Team Knowledge

**Knowledge** is domain/project "what" context agents load before working. Lives at **space level** (`aidlc/knowledge/`) in your project—never overwritten by framework.

### 10.1 Location

```text
aidlc/knowledge/
└── <agent>-agent/
    ├── README.md
    ├── domain.md
    ├── conventions.md
    └── ... (your docs)
```

Each dir named after agent (e.g. `aidlc-product-agent/`, `aidlc-architect-agent/`, `aidlc-developer-agent/`). Cross-agent shared material under `aidlc-shared/` where referenced.

### 10.2 What to Put

- **Domain facts** — Business rules, terminology, entities, workflows
- **Project context** — Architecture, modules, data models, integration points
- **Constraints** — Compliance, SLAs, security boundaries, third-party limits
- **Patterns/examples** — Preferred implementations, anti-patterns, real code snippets
- **Style guides** — API design, naming specific to this codebase
- **References** — Links to ADRs, runbooks, internal docs

Don't put "always do" in knowledge—put in memory rules. Knowledge is reference/context.

### 10.3 Authoring

1. Create per-agent dirs in project: `mkdir -p aidlc/knowledge/aidlc-developer-agent`
2. Write clear Markdown (agents read it). Use headings, tables, lists.
3. Be concrete—include file paths, module names from your codebase.
4. Organize logically (`api-design.md`, `database.md`, `testing.md`).
5. Commit to repo—it's team context.

Example:

```markdown
## Module Boundaries
- Core domain in `core/domain/`; adapters in `adapters/`
- Never import adapters from domain

## Error Handling
- Use Result types for expected failures
- Throw only for unrecoverable programmer errors

## Testing
- Unit tests co-located as `*.test.ts`
- Integration under `tests/integration/`
```

### 10.4 Usage

Lead agent (and collaborators if enabled) loads their knowledge dir before executing stage instructions. Changes take effect next stage run.

### 10.5 Knowledge vs Rules vs Stage Body

| Type | Location | Purpose | When | Editable |
|---|---|---|---|---|
| **Stage body** | `core/aidlc-common/stages/*/*.md` | Step-by-step for that stage | Stage runs | Harness engineer (core) |
| **Rules** | `aidlc/spaces/*/memory/{org,team,project}.md` | Standing "how" | Composed runtime | Team (space) |
| **Knowledge** | `aidlc/knowledge/<agent>-agent/` | Domain/project "what" | Agent loads pre-stage | Team/project (space) |

### 10.6 Best Practices

- Keep knowledge DRY—summarize and link to existing docs/ADRs.
- Focus on what agents need for *your* codebase.
- Update when architecture changes.
- Respect per-agent split—product doesn't need deep impl details; developer does.

## 11. Construction and the Swarm

Construction is where approved spec/plan becomes code. Control **autonomy posture**: how independently agents generate, whether to split into parallel Units (Bolts), and swarm behavior.

### 11.1 Autonomy Posture

Set in memory rules (`team.md`/`project.md`):

- **Guided** — Small sequential units, frequent checks
- **Pragmatic** — Balanced with gates
- **Autonomous swarm** — Split into independent Units, generate in parallel where safe, integrate later

### 11.2 Units, Bolts, Swarm

- **Unit** — A self-contained piece of planned work.
- **Bolt** — Implementation of a Unit.
- **Swarm** — Parallel execution of independent Bolts.

Flow: `units-generation` (or related) deconstructs plan → identify independent Units → run independent Bolts in parallel → integrate → verify (tests/lint/typecheck) → gates apply.

### 11.3 Configure Posture

Example memory:

```markdown
## Construction Posture
- Autonomy: pragmatic  # guided | pragmatic | autonomous
- Max parallel units: 3
- Prefer vertical slices over horizontal layers
- Each unit must be independently testable
```

Keys vary by team convention—keep declarative in memory.

### 11.4 Shape Parallelism via `units-generation`

Tune `core/aidlc-common/stages/construction/units-generation.md` body to define:

- How plan splits into Units
- Independence vs dependencies (serial where coupled)
- Acceptance criteria per Unit
- Max parallelism constraints

**Guided:** few larger Units, mostly sequential. **Autonomous:** small vertical slices, minimal cross-deps to maximize parallelism. **Constrained:** serialize where shared state/migrations couple.

### 11.5 Collaboration vs Lead-Only

`collaborators` setting (scope/intent) controls ensemble vs lead-only. `collaborators: off` → empty effective support agents; lead runs alone. Reviewers independent. Parallel Bolts follow same setting.

### 11.6 Guards, Sensors, Verification

Swarm respects:

- **Plan approval** — Code plans approved before generation unless explicitly off with proper recording
- **Sensors** — linter, type-check, traceability, required sections etc (can block)
- **Guard Policy** — How strictly guards re-check on changes
- **Gates** — Human judgement where `gate: approval` set

Autonomy shifts *when* humans judge, not *if*.

### 11.7 Best Practices

- **Design for independence** — Vertical Units with clear contracts.
- **Define per-Unit done** — Concrete acceptance (tests, artifacts).
- **Start conservative** — Pragmatic first; increase parallelism after sensors/verification prove reliable.
- **Traceability** — Link Units to plan/spec (use `aidlc-traceability`).
- **Integration discipline** — Swarm pieces must integrate cleanly; integration stage verifies whole system.

## 12. Porting to a New Harness

AI-DLC runs from single `core/`. To add a new harness, add thin surface in `harness/<name>/`—**no `core/` edits**. Packager emits `dist/<name>/`.

### 12.1 Required Files

In `harness/<name>/`:

- `manifest.ts` — Maps `core/` → harness layout; handles `{{HARNESS_DIR}}` token substitution
- Orchestrator skill: `skills/aidlc/SKILL.md`
- `emit.ts` — Per-shell emissions for shell-based harnesses (Codex/opencode/Copilot)
- Any other harness-specific files

Use existing harnesses as reference.

### 12.2 Manifest Contract

`manifest.ts` defines projection rules (file mappings, transforms, substitutions). Keep minimal/declarative. See `harness/claude/manifest.ts`, `harness/codex/manifest.ts`, `harness/opencode/manifest.ts`.

### 12.3 Emit Layer

For shell-emitting harnesses, `emit.ts` produces harness-native runtime from neutral core.

### 12.4 Workflow

1. **Create**: `mkdir -p harness/<newname>`
2. **Copy reference**: Start from closest similar harness
3. **Write `manifest.ts`**: Map `core/` → target; handle `{{HARNESS_DIR}}`
4. **Add skill + emit**: Add `skills/aidlc/SKILL.md` and `emit.ts` if needed
5. **Update packaging**: Ensure `scripts/package.ts` includes/handles this harness per convention
6. **Regenerate**: `bun scripts/package.ts` → produces `dist/<newname>/`
7. **Verify determinism**: `bun scripts/package.ts --check`
8. **Spot-check**: Inspect emitted files in `dist/<newname>/`
9. **Test**: Run relevant tests

### 12.5 Principles

- **No core edits** — Only `harness/<name>/` (+ packaging config if needed)
- **Thin surface** — Adaptation only; logic stays in core
- **Deterministic** — Byte-stable emissions (`--check` enforces)
- **Follow references** — Mirror existing harness patterns
- **Respect tokens** — Handle `{{HARNESS_DIR}}` correctly

See [Porting to a New Harness](https://welldesignedsystem.github.io/docs/harness-engineering/09-porting-to-a-new-harness/) for full details.

## 13. Authoring a Plugin

Plugins add reusable, optional extensions **without** changing core. Additive only—with all plugins disabled, install is bare core. Plugins live in `plugins/<name>/`, emitted per harness to `dist/plugins/<name>/<harness>/`, composed via `tools/data/harness.json` `plugins` filter.

### 13.1 Structure

```text
plugins/<name>/
├── .aidlc-plugin/
│   └── plugin.json        # Manifest
├── stages/                # New stages (core-shaped)
├── agents/                # New agents
├── scopes/                # New scopes
├── sensors/               # New sensors
├── contributions/         # Additive contributions to existing core stages
├── tools/                 # Plugin tools (if any)
├── hooks/                 # Plugin hooks (if any)
├── knowledge/             # Plugin knowledge
└── tests/                 # Plugin tests
```

### 13.2 Manifest (`plugin.json`)

```json
{
  "name": "test-pro",
  "version": "1.0.0",
  "description": "Reference test plugin",
  "provides": {
    "stages": ["plugins/test-pro/stages/*"],
    "agents": ["plugins/test-pro/agents/*"],
    "scopes": ["plugins/test-pro/scopes/*"],
    "sensors": ["plugins/test-pro/sensors/*"]
  },
  "contributes": {
    "stageAdditions": [
      "plugins/test-pro/contributions/stage-additions.json"
    ]
  }
}
```

Use `plugins/test-pro/` as canonical reference.

### 13.3 Additive Contributions

Plugins never override core. Extend existing core stages via `contributions/` (additive seam). Merged at compose time—core stays intact and plugins remain optional/reversible.

### 13.4 Workflow

1. **Create**: `mkdir -p plugins/<name>/.aidlc-plugin`
2. **Manifest**: Write `.aidlc-plugin/plugin.json`
3. **Add content**: Stages/agents/scopes/sensors in core-shaped layout
4. **Contribute**: Add `contributions/stage-additions.json` if extending core
5. **Tests**: Add `plugins/<name>/tests/`
6. **Package**: `bun scripts/package.ts` emits per-harness plugin trees
7. **Enable**: Add `<name>` to `tools/data/harness.json` `plugins` array
8. **Compose/verify**: New stages appear only when enabled
9. **Test**: Run `t188-plugin-compose.serial.test.ts` (mechanism) + plugin content tests

### 13.5 Principles

- **Additive only** — Never override core files
- **Optional by design** — All off = bare core
- **Declarative** — Everything in `plugin.json`
- **Core-shaped** — Mirror core structure for consistency
- **Re-enableable** — Installed files remain toggleable via harness.json
- **Tested** — Mechanism + content covered

Refs: [Plugin Mechanism](https://welldesignedsystem.github.io/docs/reference/18-plugin-mechanism/), [Authoring a Plugin](https://welldesignedsystem.github.io/docs/harness-engineering/10-authoring-a-plugin/), `plugins/test-pro/`.

## 14. Practical Tips & Next Steps

- **Edit source, never dist** — Work in `core/`, `harness/`, `plugins/`. Regenerate with `bun scripts/package.ts`; verify `--check`.
- **Compile early** — After stage/scope/agent/sensor changes run `aidlc engine graph compile` (catches slug mismatches, duplicate producers, unknown refs).
- **Start advisory** — New sensors `advisory` first; promote to `blocking` only when correctness is non-negotiable.
- **One owner per rule/check** — Don't duplicate invariants across sensors/hooks.
- **Choose the right bucket**: "Always do X" → **Rules** (memory). "Here's X context" → **Knowledge** (`aidlc/knowledge/`). "Verify X objectively" → **Sensor**.
- **Scope membership on stages** — Set `scopes:` in stage frontmatter; it's auditable per stage.
- **Doctor first** — `/aidlc --doctor` surfaces filename/name drift and common issues.
- **Test** — Use `bash tests/run-tests.sh --help` for appropriate levels (smoke/unit/integration/e2e).

**Next Step:** Start with [§4 Anatomy of a Stage](#4-anatomy-of-a-stage)—it's the foundation every other change builds on.