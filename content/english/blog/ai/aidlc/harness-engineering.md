## 10. Team Knowledge

**Knowledge** is the domain context agents load before they work. While rules encode "how" (standing decisions), knowledge encodes "what" (domain facts, project context, internal docs, constraints, terminology).

Team/domain knowledge is added at the **project space level** (runtime), not in `core/`. The framework never overwrites space-level knowledge, so your team's context persists across updates.

### 10.1 Where Knowledge Lives

At runtime in your project space:

```
aidlc/knowledge/
└── <agent>-agent/
    ├── README.md
    ├── domain.md
    ├── conventions.md
    └── ... (your docs)
```

Each subdirectory is named after an agent (e.g. `aidlc-product-agent/`, `aidlc-architect-agent/`, `aidlc-developer-agent/`, etc.). Agents load relevant knowledge from their own directory before executing a stage.

Also cross-agent shared material lives under `aidlc-shared/` in knowledge contexts (referenced in base knowledge layout).

### 10.2 What to Put in Knowledge

Good candidates:

- **Domain facts**: business rules, terminology, entities, workflows specific to your product
- **Project context**: architecture overview, key modules, data models, integration points
- **Constraints**: compliance rules, SLAs, security boundaries, third-party limitations
- **Patterns & examples**: preferred implementations, anti-patterns, reference code snippets (from your codebase)
- **Style guides**: API design, naming conventions specific to this codebase (beyond generic rules)
- **Contextual references**: links to internal docs, runbooks, ADRs

Don't duplicate rules here. If it's "always do X", put it in memory rules (`team.md`/`project.md`). If it's reference/context, put it in knowledge.

### 10.3 Authoring Knowledge

1. **Create directories per agent** — In your project: `mkdir -p aidlc/knowledge/aidlc-developer-agent`
2. **Write Markdown files** — Use clear headings, tables, lists. Keep focused and scannable (agents read this).
3. **Be concrete** — Include examples, file paths, module names from your actual codebase.
4. **Organize logically** — Split large topics (e.g. `api-design.md`, `database.md`, `testing.md`).
5. **Version with repo** — Knowledge under `aidlc/` in your project is project-owned and should be committed (it's part of your team's context).

Example `aidlc/knowledge/aidlc-developer-agent/conventions.md`:

```markdown
## Module Boundaries
- Core domain in `core/domain/`, adapters in `adapters/`
- Never import adapters from domain

## Error Handling
- Use Result types for expected failures
- Throw only for unrecoverable programmer errors

## Testing
- Unit tests co-located as `*.test.ts`
- Integration under `tests/integration/`
```

### 10.4 How Agents Use It

When a stage runs, the lead agent (and collaborators, if enabled) loads their knowledge directory before executing stage instructions. This gives them project-specific context without bloating stage files or rules.

Knowledge is additive and persistent. You can add/update it anytime; changes take effect on the next stage run for those agents.

### 10.5 Knowledge vs Rules vs Stage Body

| Type | Location | Purpose | When Read | Editable By |
|---|---|---|---|---|
| **Stage body** | `core/aidlc-common/stages/*/*.md` | Step-by-step execution for that stage | When stage runs | Harness engineer (in core) |
| **Rules** | `aidlc/spaces/*/memory/{org,team,project}.md` | Standing "how" decisions | Composed at runtime | Team (space memory) |
| **Knowledge** | `aidlc/knowledge/<agent>-agent/` | Domain/project "what" context | Agent loads before stage | Team/project (space) |

**Key point**: Knowledge lives at space level (`aidlc/knowledge/`) in your project. It's not part of `core/`, and the framework never overwrites it.

### 10.6 Best Practices

- Keep knowledge DRY—don't repeat what's in README or ADRs verbatim; summarize and link.
- Focus on what agents need to know to make correct decisions for *your* codebase.
- Update knowledge when architecture changes (it's living documentation for agents).
- Per-agent split is intentional—product agent doesn't need deep implementation details, developer agent does.

## 11. Construction and the Swarm

Construction is the phase where code is written to implement the approved spec and plan. AI-DLC gives teams control over **Construction autonomy posture**—how independently agents generate code, whether work is split into parallel units, and how the swarm behaves.

### 11.1 Construction Autonomy Posture

The autonomy posture is configured in the **rule layer** (memory). Teams choose how much autonomy Construction has:

- **Tightly guided**: Small, sequential units with frequent checks
- **Pragmatic**: Balanced autonomy with gates where needed
- **Autonomous swarm**: Split work into independent Units (Bolts), generate them in parallel where safe, then integrate

This is controlled via memory rules (team/project). The `units-generation` stage and related Construction stages read that posture to determine splitting strategy and parallelism.

### 11.2 Units, Bolts, and the Swarm

Construction often splits planned work into **Units**. Each Unit becomes a **Bolt** (a self-contained piece of work to implement). 

In an **autonomous swarm** posture:

1. **Plan deconstruction**: The `units-generation` stage (or related stage) breaks the approved plan into independent, testable Units.
2. **Parallel generation**: Independent Units can run in parallel (swarm) because they don't have blocking dependencies on each other.
3. **Integration**: Completed Bolts are integrated, verified (tests/lint/typecheck), and reconciled.
4. **Verification gates**: Sensors and stage gates still apply (linter, type-check, traceability, etc.)

The swarm maximizes parallelism while preserving determinism and human judgement at approval gates.

### 11.3 Configuring Construction Posture

Set posture in memory (`aidlc/spaces/<space>/memory/team.md` or `project.md`). Be explicit about autonomy level and constraints:

```markdown
## Construction Posture
- Autonomy: pragmatic  # or guided | autonomous
- Max parallel units: 3
- Prefer vertical slices over horizontal layers
- Each unit must be independently testable
```

The exact keys depend on team conventions; what's important is that posture is data (rules), not hardcoded. Changes take effect on next Construction run.

### 11.4 Shaping Parallelism via `units-generation`

The `units-generation` stage determines:
- How the plan is split into Units
- Which Units are independent (can run in parallel)
- Dependencies between Units (must run sequentially)
- Acceptance criteria per Unit

Tune this stage's body (in `core/aidlc-common/stages/construction/units-generation.md`) to match your team's swarm posture. For example:

- **Guided**: Favor few, larger Units, mostly sequential
- **Autonomous**: Favor small, vertical-slice Units with minimal cross-dependencies to maximize parallelism
- **Constrained domains**: Enforce serialization where shared state/DB migrations create coupling

### 11.5 Collaboration vs Lead-Only

The `collaborators` scope/intent setting controls whether stages run with support agents or lead-only (`collaborators: off` → empty effective support agents). This affects Construction topology (ensemble vs single lead). Reviewers remain independent.

When swarm runs, each parallel Bolt is executed by its assigned agent(s) according to that setting. Determinism is preserved by the engine's directive routing.

### 11.6 Guards, Sensors, and Verification

Construction still respects:
- **Plan approval** (`plan_approval`): Code plans approved before generation (unless explicitly off with proper recording)
- **Sensors**: Linter, type-check, traceability, required sections, etc. can be blocking at write/gate
- **Guard Policy**: Controls how strictly guards re-check on changes
- **Gates**: Human judgement points remain where `gate: approval` is set

Autonomy doesn't remove human judgement—it moves it to the right gates.

### 11.7 Best Practices

- **Design for independence**: Small vertical Units with clear contracts maximize safe parallelism.
- **Define acceptance per Unit**: Each Bolt needs concrete "done" criteria (tests, artifacts).
- **Start conservative**: Begin with pragmatic posture, increase parallelism only after sensors/verification prove reliable.
- **Traceability**: Ensure Units trace back to plan items/spec (use `aidlc-traceability` sensor).
- **Integration discipline**: Swarm produces pieces; integration stage must verify the whole still works (tests, typecheck, lint).

## 12. Porting to a New Harness

AI-DLC is designed to run on many harnesses from a single `core/`. To add support for a new CLI harness, you add a thin surface in `harness/<name>/`—**no edits to `core/` required**.

The build system (`bun scripts/package.ts`) reads the harness manifest/emit logic and materializes `dist/<harness>/` trees per harness. Each harness projection is generated, not hand-authored.

### 12.1 What You Need to Add

Add `harness/<name>/` with:

- `manifest.ts` — How to project `core/` into this harness's distribution layout (paths, file mappings, template substitutions like `{{HARNESS_DIR}}`)
- Orchestrator skill surface: `skills/aidlc/SKILL.md` (or as specified by manifest)
- Harness-specific files as needed
- `emit.ts` (for shell-based harnesses like Codex/opencode/Copilot) — Per-shell emissions/formatting

Harnesses like Claude/Kiro may differ slightly from shell-emitting harnesses. Follow existing harnesses as reference.

### 12.2 Manifest Contract

`manifest.ts` defines the projection rules: which `core/` files go where in `dist/<name>/`, token substitutions, and any harness-specific transforms. This is the contract the packager uses.

See existing manifests (e.g. `harness/claude/manifest.ts`, `harness/codex/manifest.ts`, `harness/opencode/manifest.ts`) as concrete examples. Keep it declarative and minimal.

### 12.3 The Emit Layer

For harnesses that emit shell/CLI artifacts (`codex/`, `opencode/`, `copilot/`), `emit.ts` controls per-shell emissions. It produces the harness-native runtime files from the neutral core. Claude/Kiro/others follow their respective patterns per manifest.

### 12.4 Porting Workflow

1. **Create harness dir**: `mkdir -p harness/<newname>`
2. **Copy closest reference**: Start from the harness most similar in CLI model (inspect `harness/*/manifest.ts`).
3. **Write `manifest.ts`**: Map `core/` → target layout. Handle `{{HARNESS_DIR}}` token substitution as used in core prose.
4. **Add skill + emit**: Add `skills/aidlc/SKILL.md` and `emit.ts` if needed for your harness type.
5. **Add harness entry**: Update packaging logic if the manifest registry expects it (check `scripts/package.ts` and existing harness list). The build discovers harnesses per convention.
6. **Regenerate**: `bun scripts/package.ts` — produces `dist/<newname>/`
7. **Verify**: `bun scripts/package.ts --check` (determinism). Also spot-check emitted files in `dist/<newname>/` match harness expectations.
8. **Test**: Run relevant tests (`tests/run-tests.sh`) focusing on packaging/graph if applicable.

### 12.5 Principles

- **No core edits**: Porting must not modify `core/`. Only `harness/<name>/`, possibly packaging config.
- **Thin surface**: Put only harness-specific adaptation in `harness/`. Keep logic in core.
- **Deterministic**: Emissions must be deterministic (byte-stable). `--check` enforces this.
- **Follow reference**: Use existing harnesses as the source of truth for manifest shape and emit patterns.
- **Token substitution**: Respect `{{HARNESS_DIR}}` in core prose—manifest handles substitution per harness.

Full details in the Developer Reference and [Porting to a New Harness](https://welldesignedsystem.github.io/docs/harness-engineering/09-porting-to-a-new-harness/) (original doc). This summary covers the practical workflow.

## 13. Authoring a Plugin

**AIDLC plugins** let you add reusable, optional extensions without changing what ships in core. Plugins are additive: they can add new stages/agents/scopes/sensors/doctor checks and contribute to existing core stages via an additive seam. With all plugins disabled, the install is bare core.

Plugins live under `plugins/<name>/` and are emitted as real host plugins per harness into `dist/plugins/<name>/{claude,codex,copilot,cursor,kiro,kiro-ide,opencode}/`. A compose hook merges plugins into an install. The enabled set is controlled via `tools/data/harness.json` `plugins` filter.

### 13.1 Plugin Structure

Each plugin follows core-shaped layout plus manifest:

```
plugins/<name>/
├── .aidlc-plugin/
│   └── plugin.json        # Plugin manifest (declarative)
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

The design is covered in Developer Reference [Plugin Mechanism](https://welldesignedsystem.github.io/docs/reference/18-plugin-mechanism/); authoring workflow here.

### 13.2 Plugin Manifest (`plugin.json`)

`.aidlc-plugin/plugin.json` declares identity, version, compatibility, and what the plugin provides:

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

Follow the reference fixture `plugins/test-pro/` for structure.

### 13.3 Additive Contributions

Plugins **never override** core. To extend existing core stages, use `contributions/` (additive seam). Contributions add content/metadata without replacing original stage body/fields. The compose mechanism merges contributions at installation time.

This preserves core integrity and keeps plugins optional/reversible.

### 13.4 Authoring Workflow

1. **Create plugin**: `mkdir -p plugins/<name>/.aidlc-plugin`
2. **Write manifest**: `.aidlc-plugin/plugin.json` with name/version/provides/contributes
3. **Add core-shaped content**: Add stages/agents/scopes/sensors under plugin dirs (mirror core structure)
4. **Add contributions**: If extending core stages, add `contributions/stage-additions.json` (or per schema)
5. **Add tests**: `plugins/<name>/tests/` for plugin content tests
6. **Package**: `bun scripts/package.ts` emits per-harness plugin trees under `dist/plugins/<name>/<harness>/`
7. **Enable in harness.json**: Edit `tools/data/harness.json` `plugins` array to include `<name>` (filter enabled graph/scope/runner surfaces)
8. **Compose/verify**: Install/compose as per your harness; verify new stages appear only when enabled
9. **Test**: Run integration tests (e.g. `t188-plugin-compose.serial.test.ts` guards mechanism; plugin tests validate content)

### 13.5 Key Principles

- **Additive only**: Plugins add; never override core files.
- **Optional by design**: All plugins off = bare core.
- **Declarative manifest**: Everything declared in `plugin.json`.
- **Core-shaped**: Mirror `core/` structure for stages/agents/scopes/sensors for consistency.
- **Re-enableable**: Installed files remain re-enableable via harness.json filter.
- **Tested**: Mechanism has integration guard; plugin content has its own tests.

Reference: `plugins/test-pro/` is the canonical fixture. See [Plugin Mechanism (Ref)](https://welldesignedsystem.github.io/docs/reference/18-plugin-mechanism/) and [Authoring a Plugin](https://welldesignedsystem.github.io/docs/harness-engineering/10-authoring-a-plugin/) for full schema/details.

## 14. Practical Tips & Next Steps

- **Edit source, never dist**: Always work in `core/`, `harness/`, or `plugins/`. Regenerate with `bun scripts/package.ts`, verify with `--check`.
- **Compile early**: After stage/scope/agent/sensor changes, run `aidlc engine graph compile` to catch errors fast (slug mismatches, duplicate producers, unknown refs).
- **Start advisory**: Make new sensors `advisory` first; promote to `blocking` only when correct behavior is unambiguous.
- **One owner per rule/check**: Avoid duplicating invariants across sensors/hooks.
- **Rules vs Knowledge vs Sensors**: 
  - "Always do X" (policy) → Rules (memory)
  - "Here's X context" (reference) → Knowledge (`aidlc/knowledge/`)
  - "Verify X objectively" (deterministic) → Sensors
- **Scope membership on stages**: Prefer setting `scopes:` in stage frontmatter; it's visible and easy to audit per-stage.
- **Test changes**: Use `bash tests/run-tests.sh --help` to run appropriate test levels (smoke/unit/integration/e2e).
- **Doctor first**: `/aidlc --doctor` surfaces filename/name drift and common issues.

### Next Step

Start with [Anatomy of a Stage](#4-anatomy-of-a-stage)—it's the foundation every other change builds on. From there, follow the sections in order as you reshape AI-DLC for your team.