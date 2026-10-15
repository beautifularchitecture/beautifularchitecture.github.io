+++
date = '2026-10-14T12:05:00+10:00'
draft = false
title = 'AI-DLC Harness Engineering: What You Can Change and What You Edit to Change It'
tags = ['AI-DLC', 'AI', 'Agentic', 'Software Engineering', 'Methodology', 'Harness Engineering', 'AWS']
summary = 'AI-DLC harness engineering is the list of changes you can make to the methodology by editing Markdown and YAML data, each mapped to the exact file you edit in core/.'
+++

## Introduction

[AI-DLC](https://github.com/awslabs/aidlc-workflows) ships a working methodology out of the box. Harness engineering is changing that methodology for your team: which stages run, which agents exist, which checks fire and which standing rules apply.

Two facts define the discipline:

- Every harness-engineering change is a data edit, not a code edit. You edit Markdown with YAML frontmatter. You never edit TypeScript.
- Every data edit happens in `core/`, the hand-authored harness-neutral source. The per-harness trees are generated from `core/` and are never edited.

The authoritative source is the [Harness Engineer Guide](https://github.com/awslabs/aidlc-workflows/blob/main/docs/harness-engineering/00-overview.md).

## The moment a change means editing code

This is the boundary. Use it to decide which guide you are in.

- If the change is a **data edit** — a Markdown file with YAML frontmatter, or a JSON config — it is harness engineering. This guide covers it.
- If the change is a **code edit** — any `.ts` file — it is the [Developer Reference](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/00-overview.md). Stop.
- The moment a change to the framework's behavior requires editing any of these, you have crossed the line into code:
  - The orchestrator's routing or state machine (`core/tools/aidlc-orchestrate.ts`).
  - A hook or a CLI tool.
  - The stage-graph compile pipeline.
  - The audit event taxonomy.
  - The test suite.
- The moment the graph compiler needs a frontmatter key it does not yet understand, or a new traversal rule, you have crossed into code.
- Adding a stage or an agent touches the workflow graph but does not change the code that reads the graph. That is why it is harness work.
- Changing how the graph is compiled, or adding a new audit event, is code. That is why it is not.

## The moment a change means editing `core/` and recompiling

Editing the source is only half the loop. The other half is regeneration.

- The moment you edit a file in `core/`, the per-harness trees are stale until you run `bun scripts/package.ts`.
- The moment you edit a stage's YAML frontmatter, `stage-graph.json` is stale until the graph is recompiled.
- In a source build, recompile with `bun scripts/package.ts`. In an installed project tree, recompile with `aidlc engine graph compile`.
- Commit the authored `core/` or `harness/` edit only. Never commit the generated roots.
- Never hand-edit `dist/<harness>/` trees. They are generated outputs, ignored and never committed.
- Never hand-edit `stage-graph.json`. It is a build artifact. Edit the YAML and recompile.

## Possible changes, in point form

Each point names the change and the exact file you edit to make it. All paths are relative to the repository root for framework source, except where noted.

### Stage changes

- **Change what a stage does** → edit the `## Steps` prose in `core/aidlc-common/stages/<phase>/<slug>.md`.
- **Change a stage's dependencies or ordering** → edit `requires_stage` in the stage frontmatter.
- **Change what a stage reads** → edit `consumes` in the stage frontmatter, per artifact, with `required` and `conditional_on`.
- **Change what a stage writes** → edit `produces` in the stage frontmatter. These are the forward edges other stages discover.
- **Change which agent leads a stage** → edit `lead_agent` in the stage frontmatter.
- **Change a stage's supporting agents** → edit `support_agents` in the stage frontmatter.
- **Change a stage's execution topology** → edit `mode` in the stage frontmatter. Values are `inline`, `subagent`, `pipeline` and `mob`.
- **Make a stage repeat once per Unit** → add `for_each: unit-of-work` to the stage frontmatter.
- **Change a stage's independent review** → edit the `reviewer` / `review_artifact` pair in the stage frontmatter.
- **Change a stage's sensor hookups** → edit the `sensors:` list in the stage frontmatter, or the `## Sensors` compartment.
- **Change a stage's scope membership** → edit the `scopes:` list in the stage frontmatter.
- **Add a brand-new stage** → create `core/aidlc-common/stages/<phase>/<slug>.md`, wire its edges, set its `scopes:`, then compile.

### Agent changes

- **Add a new persona** → create `core/agents/<slug>-agent.md` with frontmatter and a body.
- **Change the tools an agent may use** → add or edit the `tools:` allowlist in the agent frontmatter. Omit it to inherit the full session toolset.
- **Deny a specific tool** → edit `disallowedTools`. It must include `Task` on every agent.
- **Change an agent's model or reasoning effort** → edit `tier` (`judgment`, `balanced`, `templated`). Never author raw `model:` or `effort:`.
- **Customize what a shipped agent knows** → do not edit the shipped file. Add team knowledge instead (see Knowledge changes).

### Scope changes

- **Define a new scope** → create `core/scopes/aidlc-<name>.md` and add that scope name to each participating stage's `scopes:` tag.
- **Change a scope's default depth** → edit `depth` in the scope frontmatter.
- **Change a scope's test volume** → edit `testStrategy` in the scope frontmatter.
- **Change a scope's review ceiling** → edit `review_cap` in the scope frontmatter.
- **Change how far guards stand aside** → edit `guard_policy` (`strict`, `relaxed` or `off`) in the scope frontmatter.
- **Toggle a ceremony** → edit `sensors`, `learnings`, `summary_confirmation`, `plan_approval` or `collaborators` in the scope frontmatter.
- **Override a ceremony for one run** → pass the intent flag, such as `/aidlc --sensors on|off`.
- **Kill a ceremony globally** → set the matching kill switch, such as `AIDLC_DISABLE_SENSORS=1`.

### Rule changes

- **Teach a standing rule** → edit `core/memory/team.md` or `core/memory/project.md`.
- **Let a correction become a rule** → the learning loop promotes it, gated on `learnings` being enabled.

### Sensor changes

- **Add a deterministic check** → create a manifest under `core/sensors/` and add its id to a stage's `sensors:` list.
- **Change when a sensor fires** → edit `fire_on` in the sensor frontmatter (`write` runs on matching writes, `gate` runs once per deliverable at the approval gate).
- **Change whether a sensor can block a gate** → edit the binding severity (`advisory` records and opens the gate, `blocking` requires a verified pass). Blocking applies only to `fire_on: gate`.
- **Change what a sensor matches** → edit its `matches:` glob.

### Knowledge changes

- **Add team domain knowledge** → add files under `aidlc/knowledge/<agent>-agent/` at the space level, at runtime. This is the one change that is not in `core/`, and the framework never overwrites it.

### Construction and swarm changes

- **Set the team's Construction autonomy posture** → edit the rule layer under `core/memory/`.
- **Shape what parallel Bolts may run** → edit the `units-generation` stage.

### Harness changes

- **Add a new CLI harness** → add one `harness/<name>/` directory and one manifest row. No `core/` edits.
- **Add a per-harness doctor check** → add a `--doctor` arm in that harness directory.
- **Wire a harness tool binding** → edit your tool's subsections in the protocol modules.

### Plugin changes

- **Package a reusable optional change** → create a plugin in `plugins/<name>/` with a manifest at `.aidlc-plugin/plugin.json`.
- **Contribute new primitives from a plugin** → add subtrees for stages, agents, scopes, sensors, knowledge or tools.
- **Modify an existing core stage from a plugin** → add an additive contribution file under the plugin's `contributions/` directory. A plugin never edits `core/`.

## What the tooling enforces for you

- Stage filename stems must equal the frontmatter `slug`. `aidlc-graph compile` rejects stem mismatches, duplicate stage slugs and multiple producers for any consumed artifact.
- Sensor filename and id checks are compile-time hard errors.
- Scope and agent duplicate declared names are loader errors that name both files.
- Scope and agent filename-to-name drift is an advisory from `/aidlc --doctor`.

## References

- [AI-DLC repository (awslabs/aidlc-workflows)](https://github.com/awslabs/aidlc-workflows)
- [Harness Engineer Guide](https://github.com/awslabs/aidlc-workflows/blob/main/docs/harness-engineering/00-overview.md)
- [Developer Reference](https://github.com/awslabs/aidlc-workflows/blob/main/docs/reference/00-overview.md)
- [Published documentation](https://awslabs.github.io/aidlc-workflows/)
