+++
date = '2022-10-24T09:00:00+10:00'
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
- **single thread scoped**, **checkpointed** lets a thread resume without replaying everything. 
- **risk**: left unmanaged, it fills the window with stale assumptions and large tool outputs - **context distraction**.

#### Long-term memory
- not kept in the prompt. 
- persists after the task ends - written to a store and selected into context only when relevant. 
- **risks**: storing something wrong, keeping it too long or retrieving it in the wrong situation - **context poisoning**.

### By content

Content based categorization can be divided broadly into 3 categories: 

#### Semantic memory: facts

Semantic memory holds facts about **an entity or a domain**, such as **preferences**, **attributes** and **stable details** (fact remains true across many tasks). It can be kept as:
- **single profile**: _updated over time_, _is easy to read_ but can grow unwieldy and is harder to update without overwriting something correct. 
- **collection or seperate records**: searched when needed, scales better but depends on retrieval quality, and records may contradict each other as the facts change.
- **risk**: A fact that was true earlier may no longer be true, and the model has no way to know unless the **record carries a timestamp and source**. 
- **Semantic memories** (facts the agent learned from interactions) also overlap with **retrieved knowledge**(retrieved knowledge comes from a curated source of record). 

#### Episodic memory: past events
- stores what happened: **a previous task**, how it was handled and what **the outcome** was. 
- most common use is as few-shot examples, where a past success is retrieved and shown to the model as a pattern to follow. 
- **risk**: Context Distraction and retrieval mismatch.
Episodes should record the action and outcomes, so a failed attempt is not reused as a model to follow.

#### Procedural memory: how to behave
- Procedural memory holds the **rules and instructions that govern behavior**. 
- such as the instructions applied based on glob patterns - so JPA standards applied to files matching an `entities/*.java` pattern. 
- **It differs from the other two types in that it is often meant to change how the agent behaves rather than what it knows.**
- **risk**: **persisting a bad rule**. 
Some systems let the agent refine its own instructions based on feedback, similar to reflection approaches such as Reflexion. This is powerful but also involves an element of risk. In regulated settings, procedural memory should be changed through reviewed processes and versioned, not edited silently by the model.

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

### Instructions

- Instructions are the standing rules that govern behavior: **the model's role, the task boundaries, tone, escalation rules and compliance requirements.** They are usually the most stable part of the context and are **written by the developer rather than the user**. In regulated settings they often carry **security and governance obligations**. Because they **apply on every call,** they should be **concise and unambiguous**. 
- risks: **Overlapping or contradictory instructions are a common source of context clash**.

### User input and conversation history

- **User input** is the **current request**. 
- **Conversation history** is the **record of earlier turns**, which gives the model continuity but also accumulates stale assumptions, abandoned goals and superseded answers. 
- History is the part of the context that grows without anyone deciding to add to it, so it usually needs an explicit retention policy such as **trimming**, **summarizing** or **moving durable facts into state**.

### Retrieved knowledge

- Retrieved knowledge supplies information the model was not trained on or that changes over time: policies, product documents, code or search results. 
- It is selected dynamically against the current task, often through embeddings, keyword search or graph traversal. 
- Retrieval adds the risk of including irrelevant, outdated or unauthorized material, so the retrieved set needs checks for relevance, freshness and access. 
- Keep source references with each passage so a reviewer can tell evidence from model-generated conclusions.

The **quality of retrieval (RAG) depends on several choices, not just the embedding model**:

- **Chunking.** How documents are split determines what a retrieved passage can contain. Chunks that are too small lose surrounding meaning; chunks that are too large waste tokens and dilute relevance. Splitting along natural structure (sections, clauses, functions) usually beats fixed-size splits.
- **Hybrid search.** Combining semantic (embedding) search with keyword search catches both paraphrases and exact identifiers such as account numbers, product codes and policy references, which embeddings often handle poorly.
- **Reranking.** A second pass that re-scores the top candidates against the actual query, so that only the best few are placed in the window.
- **Metadata filtering.** Apply access control, freshness and jurisdiction filters before ranking, not after.
- **Evaluation.** Measure retrieval separately from generation (did the right passage come back?) so a failure can be located.

### Tool definitions

- Tool definitions **describe** what the model can do: each tool's **name**, **purpose** and **parameters**. 
- They consume context on every call in which they appear, and they shape the model's choices. 
- **Risks**: A _large catalog_ of _overlapping tools_ increases the chance of a wrong selection, which is why tool loadouts and tool-gated context are useful. A **definition describes capability** only; **permissions** still need to be enforced in the tool or application.
- Tool descriptions are effectively prompts, and their quality matters. Clear names, unambiguous purposes, well-described parameters and minimal overlap between tools all improve selection accuracy. The [Model Context Protocol (MCP)](https://modelcontextprotocol.io) is now a common standard for exposing tools and data to models. It makes integration easier, but it also makes **it easy to connect many servers and flood the window with tool definitions**. Treat every connected **MCP server as both a context cost and a trust decision**.

### Tool calls and results

- In agentic workflows, tool results are often the largest part of the context. 
- File contents, API responses, query results and command output can be long, noisy and sometimes wrong. 
**risks:** They are also a place where errors enter persistent context: a mistaken result that is summarized or written to a plan can later be treated as established fact. 
- Record which statements came from a tool and which came from the model, and filter or summarize large outputs before the next step.
- Good tools help here by design: return only the fields needed, support pagination and filtering, enforce a maximum result size and give clear error messages that the model can act on.

### Memory

- Memory is information retained outside the active prompt and loaded when relevant, such as user preferences, past decisions or operating rules. 
- It is what lets an agent behave consistently across sessions. 
- Memory is written, selected and updated by separate mechanisms, and stored items can become outdated or incorrect (see *Types of memories*).

### Examples and output specifications

- Examples show the model the intended behavior through representative input-output pairs. 
- Output specifications, such as JSON schemas, templates and required fields, constrain the form of the response so downstream systems can use it. 
- Both are developer-authored and stable, and both are worth keeping small. 
- **Risks:** A poorly chosen example can be copied too literally, and an overly rigid schema can force the model to fill fields it has no evidence for.

### Environment and task state

- State describes where the agent is in the work: the current date, the plan, completed and pending steps, user or account attributes, and the permissions in force. 
- A **compact task trail**, the **breadcrumbs**, lets the agent **resume without replaying the full history**. 
- State is also the natural place to separate what is stored from what is shown to the model on a given step.

### Non-text inputs

- Images, PDFs, spreadsheets and audio transcripts also count as context and are often expensive in tokens. 
- They are typically better handled by processing them outside the prompt, with a tool or sandbox returning only the relevant extract.

### Why the source of each item matters

- These components differ in trust and lifespan. 
- Developer-authored instructions, user statements, retrieved documents and tool outputs should not carry the same authority, and a model may be unable to tell them apart unless the system labels them. 
- Instructions embedded in retrieved content or tool output are also a route for prompt injection (see *Security: prompt injection and trust boundaries*), so untrusted content should be clearly marked and should not override standing instructions.
- Each component competes for the same limited window. 

## Context mismanagement issues

These four failure modes were named and described by Drew Breunig in [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html). LangChain's context engineering article draws on the same framing.

### Context poisoning

- Context poisoning occurs when an **incorrect claim** or **observation** is saved in the **context and then reused**. 
- A **mistaken tool result** or a **model's unsupported conclusion** (claim without evidence) can become part of a **plan or summary**. 
- Later steps may **treat it as established information** and **repeat** or **build on the error**.
- A well-known example [Gemini 2.5 Pro playing Pokémon](https://arxiv.org/abs/2507.06261), Parts of the context became "poisoned" with misinformation about the game state. Because the agent kept re-reading its own notes, the false information was slow to undo, and the model could become fixated on goals that were impossible or irrelevant.
- The practical implication is that persistent context needs provenance and **correction paths**. Systems should **distinguish** observed **tool results** from **model-generated conclusions** and should **revise or remove stale claims** when later evidence contradicts them.

### Context distraction

- Context distraction occurs when a **growing history pulls the model** toward **repeating past** actions instead of reasoning freshly about the current task. 
- The same Gemini 2.5 report observed that once the agent's context grew big, it tended to favor repeating actions from its history over synthesizing new plans. 
- A larger context window permits more material to fit, but **capacity alone does not ensure that all included material remains useful**. 
- When an agent needs to reason over a growing trajectory, **retain the parts needed to continue the task and condense or remove the rest**. 
- The point at which distraction appears varies by model and task.

### Context confusion

- Context confusion occurs when **irrelevant information affects the model's response**. 
- Irrelevant tool descriptions are one example: **a model that receives many overlapping tools** has more opportunities to choose a tool that does not fit the request. 
- Retrieving only tools relevant to the current task is one proposed response;

### Context clash

- Context clash occurs when **instructions or facts in the context disagree**. 
- Contradictions can arise between **old and new user statements**, **retrieved documents**, **tool outputs** or **intermediate model answers**. 
- **A multi-turn exchange** can also preserve an early, mistaken answer after the user provides the missing details.
- Provenance, timestamps and explicit precedence rules help make conflicts visible. 
- The system should not silently merge incompatible claims. Where the conflict affects the task and cannot be resolved from trusted evidence, the agent should ask for clarification or report the uncertainty. 


### Memory governance
A memory system needs explicit rules for:

- **What qualifies for storage.** Not every statement or tool result should become a memory. Store information with a clear future use.
- **Provenance.** Record whether a memory came from a user statement, a trusted system or a model inference, along with when it was written.
- **Correction and expiry.** Define how a record is updated, how a wrong one is removed and when old records lapse. Without this, context poisoning becomes permanent.
- **Scope and access.** Keep memories separated by user, tenant or task, and enforce this in the store rather than only in the prompt. A memory loaded into the wrong session is a data leak.
- **Sensitive data.** Decide which personal or financial details must never be stored, or must be stored only in protected fields that are not sent to the model automatically.
- **Auditability.** Make it possible to see which memories were retrieved for a given response, so reviewers can trace an output back to its inputs.

Writing a memory does not guarantee that it will help. Treat memory the same way as other context: select the smallest relevant set for the current step, and measure whether it improves the result.


## Four operations for context engineering

LangChain uses four verbs to describe common approaches: **write** context outside the active window, **select** useful context for the next call, **compress** context to retain its important information and **isolate** context so each part is exposed only where needed.

Drew Breunig's follow-up article turns these four operations into six context management tactics: 
- **RAG** selects relevant knowledge,
- **tool loadouts** select relevant tools
- **context quarantine** isolates work in separate threads
- **pruning** removes unneeded material
- **summarization** condenses accrued context  
- **offloading** stores information outside the active prompt. 

### Write: preserve information outside the active prompt

Writing context means saving useful information somewhere the agent can access later instead of relying on the current prompt to remain available.

**Scratchpads** preserve working information for a task or conversation. An agent can save a plan, decisions, intermediate findings or a checklist to a file or runtime state. A later step can retrieve the relevant part. LangChain cites Anthropic's research agent, which stores its plan in memory so that the plan remains available if a long trajectory is truncated.

**Structured note-taking** is a closely related pattern highlighted in Anthropic's [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents). The agent regularly writes notes (a to-do list, a progress file, key decisions) to storage outside the window and reads them back after a context reset. This gives a long-running agent continuity without carrying the full history.

**Long-term memory** preserves information across tasks or sessions. The research LangChain cites includes Reflexion, which reuses self-generated reflections, and Generative Agents, which periodically synthesizes memories from prior experiences. These examples distinguish retaining a task's working notes from retaining information intended to shape future interactions.

Memory can store different kinds of information. LangChain's memory reference distinguishes **semantic** memory (facts), **episodic** memory (examples of past events) and **procedural** memory (instructions about how to behave). Each kind serves a different retrieval purpose. A user preference, a successful example and an operating rule should not be treated as interchangeable records.

Writing information does not guarantee that it will help. A memory system needs rules for what qualifies for storage, how the record can be updated and how errors or outdated information are corrected. The references in LangChain's article describe memory mechanisms; they do not establish that any one storage design is right for every application.

### Select: retrieve only what the next step needs

Selection brings information into the active context. The agent may read a scratchpad, load a subset of runtime state, retrieve documents or choose from a large tool catalog.

For a small, stable set of rules, an agent may always receive a few designated instruction files. For a large memory collection, selection needs a retrieval strategy. Embeddings can find semantically related records, while knowledge graphs can represent and traverse relationships. These techniques can help locate candidates, but retrieval still needs to be checked for relevance and unintended disclosure.

**Just-in-time retrieval.** Rather than loading all potentially relevant data up front, an agent can keep lightweight identifiers (file paths, record IDs, URLs, saved queries) and load the underlying data through tools only when a step needs it. This mirrors how a person uses a filing system instead of memorizing it. It keeps the window small and the information fresh, at the cost of extra tool calls and latency. Many systems combine both: load a small amount of essential context up front, then explore on demand.

Tool selection is also context selection. Overlapping descriptions can confuse an agent, so a system can retrieve a small set of tool definitions based on the request rather than expose the full catalog. LangChain links to the [Toolshed research](https://arxiv.org/abs/2410.14594) on retrieval over tool knowledge bases and identifies semantic search as one approach to selecting tools. Breunig calls the selected set a **tool loadout** and links to research on retrieving relevant tool descriptions for each request.

Retrieval quality is not just an indexing problem. In code agents, for example, matching a query to code chunks may miss structural relationships or relevant files. LangChain's article quotes a Windsurf engineer describing a combined approach: syntax-aware parsing, search, graph-based retrieval and reranking. The appropriate mix depends on the repository and task. The important design principle is to evaluate the retrieved context against the actual work the agent must perform.

### Compress: keep the useful signal

Compression reduces the amount of context while attempting to preserve what the next step needs. Two common methods are summarization and trimming.

**Summarization** uses a model or another process to distill a long interaction into a shorter representation. A summary can preserve the goal, decisions, evidence, unresolved questions and next steps while leaving out repetitive tool output. Summaries can be applied to a whole trajectory, to a phase boundary or to an individual tool result. Recursive and hierarchical approaches summarize in stages when the source material is too large to handle in one pass.

**Compaction** is the name Anthropic uses for summarizing a conversation that is nearing the window limit and starting a fresh window with the summary. The art is in choosing what to keep (architectural decisions, unresolved issues, key details) and what to drop (redundant tool output, superseded messages). One of the safest and lightest forms is **clearing old tool results** once they have been used, since the raw output is rarely needed again.

Summaries are lossy. A summary that omits a constraint, decision or source detail can cause later steps to behave incorrectly. Define what a useful summary must preserve, then test summaries against tasks that depend on those details. LangChain notes that some systems invest in task-specific summarization because the right details can be difficult to preserve automatically.

**Trimming** removes context using rules rather than asking a model to rewrite it. A system might discard older messages, cap retained tool output or remove fields that are not needed by the next step. Trimming can be predictable, but a rule can remove information that later proves important. Apply trimming where the data's role is understood and verify its effect on task performance.

Pruning can also target retrieved documents rather than conversation history. Breunig points to [Provence](https://arxiv.org/abs/2501.16214), a research approach for reducing retrieved passages to the portions most relevant to a question. Any pruning component should be evaluated for both relevance and information loss.

LangGraph's documentation, linked from LangChain's article, describes utilities for summarizing and trimming message history. Those are implementation examples; the underlying decision (what to keep for the next invocation) applies to other orchestration systems as well.

### Isolate: separate information by task and access

Isolation keeps context in separate places and exposes it selectively. Isolation can reduce cross-task interference, limit what a model sees and let a system handle large data without placing all of it in the prompt.

**Separate agents** can investigate different subproblems with their own instructions, tools and context windows. A lead agent can then receive concise findings instead of every intermediate search result. Anthropic's multi-agent research system describes this pattern for broad research questions: subagents explore independently and return compressed findings for synthesis. The same article reports significant token costs and coordination challenges, so parallel agents are most suitable when subtasks are independent enough to benefit from separate work.

Breunig calls the use of separate threads **context quarantine**. Quarantined work can reduce unrelated history in each subtask, but it is a poor fit when workers need to share a large amount of evolving context. The system still needs to decide which findings cross each boundary and how the receiving agent checks their quality.

**Separate runtime state** can keep tool output or sensitive data in fields that are not automatically sent to the model. A workflow can expose only the fields required for a given step. State schemas make the distinction explicit: one field may hold conversation messages while another stores data for later processing.

**Separate execution environments** can hold token-heavy objects such as files, images or other data outside the model's prompt. The agent can use code or tools to operate on that data and return a small result to the model. LangChain points to Hugging Face's CodeAgent and sandbox examples as illustrations of this design.

Isolation is a context design technique, not a substitute for access control. Systems still need to enforce permissions when tools read or modify external data. The cited examples show ways to limit what enters a model invocation; they do not by themselves define a complete security model.

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

## Security: prompt injection and trust boundaries

Because context is assembled from many sources, anything that reaches the window can try to steer the model. **Prompt injection** is the attack in which instructions are hidden in content the model reads (a web page, an email, a PDF, a tool result, a retrieved document) and the model follows them as if they came from the developer or user. The model has no reliable built-in way to distinguish instructions from data, so the defence has to be in the system design.

Simon Willison's ["lethal trifecta"](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) gives a practical test. An agent is at serious risk of data theft when it combines all three of:

1. **Access to private data** (accounts, emails, internal documents).
2. **Exposure to untrusted content** (anything an attacker can influence).
3. **The ability to communicate externally** (send email, make web requests, render links or images).

Removing any one leg breaks the attack. Context engineering decisions, such as what is retrieved, which tools are exposed and what is isolated, are often the way to do that.

Practical measures:

- **Label provenance and trust.** Mark each block as instruction, user input, retrieved content or tool output, and tell the model that untrusted blocks are data, not commands. This helps but is not a guarantee.
- **Separate privileges.** Use one low-privilege component to read untrusted content and a separate, constrained component to act. Pass between them only structured, validated output.
- **Gate tools by step.** Do not expose send, write or payment tools while the agent is processing untrusted input (tool-gated context).
- **Enforce permissions outside the model.** Check authorization in the tool or application, using the end user's identity, never only in the prompt.
- **Require human approval** for irreversible or high-impact actions, such as moving money or sending external messages.
- **Filter outputs and outbound channels.** Restrict where data can be sent, and treat rendered links and images as potential exfiltration routes.
- **Log what entered the context** so an incident can be traced back to its source (see *Auditability*).
- **Vet third-party tool servers.** Tool descriptions and results from an MCP server are themselves untrusted input.

## Cost, latency and caching

Context design is also cost design. Tokens in the window contribute to processing and cost, although prompt caching can reduce both for repeated content. Long contexts can increase latency. In agent loops, costs can compound as history is sent again at each step; caching can reduce the cost of repeated prefixes. Multi-agent systems can multiply token use further: Anthropic reports that its research system uses substantially more tokens than a single-agent chat, so the quality gain must justify the spend.

**Prompt caching** can reduce both cost and latency substantially for the repeated part of a prompt. Providers typically cache a stable **prefix**, and a cache hit requires the beginning of the prompt to match exactly. This leads to a design rule:

- Put **stable content first**: instructions, tool definitions, examples and output specifications.
- Put **dynamic content last**: retrieved documents, recent history, user input and state.
- Avoid changing early content (a timestamp in the system prompt, a reordered tool list) because it invalidates the cache for everything after it.

Caching creates a trade-off with compression: rewriting or trimming early parts of the history breaks the cached prefix. Compact at deliberate boundaries rather than editing continuously. Check your provider's documentation for current caching behavior (for example, [Anthropic's prompt caching guide](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)).

**Ordering also affects accuracy.** Given the position effects noted earlier, place the most important instructions at the start, put the user's actual question and key evidence near the end, and avoid burying critical facts in the middle of a long block.

## Context engineering in a stateful agent

An agent's runtime state can serve as both a scratchpad and a boundary between information that is available and information that is currently exposed. LangGraph uses state passed between graph steps and supports checkpointing so state can persist across a thread. Its long-term memory supports information stored across sessions. Each node can decide which state fields or memories to use for its model call.

This pattern separates storage from presentation. The system can retain a large state object while building a smaller prompt for each step. It can also add nodes that summarize a tool result, retrieve memory, select tools or request human input before continuing. LangChain's article describes these as framework capabilities; teams using another framework can implement equivalent decisions in their own orchestration layer.

## Observe and evaluate context changes

Context engineering is iterative. First inspect what the agent receives and how context changes across calls. Track token use, tool inputs and outputs, retrieval results and the content passed between steps. Traces help identify whether a failure came from missing context, irrelevant retrieval, a misleading tool result or an ineffective summary.

Then evaluate whether a context change improves the task. Compare the original and revised design against representative examples. Measure outcome quality and relevant operational costs, such as latency and token use. For open-ended agents, evaluation should allow different valid paths to the same result; judging only whether an agent followed one fixed sequence can reject correct behavior. Anthropic's account of its research agent describes combining test cases, output rubrics and human review to assess accuracy, completeness, source quality and tool efficiency.

Useful metrics to track include:

- **Tokens per step and per task**, plus cache hit rate.
- **Retrieval quality**: whether the needed passage was returned, and how much irrelevant material came with it.
- **Tool-call accuracy**: right tool, right parameters, and how often a call fails or is repeated.
- **Groundedness**: whether claims in the output are supported by cited evidence.
- **Task success and escalation rate**, measured against representative cases, including adversarial ones for injection.

LangChain recommends using tracing and evaluations to find where context work is useful and to test its effects. Its own products are examples of those practices, not a requirement to use a particular vendor. The general loop is: inspect the context, identify a failure, change one context decision and compare results.

## A practical design sequence

For each agent step, make the context decisions explicit:

1. **Define the step's purpose.** State what the model must decide or produce at this point in the workflow.
2. **List required inputs.** Identify the instructions, knowledge, memories, tool descriptions and feedback needed for that decision.
3. **Choose where each item lives.** Keep durable information in an appropriate store or state field instead of relying on the full conversation history.
4. **Select the smallest relevant set.** Retrieve documents, memories and tools based on the current task, loading detail just in time where possible.
5. **Set retention and compression rules.** Decide what to write, summarize, trim or expire, and preserve information needed for later verification.
6. **Define isolation and trust boundaries.** Decide which agents, workflow steps and tools can access each part of the context, and label untrusted content.
7. **Order for caching and attention.** Place stable content first and dynamic content last, with critical items at the start or end.
8. **Trace and evaluate.** Inspect representative runs, test context changes (including for security and cost) and refine the design based on observed outcomes.

The central trade-off is between providing enough context to act correctly and limiting material that can distract, conflict or carry an error forward. More context is useful only when the additional information improves the result.

## Sources

- [Context Engineering — LangChain](https://www.langchain.com/blog/context-engineering-for-agents)
- [Effective Context Engineering for AI Agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [How Long Contexts Fail — Drew Breunig](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html)
- [How to Fix Your Context — Drew Breunig](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)
- [Building Effective Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [How We Built Our Multi-Agent Research System — Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Evaluate a Complex Agent — LangSmith documentation](https://docs.langchain.com/langsmith/evaluate-complex-agent)
- [Gemini 2.5 Technical Report — Google](https://arxiv.org/abs/2507.06261)
- [Context Rot — Chroma Research](https://research.trychroma.com/context-rot)
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
- [The Lethal Trifecta — Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- [Prompt Caching — Anthropic documentation](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [Toolshed: Scale Tool-Equipped Agents with Advanced RAG-Tool Fusion](https://arxiv.org/abs/2410.14594)
- [Provence: Efficient and Robust Context Pruning for RAG](https://arxiv.org/abs/2501.16214)
- [Context Window Comparison — BenchLM](https://benchlm.ai/tools/context-window)
