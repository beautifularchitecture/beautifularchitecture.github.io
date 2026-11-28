+++
date = '2026-11-14T12:00:00+10:00'
draft = true
title = 'Where I Trust AI in Fintech Software Delivery'
tags = ['AI', 'Fintech', 'Software Delivery', 'Trust', 'Context Engineering', 'Evals', 'Agentic AI']
summary = "From vibe coding to governed delivery: where AI earns trust in fintech."
+++

<!-- Status: blocks 1 and 2 drafted. Remaining blocks are TODO. Set draft = false when complete. -->

An AI agent can return a working-looking change in minutes. That change still needs a source of truth, a test, a reviewer and a rollback path. Fast generation is not fast delivery.

The experiments behind this site began with the simplest posture: describe a task in conversation and accept what the model produces. That is vibe coding. It made output cheap. It also moved the cost of verification to the reviewer.

This post follows the changes to that posture in sequence. Each change fixed a failure that the previous posture exposed.

## Vibe coding and the prompt response

The first posture was conversational generation. We described a task, accepted the generated change and moved to the next task. Output became cheap. Correctness did not. Prompt engineering was our first response to that gap.

### The constraints we started with

Early tools gave us a small context window and a monthly usage allocation. The context window limited how much code and documentation the model could see in one request. The usage allocation limited how many attempts the team could afford in a month.

The constraint was inconvenient. It also produced a useful discipline. We learned to select the material that mattered, to state the task narrowly and to stop paying tokens for restatement. Token budgeting was our first engineering practice around AI.

### Context drift and loss

Vibe coding is conversational. Each follow-up question stays in the thread. The thread drifts away from the original goal, and the coding agent still receives the whole history as context.

Every irrelevant turn is paid for on every later call. Anthropic describes context as a finite attention budget: as tokens accumulate, recall and long-range reasoning lose precision ([Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). A long chat is not just noisy. It is expensive.

There are a few failure modes. **Context poisoning** happens when an error enters the context and later steps build on it. **Context distraction** happens when history pushes the agent to repeat past actions instead of reasoning about the current task. **Context confusion** happens when irrelevant tools or documents lead the model toward the wrong instruction. **Context clash** happens when contradictory information leaves the agent choosing between incompatible assumptions ([Weaviate, 2025](https://weaviate.io/blog/context-engineering)).

When the window fills, the harness compacts or prunes the history. Compaction summarises older turns and restarts the window with that summary. The choice of what to keep is a lossy decision. Anthropic warns that aggressive compaction removes subtle context whose importance becomes clear only later. Pruning removes passages judged unneeded. Compaction can preserve the shape of a conversation and drop the constraint that mattered.

These failures are not prompt problems. They are context problems. The conversational posture created a new class of failure, and fixing the instruction would not fix the context. The [context engineering notes](/blog/ai/fintech/context-engineering/) catalogue these failure modes in detail.

### Velocity, accuracy and review

Generated code and pull requests arrived at scale. Incorrect code and documents arrived with the working ones. Every wrong artifact needed rework, and every rework round consumed another portion of the monthly allocation.

We adopted two habits to break the loop. We shrank the request before sending it, and we inspected the output before asking for more. Across teams I watched, the same loop appeared: generation at scale followed by rework that consumed the allocation. Reducing rework became a way of reducing token spend.

Velocity did not improve accuracy. A generated change can compile and still implement the wrong rule. The reviewer had to compare the change with the requirement rather than with the model's explanation. Passing tests established that the change met the tests. The tests had to come from an approved specification. Generated documents carried the same risk: a readable summary is not an approved source of truth.

### The review bottleneck

People generated code and documents they had not read. The reviewer became the first person to check whether the artifact matched the task. That moved accountability from the producer to the reviewer.

The bottleneck grew with volume. In fintech, the reviewer also carries the audit consequence of a wrong control or a wrong customer message. The review bottleneck is the failure that made vibe coding stop scaling for us.

### What I saw across teams and demos

Prompt libraries, skills, custom instructions, agents, scripts and hooks appeared as the first response. Skills were built and distributed. Demos combined them into elaborate workflows.

These artifacts made the instruction repeatable. They became the first prompt libraries we treated as project assets.

### What prompting fixed

Prompt engineering gave the instruction a stable form. A prompt became a file in the repository. The team could review the file, diff the file and reuse it across sessions. Two people starting the same task no longer depended on two different conversational phrasings.

It also made the request smaller. We stated the task, the output shape and the constraints before calling the model. That discipline cut restatement and reduced the number of retries that consumed the monthly allocation.

Skills and custom instructions carried standing rules into each session. Agents, scripts and hooks turned some repeated steps into defined actions. Anthropic defines prompt engineering as methods for writing and organising LLM instructions. Our prompt libraries did exactly that ([Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).

### What prompting could not fix

A better instruction did not supply the facts. The prompt could say "follow the payment-limit policy". It could not tell the model which policy version was current, where the ADR lived or whether the repository had already changed. Prompt engineering writes the request. Context engineering decides what the model can see.

It did not enforce anything. A prompt can ask the agent to run tests. A harness can refuse to advance until the tests pass. The instruction carries no power to stop an action.

It did not control context growth. Skills, retrieved documents and long histories still accumulated in the window. Poisoning, distraction, confusion and clash remained possible. Compaction still removed material judged unneeded.

It did not create a source of truth. When an ADR changed, every prompt that quoted the old rule became wrong. More prompt files created more places for rules to drift apart. Weaviate states that prompt engineering alone cannot fix a disconnected model; prompting techniques work best with retrieved documents, user history and domain-specific data ([Weaviate, 2025](https://weaviate.io/blog/context-engineering)).

Those limits moved the work from the instruction to the material around it. That is context engineering.

### What we measured

> **TODO — needs your data.** Record time to first working version, review time, rework count and defects that escaped the first review. Mark every metric that was not measured as not measured.

| Metric                        | Baseline | AI-assisted | Evidence | Status       |
| ----------------------------- | -------- | ----------- | -------- | ------------ |
| Time to first working version | TODO     | TODO        | TODO     | TODO         |
| Review time                   | TODO     | TODO        | TODO     | TODO         |
| Rework count                  | TODO     | TODO        | TODO     | TODO         |
| Escaped defects               | TODO     | TODO        | TODO     | Not measured |

### Industry signal

Five external findings match what we saw.

Sarkar and Drosos published the first empirical study of vibe coding. They analysed over 8 hours of curated session video and found that developers cycled between prompting, rapid scanning, application testing and manual editing. Their conclusion matters here: vibe coding does not remove the need for programming expertise; it moves that expertise toward context management, rapid evaluation and decisions about when to take over from the model. Trust in the tool was dynamic and contextual, built through repeated verification rather than blanket acceptance ([Sarkar and Drosos, 2025](https://arxiv.org/abs/2506.23253)).

METR ran a randomised controlled trial with early-2025 AI tools. Experienced open-source developers worked in their own repositories, and the AI-allowed tasks took 19% longer, with a confidence interval from +2% to +39% ([METR, July 2025](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)). METR's [February 2026 update](https://metr.org/blog/2026-02-24-uplift-update/) reports later data with some evidence of speedup, but the authors do not treat that estimate as reliable. Developers declined to participate or to submit tasks because they did not want to work without AI, and the later study paid a lower hourly rate.

Veracode tested more than 100 large language models across Java, Python, C# and JavaScript. 45% of the generated code samples failed security tests and introduced OWASP Top 10 vulnerabilities. The report found that security performance stayed flat as models became better at producing functional code ([Veracode, 2025](https://www.veracode.com/blog/genai-code-security-report/)). A passing build did not establish a safe change.

The DORA 2024 report found that AI adoption increased individual productivity, flow and job satisfaction while reducing software delivery stability and throughput. The report names small batch sizes and robust testing as crucial fundamentals ([DORA 2024](https://dora.dev/research/2024/dora-report/)). The DORA 2025 report goes further: AI acts as an amplifier of the existing organisational system, and the returns come from that system rather than from the tool ([DORA 2025](https://dora.dev/research/2025/dora-report/)).

The Stack Overflow 2025 survey shows the same caution at individual level. 84% of respondents used or planned to use AI tools in development and 51% of professional developers used them daily. On accuracy, 46% actively distrusted the output against 33% who trusted it, and 3% reported high trust. 87% of respondents were concerned about the accuracy of AI agents and 81% were concerned about data security and privacy ([Stack Overflow Developer Survey 2025](https://survey.stackoverflow.co/2025/ai/)).

These studies measure different things: task time in a repository, generated code against security tests, survey sentiment and delivery outcomes. None of them measures whether a specific fintech team can trust a generated change. That decision needs evidence from the team's own system.

### Transition

The first response made the instruction repeatable. The failures that remained sat in the material around the instruction. That is where the next section starts.

## Context engineering: what to create and how to create it

Prompt engineering made the request repeatable. It did not decide what the model could see. Context engineering took over that decision: select a small set of trusted material for the next step, in a defined order, under a defined token budget.

### What the change fixed

Each failure from the conversational posture had a mechanism, not a better instruction.

- **Drift and irrelevant history.** The agent receives a bundle assembled for the current task, not the full chat. The bundle is ranked, deduplicated and budgeted. We took that shape from the [documentation graph SDD](/blog/ai/fintech/fintech-documentation-graph-sdd/).
- **Context rot and position effects.** Chroma tested 18 models and found that accuracy becomes unreliable as input length grows, even on simple tasks ([Chroma, 2025](https://research.trychroma.com/context-rot)). Liu et al. found that recall is highest when relevant information sits at the beginning or end of a long context and degrades in the middle ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)). We stopped loading whole repositories and loaded material only when a step needed it.
- **Attention budget.** Anthropic treats context as a finite attention budget with diminishing returns. Every new token depletes the budget ([Anthropic, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). A smaller bundle reduced token spend and reduced wrong answers at the same time.
- **Compaction loss.** Anthropic recommends structured note-taking outside the window and reading the notes back after a reset ([Anthropic, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). We kept decisions and constraints in files. A compaction event could discard chat history without discarding the rule.
- **Poisoning, confusion and clash.** Retrieved material carries a source, a status and an owner. Superseded documents do not enter the bundle. Standing rules outrank retrieved text. Weaviate names the four failure modes and states that better prompting alone cannot fix a disconnected model ([Weaviate, 2025](https://weaviate.io/blog/context-engineering)).

### Three kinds of context

We classify context into three kinds. The classification is ours. It is not an external standard.

- **Declarative context** states what to create: requirements, ADRs, domain rules and the current state of the repository.
- **Procedural context** states how to create it: standing rules, output templates, skills and workflow steps.
- **Evidence context** states what counts as done: tests, schemas, tool output and review findings.

Evidence context is the bridge to enforcement. A test result or a gate result is data that a program can check. An instruction is not.

### The fintech hierarchy

The fintech approach has two parts: a hierarchy for authority and a template for every artifact.

The AWS AI-DLC workspace supplies the hierarchy. `memory/` holds `org.md` for framework defaults, `team.md` for team practices, `project.md` for project practices and `phases/` for phase rules. `templates/` holds one output-format override per artifact ([AWS, Spaces and Intents](https://awslabs.github.io/aidlc-workflows/guide/03-spaces-and-intents/)). A more specific file overrides a general one. Conflicts are resolved by level before the model sees the material. The model never chooses between two versions of the same rule.

`knowledge/documents/` holds originals that we own. `knowledge/documentkb/` holds the derived catalog: `index.json` plus metadata and extracted content. The catalog is rebuildable, so hand-editing it is a mistake. `codekb/<repo>/` holds architecture, component inventory and a freshness marker. `intents/` holds the record of each piece of work and its audit trail ([AWS, Spaces and Intents](https://awslabs.github.io/aidlc-workflows/guide/03-spaces-and-intents/)). We never let the model rewrite an original to make a conflict disappear.

Template driven means each artifact has a fixed shape. In construction we extract a structured document from code through `code → AST → documentation` against a fixed Markdown template, then compare the filled template with the knowledge base produced in inception and elaboration ([Fintech AI-DLC approach](/blog/ai/fintech/fintech-aidlc-approach/)). The template gives retrieval a target. The agent knows which fields must be filled and which source fills each field.

Hierarchy and templates form the basis for knowledge in this approach. Every artifact has a level, an owner, a path and a shape. A statement without an owner is not a rule.

### Six retrieval buckets

The six buckets are our design, not an external standard. They extend the five usage buckets in the [documentation graph SDD](/blog/ai/fintech/fintech-documentation-graph-sdd/) by giving standing rules and evidence their own retrieval paths.

| #   | Bucket                               | Source of truth                                                            | Retrieval implementation                                                                                       |
| --- | ------------------------------------ | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 1   | Standing rules                       | `memory/org.md` → `team.md` → `project.md` → `phases/`                     | Always loaded. Resolved by specificity. Versioned in git. No similarity search.                                |
| 2   | Domain and regulatory documents      | `knowledge/documents/` originals and the derived `documentkb/`             | Key lookup by document id, persona and status. Full-text and vector seeds. Freshness filter on every result.   |
| 3   | Architecture decisions               | ADRs and non-functional requirements                                       | Relationship navigation from the feature document to linked ADRs. `SUPERSEDED` entries excluded.               |
| 4   | Requirements and acceptance criteria | Feature documents, units and output templates                              | Key lookup by `featureKey`. Template-constrained extraction. Filled template compared with the knowledge base. |
| 5   | Code reality                         | `codekb/<repo>/` and AST-derived documentation                             | Key lookup by repository and component. Freshness marker checked before use. Difference reported to reviewer.  |
| 6   | Evidence and audit                   | Tests, schemas, tool output, review findings and `intents/<record>/audit/` | Keyed by intent. Gate results and link checks attached to the artifact. Provenance kept for review.            |

Buckets 2 to 5 supply declarative context. Bucket 1 and the output templates in bucket 4 supply procedural context. Bucket 6 supplies evidence context.

Each bucket returns the same bundle shape: ranked candidates, deduplicated nodes, a token budget and provenance. Buckets 1 and 4 always load. Buckets 2 and 3 produce seeds by exact key. Full-text and vector search produce seeds only when the caller sends text. Buckets 3 and 5 expand seeds through one graph hop. Bucket 6 attaches evidence to the result. The assembler stops when the budget is exhausted and attaches `sourceUrl` and `commitSha` provenance to the bundle ([SDD](/blog/ai/fintech/fintech-documentation-graph-sdd/)).

The [context engineering notes](/blog/ai/fintech/context-engineering/) cover retrieval quality, just-in-time loading and tool selection in detail.

### Transition

Context can state a rule. Only a harness can stop an action. The next section covers the difference.

## Harness engineering: enforce the rule

> **TODO — block 3.** Write after review of block 2.

## One core, many harnesses: the AWS AI-DLC decision

> **TODO — block 4.** Write after review of block 3.

## Measurement spine

> **TODO — block 5.** Write after review of block 4.

## Where AI acts and where it does not

> **TODO — block 6.** Write after review of block 5.

## What I learned

> **TODO — block 7.** Write after review of block 6.

## References

> **TODO — block 7.** Collect all cited sources here. Sources used so far: Sarkar and Drosos, METR July 2025, METR February 2026, Stack Overflow 2025, Chroma 2025, Liu et al. 2023, Anthropic, Weaviate, AWS AI-DLC.
