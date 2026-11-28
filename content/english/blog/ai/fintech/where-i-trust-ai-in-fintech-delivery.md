+++
date = '2026-11-14T12:00:00+10:00'
draft = true
title = 'Where I Trust AI in Fintech Software Delivery'
tags = ['AI', 'Fintech', 'Software Delivery', 'Trust', 'Context Engineering', 'Evals', 'Agentic AI']
summary = "How token limits, reusable capabilities and a knowledge structure changed where we trust AI."
+++

<!-- Status: full story draft. Persona decision rights are grounded in the earlier persona post. Review before clearing draft. -->

I lead the AI work for our delivery organisation. When the first coding models reached us, I expected the hard part to be adoption. I was wrong. The hard part was learning where a fast answer could be trusted.

This is the sequence that changed my view. Each step solved one problem and exposed the next one.

## Look what it can do

When the tools first arrived, I asked teams to try them on real work rather than toy prompts. A developer asked for a payment-status endpoint and got a working handler. A second request produced tests. A third produced a short explanation of the design. The answers arrived in minutes.

I remember the feeling in those reviews. We were comparing the model with the blank page, not with a reviewed design. A first version looked like progress because it existed.

A first version is not a delivery. A fintech change still needs an approved rule, a test that proves the rule, a reviewer who understands the audit consequence and a rollback path. The model gave us the first artifact. It did not give us the other four.

I started to separate two questions. Can the model produce something that looks right? Can we show that it is right? Early demos answered only the first question.

## The token wall

Then the constraint became visible. We were encouraged to use AI more. People on my teams ran out of token allocation before the sprint ended. Some waited for the monthly reset. Others asked me for an extension.

I stopped treating tokens as an API detail and started treating them as a budget. A budget changes behaviour. If a request fails on the last day of a sprint, the cost is not latency. The work stops until the next cycle.

We changed how we worked. We cut restatement. We stopped pasting whole repositories into a request. We asked which document actually carried the rule and loaded that one. A request that failed after three retries cost more than a request that succeeded once, so reducing rework became a way of protecting the allocation.

There was a second cost. Anthropic describes context as a finite attention budget with diminishing returns. Every new token draws on that budget ([Anthropic, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). Long threads were not just expensive. They made recall worse.

I learned to ask two questions before any request. What does the model need to see? What can it safely ignore? Those questions became the start of context engineering, long before we had a name for it.

Token accounting also changed the shape of a request. We treated the window as a sequence of decisions: standing rules first, then the approved spec, then the current repository state, then evidence from the last run. Material that did not change the next action stayed out. When a thread grew too long, we wrote the decisions to a file and restarted from that file instead of carrying the transcript. Anthropic calls this structured note-taking, and it is the practical answer to lossy compaction ([Anthropic, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). A compacted summary can keep the shape of a conversation and drop the one constraint that mattered. A file with named fields makes that loss visible.

We did not remove the budget problem. We made it manageable. A bounded bundle costs less, fails less often and leaves room for rework.

## Building the capability layer

Teams around me responded by building capabilities. Skills, agents, custom instructions, hooks, scripts and MCP tools appeared in repositories. Each one started as an instruction written down in a form the model could reuse.

Anthropic defines prompt engineering as methods for writing and organising LLM instructions ([Anthropic, 2025](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). That is exactly what these artifacts were. A skill captured a repeated task. A custom instruction carried a standing rule into every session. A hook ran a check at a fixed point. A script made a step deterministic. An MCP tool gave the agent a controlled way to reach an external system.

This solved a real problem. A prompt in a file can be reviewed and diffed. A skill can be handed to another engineer. Two people starting the same task no longer depend on two different conversational phrasings. We moved from asking well to having a reusable way of asking.

I still see demos that stop here. The team shows a library of prompts, a set of skills and an agent that chains them together. The instruction is repeatable. The facts are not. The prompt can say "follow the payment-limit policy". It cannot tell the agent which policy version is current, where the ADR lives or whether the repository has already changed.

A prompt can ask the agent to run tests. It cannot stop the agent from shipping when the tests fail. That is a harness job.

So the capability layer answered "how to create" for a growing set of tasks. It left "what to create" untouched.

There was also a governance problem inside the capability layer. Skills and instructions were written by whoever needed them. Two teams could carry two versions of the same standing rule. A prompt that quoted an old policy kept working long after the policy changed. More prompt files created more places for rules to drift apart.

We started treating each capability as a repository artifact. It has an owner, a review path and a version. A hook that enforces a check is reviewed like code. A skill that carries a rule is reviewed like a standard. An MCP tool is granted only the access its task needs. None of that gives the capability the facts it needs. Governance stops a capability from becoming an unowned instruction. It does not tell the agent which policy version is current.

## The problem that prompts did not solve

This is where the context failures showed up. Weaviate names four. Context poisoning puts an error into the context and later steps build on it. Context distraction pushes the agent to repeat past behaviour instead of reasoning about the current task. Context confusion lets irrelevant tools or documents pull the model toward the wrong instruction. Context clash leaves it choosing between contradictory assumptions ([Weaviate, 2025](https://weaviate.io/blog/context-engineering)).

Length made each failure more likely. Chroma tested 18 models and found that accuracy becomes unreliable as input length grows, even on simple tasks ([Chroma, 2025](https://research.trychroma.com/context-rot)). Liu et al. found that recall is highest when relevant information sits at the beginning or end of a long context and degrades in the middle ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)). When a window filled, the harness compacted or pruned history. Compaction is a summary, and a summary can drop the constraint that mattered.

In fintech these are not abstract failures. A stale policy version becomes a wrong limit. A retrieved customer message becomes an instruction. A summary of a design discussion loses the one restriction that kept a control in place.

I stopped trying to fix this with larger windows. A larger window holds more material. It does not decide which material is true.

Here is the failure I use when I explain this to new teams. A feature document links to a payment-limit policy. The policy was superseded, but the link still points at the old document. An earlier turn in the conversation states the new limit. The retrieved chunk states the old limit. The agent now holds a clash. If it resolves the clash toward the retrieved chunk, the generated code enforces the wrong control. If the conversation is compacted before review, the correction can disappear and the wrong limit survives as the only visible value. No amount of instruction wording fixes that. The fix is status on the policy, one approved version in the bundle and provenance that shows which version was used.

## A structure for what to create

My answer was a taxonomy and an ontology. Both are our project design. They are not an external standard and make no conformance claim. The [documentation graph specification](/blog/ai/fintech/fintech-documentation-graph-sdd/) is where we wrote them down.

The taxonomy names the classes. A repository contains domains. A domain contains applications. Applications and folders contain documents. Documents contain sections. Sections contain chunks with embeddings. Every class has a defined identifier and an owner.

I keep the taxonomy and the ontology separate on purpose. The taxonomy says what a thing is. The ontology says which relationships it may have. A document can link to another document. It cannot become a person. A section can sit inside a document and inside a folder through containment. It cannot acquire an owner of its own without a front matter field. That constraint sounds obvious on paper. In a retrieval system it stops an agent from inventing a relationship that looks plausible and is not true.

The ontology names the relationships. `CONTAINS` preserves hierarchy. `LINKS_TO` preserves the reason one document refers to another. `HAS_PERSONA` records who a document serves. `HAS_SECTION` attaches structure. `EMBEDDED_WITH` records vector provenance. These relationships let an agent move from a feature to the documents that explain it, and from a changed document to every section that references it.

Front matter carries the control data. A document has a status, a persona, an owner and an update time. Chunks carry `sourceUrl` and `commitSha`. The retrieval service returns a ranked, deduplicated and token-budgeted bundle with provenance. Full-text and vector results keep separate scores and are combined as ranks rather than as raw scores. A repository filter runs before the bundle is built.

The assembly pipeline is deliberately boring. Full-text search and vector search produce seed candidates through separate statements. Each statement returns its own score. The assembler converts each ordered list to ranks and combines the ranks with configured weights. It never compares a raw full-text score with a raw vector score, because the two scales do not mean the same thing. Seeds expand through links and containment. Duplicates are removed by identifier. Per-repository caps stop one large repository from dominating the result. A token budget trims the bundle. Every item keeps `sourceUrl` and `commitSha` so a reviewer can trace a generated change to the material that shaped it ([documentation graph specification](/blog/ai/fintech/fintech-documentation-graph-sdd/)).

The access boundary runs before retrieval. A consumer identity and an allowed repository set are applied to the query. A document that the caller may not read is not retrieved and then hidden. It never enters the candidate set. That distinction matters in fintech, where a document can contain customer data or an unreleased control.

That structure is what decides what to create. The agent no longer guesses which rule applies. It follows a key, then a relationship, then a status filter.

We separated the context into three kinds. Declarative context states what to create: requirements, ADRs, domain rules and current repository state. Procedural context states how to create it: standing rules, output templates, skills and workflow steps. Evidence context states what counts as done: tests, schemas, tool output and review findings. The split is ours, not a standard.

The AWS AI-DLC workspace gave us the hierarchy for authority. `memory/` holds `org.md`, `team.md`, `project.md` and `phases/`. `templates/` holds one output-format override per artifact ([AWS Spaces and Intents](https://awslabs.github.io/aidlc-workflows/guide/03-spaces-and-intents/)). A more specific file overrides a general one. Conflict is resolved by level before the model reads either rule.

We use six retrieval buckets on top of that hierarchy:

| #   | Bucket                               | Source of truth                                                            | Retrieval implementation                                                                                       |
| --- | ------------------------------------ | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 1   | Standing rules                       | `memory/org.md` → `team.md` → `project.md` → `phases/`                     | Always loaded. Resolved by specificity. Versioned in git. No similarity search.                                |
| 2   | Domain and regulatory documents      | `knowledge/documents/` originals and the derived `documentkb/`             | Key lookup by document id, persona and status. Full-text and vector seeds. Freshness filter on every result.   |
| 3   | Architecture decisions               | ADRs and non-functional requirements                                       | Relationship navigation from the feature document to linked ADRs. `SUPERSEDED` entries excluded.               |
| 4   | Requirements and acceptance criteria | Feature documents, units and output templates                              | Key lookup by `featureKey`. Template-constrained extraction. Filled template compared with the knowledge base. |
| 5   | Code reality                         | `codekb/<repo>/` and AST-derived documentation                             | Key lookup by repository and component. Freshness marker checked before use. Difference reported to reviewer.  |
| 6   | Evidence and audit                   | Tests, schemas, tool output, review findings and `intents/<record>/audit/` | Keyed by intent. Gate results and link checks attached to the artifact. Provenance kept for review.            |

Buckets 2 to 5 carry declarative context. Bucket 1 and the templates in bucket 4 carry procedural context. Bucket 6 carries evidence. Each bucket returns the same bundle shape. Standing rules and output templates always load. Exact keys produce seeds. Full-text and vector search produce seeds only when the caller sends text. Graph relationships expand seeds through one hop. The assembler applies source caps and a token budget, then attaches provenance.

Templates matter as much as the graph. A template tells the agent which fields must be filled and which source fills each field. Our construction work extracts a structured document from code through `code → AST → documentation` against a fixed Markdown template, then compares the filled template with the knowledge base from inception and elaboration ([Fintech AI-DLC approach](/blog/ai/fintech/fintech-aidlc-approach/)). Without the template, retrieval has no target shape.

## Filling the structure

Once the structure was defined, populating it became the work. This is where taxonomy meets operations.

Git is the source of truth. We populate GitHub templates from Confluence at the start, so existing approved material becomes the first version of the knowledge base. After that, the direction changes. A CI/CD pipeline syncs the approved content back into Confluence as a read-only projection.

Read-only is the important word. A reviewer in Confluence can see the current rule. A reviewer cannot change the rule there. The change happens in the repository, goes through review and lands through the pipeline. If the projection and the repository disagree, the repository wins.

The catalog under `knowledge/documentkb/` is derived from `knowledge/documents/`. It is rebuildable. Hand-editing it is a mistake. Originals stay under human ownership. The model never rewrites an original to make a conflict disappear. `codekb/<repo>/` carries architecture, a component inventory and a freshness marker. `intents/` carries the record of each piece of work and its audit trail ([AWS Spaces and Intents](https://awslabs.github.io/aidlc-workflows/guide/03-spaces-and-intents/)). A freshness check runs before an agent uses code knowledge. A stale component inventory is treated as missing context, not as a minor warning.

The sync has an order. A change lands in Git and passes review. The pipeline validates front matter, resolves links and checks that referenced documents are published. Only then does it publish the read-only projection. A rebuild drill proves that the catalog can be regenerated from originals without manual repair. If the drill fails, the knowledge base is not trustworthy and the agent should not use it as a source of truth.

I treat that drill as part of the product. A knowledge base that cannot be rebuilt is a set of files with no recovery path. A knowledge base that rebuilds cleanly gives every later decision a place to point.

This is the part of the story I expected to be boring. It turned out to be the part that made the rest possible. A structure that nobody maintains becomes a second source of wrong answers.

## The persona ceiling

I lead AI, so I measure outcomes beyond the developer terminal. That changed the story again.

Development is only a part of the software delivery lifecycle. Our knowledge and context work produced a clear gain inside development. The gain stayed inside that phase. A faster developer did not make a slower approval chain faster. A better generated change did not help a business analyst who still reconstructed intent from a document. A cleaner test suite did not help a compliance reviewer who received the evidence after the change was already built.

I could show improved developer throughput and still watch a release wait on decisions that had nothing to do with code. That is the persona ceiling. Optimising one role cannot move a lifecycle that depends on many roles.

We are working with AWS as an active partner to develop a lifecycle that extends across all personas. The scope is the full delivery process, not code generation alone. The AWS reference implementation already ships five phases, 33 stages, 14 agents, 11 workflow profiles and a 134-event audit taxonomy ([AI-DLC workflows](https://github.com/awslabs/aidlc-workflows), [phases and stages](https://github.com/awslabs/aidlc-workflows/blob/main/docs/guide/04-phases-and-stages.md)). Those numbers matter because the work is represented as stages and gates rather than as a single long conversation. A scope selects which stages run. A regulated feature can run the full lifecycle. A bugfix runs a small part of it.

The lifecycle also makes human involvement a choice per unit rather than a blanket rule. We use three modes from our earlier work. HITL means a human blocks before each significant step. OHOTL means a human observes and can interrupt. AHOTL means a human reviews at completion ([Fintech AI-DLC approach](/blog/ai/fintech/fintech-aidlc-approach/)). Novel domains, architecture decisions and production data stay in HITL. Work with clear acceptance criteria and strong test coverage can run in AHOTL.

That choice forced me to state what we were measuring. We tracked time to first working version, review time, rework count and defects that escaped the first review. The first metric moved most. The last two decided whether a gain was real. A faster first version that doubles rework is not a gain. Those four metrics are the spine of the measurement we now use across personas.

| Metric                        | Question it answers                                   | Why I keep it                                               |
| ----------------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| Time to first working version | How fast did the model produce a plausible artifact?  | It measures generation. It says nothing about correctness.  |
| Review time                   | How long did a human spend checking the artifact?     | It exposes the bottleneck that generation moved downstream. |
| Rework count                  | How many rounds did the change need before it passed? | It connects rework to token spend and to reviewer load.     |
| Escaped defects               | What reached production after the first review?       | It decides whether a faster path is also a safe path.       |

I stopped reporting a single speed number to leadership. A speed number without review time and escaped defects invites the wrong decision. The four metrics together show where a persona gains time and where the lifecycle absorbs it.

The partnership is about extending that structure to the roles around the developer. The model can draft artifacts for each role. The role keeps the decision that carries accountability.

## What the proof of concept changes

In the proof of concept the developer's responsibility shifts. The developer moves toward code review and architecture decision making. Generation is no longer the scarce act. Judgment about whether a generated change satisfies a spec and an ADR is the scarce act.

The product owner decides on intents. Our [persona model](/blog/ai/fintech/fintech-aidlc-approach/) gives the product owner intent, success criteria and sign-off, with requirements approval and release approval as gates.

The remaining roles keep their own decisions. The role definitions come from that persona model. The proof of concept assigns each decision to the person already accountable for it:

- Business analysts own requirements elicitation and domain modelling. They decide whether the rule captured in the knowledge base matches the business rule.
- Subject matter experts own domain correctness. They decide whether an edge case in a payment or entitlement flow is represented accurately.
- Testers and SDETs own test strategy and traceable test evidence. They decide whether the evidence proves the acceptance criteria, not whether the generated tests pass.
- Solution architects own design gates, ADRs and trade-offs. They decide whether a change fits the domain design and the non-functional requirements.
- Security and compliance own their gates. They decide whether an auditable checkpoint and the automated compliance evidence are sufficient.
- SREs own operations, rollback and SLOs. They decide whether a change is ready to run.
- Chapter leads own the method. They decide how the memory and knowledge layers change.

The point of the shift is not to remove people from delivery. It is to move each person from producing the first draft to owning the decision that only a human can sign.

The harness makes that possible. Rules and sensors turn a written standard into a gate. An additive gate, once confirmed, cannot be removed by a later builder. Auditable checkpoints sit at requirements sign-off, design approval, code review and release approval. The audit trail is produced by the lifecycle rather than assembled after an incident.

Each role also has a gate it cannot delegate to the model. Requirements approval and release approval sit with the product owner. Design review against non-functional requirements sits with the architect. Coverage floor and acceptance tests sit with the tester. Auditable checkpoints and automated compliance gates sit with security and compliance. Runbooks, rollback readiness and SLO conformance sit with SRE. A gate is not a suggestion in a prompt. When a gate fails, the lifecycle does not advance.

Rules carry standing constraints. Sensors check objective facts. A rule can state that no change may write to production without an approved intent. A sensor can check that the intent exists and carries a release approval. A rule can state that an ADR is required for a boundary change. A sensor can check that the ADR is published and linked from the feature document. A rule can state a coverage floor. A sensor can read the test report. The distinction matters because a rule alone can be ignored by a model. A sensor produces evidence that the harness can act on.

This is why the developer's change in responsibility matters. The developer used to be the person who produced the artifact and the person who checked it. In the proof of concept those acts separate. The model produces. The developer reviews against the spec and the ADR. The architect owns the decision when the change affects boundaries. The accountabilities do not move to the model.

## What I trust now

Trust, for me, is no longer a feeling about a model. It is a property of a path from an approved source to a checked result.

I trust AI to retrieve a defined artifact and show its source. I trust it to fill a template whose fields are named in advance. I trust it to generate a first version of code, tests and documentation when the spec and the acceptance criteria are already approved. I trust it to assemble an evidence bundle for a reviewer.

The boundary is simple enough to state as a rule. AI may retrieve, draft, assemble and check. Humans approve the rule, the design, the evidence and the release. Where money moves or entitlements change, a named person signs. Where production data is read or written, a named person signs. Where the rules that govern the agent change, a named person signs.

I do not trust it to choose the governing rule. I do not trust it to decide which policy version is current. I do not trust a summary to carry an unseen constraint. I do not trust a passing build to stand in for an approved control. I do not trust a generated document to become a source of truth because it reads well.

The sequence that got me here was not a model upgrade. It was a budget, a capability layer, a taxonomy, a knowledge pipeline and a lifecycle that includes every persona. Each step narrowed what the model had to guess.

That is where I trust AI in fintech software delivery: inside a bounded path, with provenance, with evidence and with a person accountable at the end.

## References

- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic.
- [Context engineering](https://weaviate.io/blog/context-engineering) — Weaviate.
- [Context rot: how increasing input tokens impacts LLM performance](https://research.trychroma.com/context-rot) — Chroma.
- [Lost in the middle: how language models use long contexts](https://arxiv.org/abs/2307.03172) — Liu et al.
- [Spaces and intents](https://awslabs.github.io/aidlc-workflows/guide/03-spaces-and-intents/) — AWS AI-DLC.
- [AI-DLC workflows](https://github.com/awslabs/aidlc-workflows) and [phases and stages](https://github.com/awslabs/aidlc-workflows/blob/main/docs/guide/04-phases-and-stages.md) — AWS.
- [Fintech documentation graph specification](/blog/ai/fintech/fintech-documentation-graph-sdd/) — our taxonomy, ontology and retrieval design.
- [Fintech AI-DLC approach](/blog/ai/fintech/fintech-aidlc-approach/) — persona model, gates and decision rights.
