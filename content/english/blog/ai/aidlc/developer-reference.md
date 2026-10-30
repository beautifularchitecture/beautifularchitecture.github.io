+++
date = '2026-10-14T12:10:00+10:00'
draft = false
title = 'AI-DLC Developer Reference: Changing the Framework Itself'
tags = ['AI-DLC', 'AI', 'Agentic', 'Software Engineering', 'Methodology', 'Developer Reference', 'AWS', 'Orchestration']
summary = 'A complete guide to the code behind AI-DLC: the 5-layer architecture, the plane split, the orchestrator and its state machine, the stage protocol and definition, the agent, knowledge, rule and sensor systems, hooks and CLI tools, the compile pipeline and runtime graph, the engine and skill system, plugins, supply-chain security, commit provenance, the test suite, and how to contribute.'
+++

# AI-DLC Developer Reference: Changing the Framework Itself

> Part of the AI-DLC series. [User Guide](/docs/guide/) → build with AI-DLC. [Harness Engineer Guide]({{< relref "blog/ai/aidlc/harness-engineering.md" >}}) → reshape how AI-DLC behaves through configuration. **This guide** → change AI-DLC's code: the orchestrator, hooks, CLI tools, the compile pipeline, and the test suite.

AI-DLC is one hand-authored, harness-neutral core that ships to many harnesses (Claude Code, Kiro CLI, Kiro IDE, Codex CLI, Cursor, opencode, and GitHub Copilot). The [Harness Engineer Guide]({{< relref "blog/ai/aidlc/harness-engineering.md" >}}) shows how to reshape it with data — stages, agents, scopes, rules, sensors, knowledge. This guide is the layer beneath that: the **code** that reads that data, and how to change it.

Where the harness engineer edits Markdown with YAML frontmatter and JSON, the developer edits TypeScript — the orchestrator, the hooks, the CLI tools, the stage-graph compile pipeline, the audit taxonomy, and the test suite. This post folds the whole Developer Reference into one narrative, chapter by chapter, so the boundary between a data edit and a code change is always in view.

## How to Read This Reference

This reference documents the internal architecture and implementation of AI-DLC. It is meant for contributors changing the AI-DLC codebase itself: the orchestrator, hooks, CLI tools, the stage-graph compile pipeline, the audit taxonomy, and the test suite.

If you are **using** AI-DLC to build software, the [User Guide](/docs/guide/) is your starting point. If you are **reshaping how AI-DLC behaves** through configuration — adding a stage or an agent, defining a scope, authoring rules and sensors, adding team knowledge — the [Harness Engineer Guide]({{< relref "blog/ai/aidlc/harness-engineering.md" >}}) is the place to begin; those are data changes, not code changes, and its chapters narrate the workflow before pointing back here for the exhaustive schema.

### The four path conventions

AI-DLC is authored once and generated per harness, so a file is named by one of four conventions, by intent:

- **`core/…`** — the hand-authored, harness-neutral **source of truth** (for example `core/tools/aidlc-orchestrate.ts`, `core/aidlc-common/stages/`). This is where you edit. When a path names where a file is *authored* or *changed*, it is a `core/` path.
- **`dist/<harness>/...`** — an **ignored local generated** source/development and manual-copy projection (`dist/claude/.claude/`, `dist/kiro/.kiro/`, `dist/kiro-ide/.kiro/`, `dist/codex/`, `dist/cursor/`, `dist/opencode/`, `dist/copilot/`). Never hand-edited or committed; materialized by `bun scripts/package.ts`. Releases carry it under `runtime/<harness>/` inside the Bun-shaped `aidlc-copy-runtime-X.Y.Z.tar.gz`.
- **`dist-release/<harness>/…`** — the corresponding ignored native projection. Releases carry it under `runtime/<harness>/` inside `aidlc-runtime-X.Y.Z.tar.gz`, which the native installers and lifecycle commands consume.
- **`<harness-dir>/...`** (for example `.claude/`, `.kiro/`, `.codex/`) — the **runtime** location inside an *installed* project, where commands run and the framework reads and writes during a workflow (`aidlc engine graph compile`, `loadAgents()` reading `.claude/agents/`). The directory is a parameter of the harness.

Where this reference shows a bare `.claude/` path, read it as the runtime location for the Claude harness specifically; the same file is authored in `core/` and ships to each harness's own directory.

### What this reference covers

The chapters follow the reference's own order:

| Section | Chapter |
|---|---|
| §2 | Architecture: the 5-layer model, the configuration-layers routing principle, the execution model, design decisions |
| §3 | Plane Architecture: the control / data / management plane separation and its boundaries |
| §4 | Orchestrator: the `SKILL.md` conductor, the forwarding loop, the gate ritual, the state machine it drives |
| §5 | Stage Protocol: the behavioral contract, approval gates, the compliance checklist |
| §6 | Stages: per-phase stage documentation |
| §7 | Agent System: agent structure, frontmatter contract, the configuration matrix |
| §8 | Hooks and Tools: the hook system, CLI tools, the 119-event audit taxonomy |
| §9 | Sensor System: the sensor manifest schema, PULL imports, the fire model, default severity |
| §10 | Rule System: rule file layout, scope derivation, the layer-chain resolver, conflict gates |
| §11 | Testing: the test pyramid, tiers, stubs, fixtures, the test registry |
| §12 | Knowledge System: the two-tier architecture, the DocumentKB derived catalog, loading order, templates |
| §13 | Contributing: the development workflow, the utility handler checklist, the documentation policy |
| §14 | State Machine: the workflow / phase / stage machines, the 119-event taxonomy, audit-first rules |
| §15 | Runtime Graph: the compiled `runtime-graph.json`, the data-plane mirror of the stage graph |
| §16 | Harness Primitives Mapping: how each AI-DLC concept maps to a harness's native primitives |
| §17 | Stage Definition: the YAML frontmatter contract, the three-compartment body, the compile pipeline |
| §18 | Artifact Vocabulary: naming rules, collision policy, filesystem mapping, the live registry |
| §19 | Engine and Skill System: the orchestration engine (`next`/`report`/`park`), the typed directive contract, the conductor, plural skills, scope shape, the swarm referee |
| §20 | Plugin Mechanism: the manifest, install-time composition as a real host plugin, the additive contribution seam, multi-tenant guards |
| §21 | Supply-Chain Security: release attestations, SLSA provenance, checksums, workflow hardening, installer defenses, ownership, transport |
| §22 | Commit Provenance: content-derived attribution from commits to reviewed units, `aidlc attest resolve`, `SOURCE_COMMITTED` anchors |
| §23 | Diagrams: all Mermaid diagrams in one place |
| §24 | Harness reference: the Kiro IDE hook payload |
| §25 | Examples and research: the plugin fixture and design notes |

### How to navigate

- **Where does a new concern (a rule, a methodology fact, a knowledge fact) belong?** Read [§2 configuration layers](#22-configuration-layers) — the two-axis model (authorship × consumption) with boundary tests routes any new concern to the correct file.
- **Adding a new stage?** Read [§5 Stage Protocol](#5-the-stage-protocol), then the relevant phase file in [§6](#6-stages-by-phase), then [§13 Contributing](#13-contributing).
- **Changing the stage definition format?** Read [§17 Stage Definition](#17-the-stage-definition) before editing any stage `.md` file. The stage file format is data-driven; the runtime reads a compiled JSON.
- **Adding or renaming an artifact?** Read [§18 Artifact Vocabulary](#18-the-artifact-vocabulary) — the naming rules, the stability policy (rename/removal is major, addition is minor), and `aidlc engine graph artifacts` for the live list. The registry is derived from stage files, not written.
- **Adding a new scope?** Read [§13 Contributing](#13-contributing). Scopes are file-authored — a scope file plus a `scopes:` tag on each member stage — no TypeScript edits required.
- **Adding a new agent?** Read [§13 Contributing](#13-contributing) and [§7 Agent System](#7-the-agent-system). Agents are data-driven via their `.md` frontmatter — no TypeScript edits required.
- **Working on hooks?** Read [§8 Hooks and Tools](#8-hooks-and-tools) and [§11 Testing](#11-the-testing-system).
- **Changing the orchestrator?** Read [§4 Orchestrator](#4-the-orchestrator) and [§2 Architecture](#2-architecture). If you are adding or modifying audit events, start with [§14 State Machine](#14-the-state-machine) — the drift test catches you if you do not.

### Relationship to the User Guide

The [User Guide](/docs/guide/) explains **what** AI-DLC does and **how to use it**. This Developer Reference explains **how it works** and **how to change it**. Some topics appear in both:

| Topic | User Guide | Developer Reference |
|---|---|---|
| Agents | What they do, when they appear | Frontmatter contract, how to add/modify |
| Knowledge | How to add company standards | Loading order internals, template system |
| Hooks | What gets logged | Hook implementation, audit event taxonomy |
