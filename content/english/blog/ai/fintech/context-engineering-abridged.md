+++
date = '2024-10-24T09:00:00+10:00'
draft = false
title = 'Context Engineering - Ideas Organised'
tags = ['Context Engineering','Prompt Engineering','Generative AI','RAG','AI Agents','Security','Governance','MCP', 'PDD', 'Abridged']
summary = "A practical guide to reliable, secure and auditable AI context."
+++

Context engineering is the design of systems that provide an LLM with **right-sized information**, **in the right format**, **at the right time**, to accomplish a task.

## Why it matters: bigger windows do not remove the problem

- Earlier years you had the problem of **smaller context size**, which was solved by models introducing **large context size**.
 
- It is tempting to think that a larger context window makes context management unnecessary. Capacity is different from attention. The evidence points the other way:

- **Position effects.** The **"Lost in the Middle"** more often lots of models use information near the start and end of a long input more reliably than information in the middle.
- **Context rot.** Even on simple tasks lots of models including the frontier models and found that performance becomes less reliable as input length grows.
- **Cost and latency.** Every token in the window is paid for and processed on every call (see *even with caching enabled*).

 The **goal** is the **smallest** set of **high-signal information** that lets the **next step succeed**. 

## Types of memories

Memories can be classified by **how long they last** and by **what kind of information they hold**. Each combination has different rules for writing, retrieval, correction and expiry.

### By duration

#### Short-term memory
- agent **working with right now**. 
- **conversation history**, **scratchpad**, **task state**. 
- **risk**: left unmanaged, it fills the window with stale assumptions and large tool outputs - **context distraction**.
- decisions you make during vibe coding are lost as it is part of the short term memory. 

#### Long-term memory
- not kept in the prompt. 
- persists after the task ends - written to a store and selected into context only when relevant. 
- **risks**: storing something wrong, keeping it too long or retrieving it in the wrong situation

### By content

Content based categorization can be divided broadly into 3 categories: 

#### Semantic memory: facts

Semantic memory holds facts about **an entity or a domain**, such as **preferences**, **attributes** and **stable details** (fact remains true across many tasks). It can be kept as:
- **single profile**: _updated over time_, _is easy to read_ but can grow out of control. Several facts grouped into one record. e.g. user's communication preferences.
- **collection of seperate records**: scales better but depends on retrieval quality, .
- **risk**: A fact that was true earlier may no longer be true, and the model has no way to know unless the **record carries a timestamp and source**. 
- **Semantic memories** (facts the agent learned from interactions) also overlap with **retrieved knowledge**(retrieved knowledge comes from a curated source of record). 

#### Episodic memory: past events
- stores **a previous task** and **the outcome**. 
- **risk**: retrieval mismatch.

#### Procedural memory: how to behave
- holds the **rules and instructions that govern behavior**. 
- such as the instructions applied based on glob patterns - so JPA standards applied to files matching an `entities/*.java` pattern. 
- defines **how** the agent behaves rather than **what** it knows.

## What goes into context

The system assembles Contexts from several sources, and each has a different lifespan, owner and level of trust. Knowing the source of the info matters when you later decide whether to retrieve, compress, isolate or discard it.

| Component | What it contains | Typical source | Lifespan |
| --- | --- | --- | --- |
| Instructions | Role, rules, tone, constraints, policies | Developer or organization | Stable across calls |
| User input | The current request and any supplied details | User | Current turn |
| Conversation history | Prior turns and responses | Runtime state | Grows until trimmed or summarized |
| Retrieved knowledge | Documents, records, search results | Retrieval layer | Per step |
| Tool definitions | Names, descriptions and parameter schemas | Orchestration layer | Per step or per workflow |
| Tool results | Output from searches, APIs, code and file reads | Tools | Per step, often large |
| Memory | Facts, past examples and learned preferences | External store | Across sessions |
| Examples | Sample inputs and outputs | Developer | Stable |
| Output specification | Schemas, templates, format requirements | Developer | Stable |
| Environment and state | Date, task progress, plans, permissions | Runtime | Changes during the task |
| Non-text inputs | Images, PDFs, audio transcripts | User or tools | Per step |

## Context mismanagement issues
These four failure modes:

### Context poisoning

- Context poisoning occurs when an **incorrect claim** or **observation** is saved in the **context and then reused**. 
- how does it happen? A **mistaken tool result** or a **model's unsupported conclusion** (claim without evidence) can become part of a **plan or summary**. 
- Later steps may **treat it as established information** and **repeat** or **build on the error**.
- A well-known example [Gemini 2.5 Pro playing Pokémon](https://arxiv.org/abs/2507.06261), Parts of the context became "poisoned" with misinformation about the game state. Because the agent kept re-reading its own notes, the false information was slow to undo, and the model could become fixated on goals that were impossible or irrelevant.
- solution is context needs a correction paths. should revise or remove stale claims when later evidence contradicts them **Context Pruning**.

### Context distraction

- Context distraction occurs when a **growing history pulls the model** toward **repeating past** actions instead of reasoning freshly about the current task. 
- **retain the parts needed to continue the task and condense or remove the rest**  aka Context Isolation.

### Context confusion

- Context confusion occurs when **irrelevant information affects the model's response**. 
- MCP is one example: **a model that receives many overlapping tools** has more opportunities to choose a tool that does not fit the request. 
- Retrieving only tools relevant to the current task is one proposed response - also called - **context selection**;

### Context clash

- Context clash occurs when **instructions or facts in the context disagree**. 
- Contradictions can arise between **old and new user statements**, **retrieved documents**, **tool outputs** or **intermediate model answers**. 
- Provenance, timestamps and explicit precedence rules help make conflicts visible. 
- Context shouldnt be silently merged or compressed (**summarization, compaction, trimming**) incompatible claims. 
- If conflict occurs - **resolve from trusted evidence** - **if not possible** the agent should **ask for clarification** or **report the uncertainty**. 

## Five implementation strategies

These five strategies describe ways to implement those decisions in an agent workflow, including a financial services workflow where relevance, traceability and access boundaries matter.

| Strategy | How it applies | Related operation |
| --- | --- | --- |
| Retrieve and trim | Retrieve records relevant to the current task, then remove passages, fields or messages the next step does not need. | Select and compress |
| Structured bundling | Assemble related inputs into a consistent structure, such as separate fields for the request, retrieved evidence, account state and required output. | Select and isolate |
| Breadcrumbs | Carry forward a compact task trail with key decisions, evidence references and pending actions so later steps can resume without replaying the full history. | Write and compress |
| Tool-gated context | Expose a tool's description and relevant data only when the workflow permits that tool for the current step. Enforce permissions in the tool or application as well as in the prompt. | Select and isolate |
| Policy-wrapped context | Apply standing instructions that govern how the model uses the task context and produces an output. Instructions can cover security, compliance and governance, as well as implementation conventions scoped to files or tasks, such as JPA standards for files matching an `entities/*.java` pattern. | Select and isolate |

These strategies can be combined. For example, a workflow can retrieve and trim transaction evidence, bundle the selected evidence with the request, record a breadcrumb that points to the source records, expose an approved tool for the current step and apply the instructions that govern the action and output. The workflow should preserve source references so reviewers can distinguish evidence from summaries or model-generated conclusions.

### Where this work fits in the larger roadmap

This work is part of larger 
- harness-neutral design
- staged scoped workflow
- knowledge as the single source of truth - fully reviwed and approved.
- full tracability and auditability
- persona based context and ai agents.
- unit level isolation (AI autonomous execution of stories)
- review and feedback
- specialist agents and approval gates

