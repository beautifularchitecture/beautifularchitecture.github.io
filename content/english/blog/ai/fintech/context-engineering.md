+++
date = '2026-10-21T09:00:00+10:00'
draft = false
title = 'Context Engineering - Organizing ideas for PDD'
tags = ['Context Engineering','Prompt Engineering','Generative AI','RAG','AI Agents','Security','Governance']
summary = "A practical guide to reliable, secure and auditable AI context."
+++

Context engineering is the work of deciding what information an AI system receives at each step, how the system obtains that information and what it keeps for later. The goal is not to fill the model's context window. The goal is to provide the information needed for the current task while limiting irrelevant, stale or conflicting material.

This post starts with the framework in [LangChain's article on context engineering](https://www.langchain.com/blog/context-engineering-for-agents), then follows its references to explain the ideas behind the framework. The article groups context work into four operations: write, select, compress and isolate.

## Context is working memory

An LLM receives a finite context window with each invocation. That window may contain instructions, conversation messages, retrieved knowledge, memory, tool definitions and tool results. For an agent, the contents change as the agent calls tools and receives feedback.

LangChain compares the context window to computer memory: it is the working area available to the model for the next step. Context engineering is the process of choosing what enters that working area. The relevant material varies by task and by agent step. [LangChain defines context engineering](https://www.langchain.com/blog/context-engineering-for-agents) around that choice.

Three broad context types recur in LLM applications:

- **Instructions** tell the model how to act. They include prompts, persistent rules, examples and tool descriptions.
- **Knowledge** supplies facts relevant to the task. It can include retrieved documents and information retained from prior interactions.
- **Tool feedback** reports what happened when the agent used a tool, such as search results, API responses or execution output.

These categories can overlap in a system. A memory may contain a user preference that acts as an instruction in one task and a fact in another. The design question is which information should be available for a particular model call.

## Why context needs active management

Agents can run across many model calls. Every call may add messages, tool definitions, observations and intermediate conclusions. A long trajectory can exceed the model's context limit, raise token costs and latency or reduce answer quality before the limit is reached. [LangChain describes these pressures](https://www.langchain.com/blog/context-engineering-for-agents) and links them to four failure modes described by [Drew Breunig](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html).

### Context poisoning

Context poisoning occurs when an incorrect claim or observation is saved in the context and then reused. A mistaken tool result or a model's unsupported conclusion can become part of a plan or summary. Later steps may treat it as established information and repeat or build on the error.

The practical implication is that persistent context needs provenance and correction paths. Systems should distinguish observed tool results from model-generated conclusions and should revise or remove stale claims when later evidence contradicts them. The failure mode and an example involving an agent's game state are discussed in [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html).

### Context distraction

Context distraction occurs when a large history draws the model toward repeating prior actions instead of reasoning from the current task. A larger context window permits more material to fit, but capacity alone does not ensure that all included material remains useful.

Summarization and fact retrieval are two reasons to use long context, according to Breunig's analysis. When an agent needs to reason over a growing trajectory, retain the parts needed to continue the task and condense or remove the rest. Performance limits vary by model and task, so evaluate the actual application instead of assuming that filling the window is harmless.

### Context confusion

Context confusion occurs when irrelevant information affects the model's response. Irrelevant tool descriptions are one example: a model that receives many overlapping tools has more opportunities to choose a tool that does not fit the request. Retrieving only tools relevant to the current task is one proposed response; LangChain points to research on retrieval-augmented selection of tool descriptions.

The wider lesson is to avoid presenting every available instruction, document and tool on every turn. Select the smallest useful set for the current step, then measure whether the selection improves the outcome.

### Context clash

Context clash occurs when instructions or facts in the context disagree. Contradictions can arise between old and new user statements, retrieved documents, tool outputs or intermediate model answers. A multi-turn exchange can also preserve an early, mistaken answer after the user provides the missing details.

Provenance, timestamps and explicit precedence rules help make conflicts visible. The system should not silently merge incompatible claims. Where the conflict affects the task and cannot be resolved from trusted evidence, the agent should ask for clarification or report the uncertainty. Breunig's article describes examples where information split across turns produced worse results because earlier assumptions remained in the conversation.

## Four operations for context engineering

LangChain uses four verbs to describe common approaches: **write** context outside the active window, **select** useful context for the next call, **compress** context to retain its important information and **isolate** context so each part is exposed only where needed.

| Operation | Question to answer | Typical mechanisms |
| --- | --- | --- |
| Write | What should be retained for later? | Scratchpads, task state and persistent memory |
| Select | What does this step need? | Retrieval, state filtering and tool selection |
| Compress | What can be shortened or removed? | Summaries, message trimming and output filtering |
| Isolate | What should remain separate? | Per-agent context, structured state and execution sandboxes |

Drew Breunig's follow-up article turns these four operations into six context management tactics: **RAG** selects relevant knowledge, **tool loadouts** select relevant tools, **context quarantine** isolates work in separate threads, **pruning** removes unneeded material, **summarization** condenses accrued context and **offloading** stores information outside the active prompt. The tactics map to LangChain's four operations; they are not competing categories. [How to Fix Your Context](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) discusses each tactic with examples.

### Write: preserve information outside the active prompt

Writing context means saving useful information somewhere the agent can access later instead of relying on the current prompt to remain available.

**Scratchpads** preserve working information for a task or conversation. An agent can save a plan, decisions, intermediate findings or a checklist to a file or runtime state. A later step can retrieve the relevant part. LangChain cites Anthropic's research agent, which stores its plan in memory so that the plan remains available if a long trajectory is truncated.

**Long-term memory** preserves information across tasks or sessions. The research LangChain cites includes Reflexion, which reuses self-generated reflections, and Generative Agents, which periodically synthesizes memories from prior experiences. These examples distinguish retaining a task's working notes from retaining information intended to shape future interactions.

Memory can store different kinds of information. LangChain's memory reference distinguishes **semantic** memory (facts), **episodic** memory (examples of past events) and **procedural** memory (instructions about how to behave). Each kind serves a different retrieval purpose. A user preference, a successful example and an operating rule should not be treated as interchangeable records.

Writing information does not guarantee that it will help. A memory system needs rules for what qualifies for storage, how the record can be updated and how errors or outdated information are corrected. The references in LangChain's article describe memory mechanisms; they do not establish that any one storage design is right for every application.

### Select: retrieve only what the next step needs

Selection brings information into the active context. The agent may read a scratchpad, load a subset of runtime state, retrieve documents or choose from a large tool catalog.

For a small, stable set of rules, an agent may always receive a few designated instruction files. For a large memory collection, selection needs a retrieval strategy. Embeddings can find semantically related records, while knowledge graphs can represent and traverse relationships. These techniques can help locate candidates, but retrieval still needs to be checked for relevance and unintended disclosure.

Tool selection is also context selection. Overlapping descriptions can confuse an agent, so a system can retrieve a small set of tool definitions based on the request rather than expose the full catalog. LangChain links to the [Toolshed research](https://arxiv.org/abs/2410.14594) on retrieval over tool knowledge bases and identifies semantic search as one approach to selecting tools. Breunig calls the selected set a **tool loadout** and links to research on retrieving relevant tool descriptions for each request.

Retrieval quality is not just an indexing problem. In code agents, for example, matching a query to code chunks may miss structural relationships or relevant files. LangChain's article quotes a Windsurf engineer describing a combined approach: syntax-aware parsing, search, graph-based retrieval and reranking. The appropriate mix depends on the repository and task. The important design principle is to evaluate the retrieved context against the actual work the agent must perform.

### Compress: keep the useful signal

Compression reduces the amount of context while attempting to preserve what the next step needs. Two common methods are summarization and trimming.

**Summarization** uses a model or another process to distill a long interaction into a shorter representation. A summary can preserve the goal, decisions, evidence, unresolved questions and next steps while leaving out repetitive tool output. Summaries can be applied to a whole trajectory, to a phase boundary or to an individual tool result. Recursive and hierarchical approaches summarize in stages when the source material is too large to handle in one pass.

Summaries are lossy. A summary that omits a constraint, decision or source detail can cause later steps to behave incorrectly. Define what a useful summary must preserve, then test summaries against tasks that depend on those details. LangChain notes that some systems invest in task-specific summarization because the right details can be difficult to preserve automatically.

**Trimming** removes context using rules rather than asking a model to rewrite it. A system might discard older messages, cap retained tool output or remove fields that are not needed by the next step. Trimming can be predictable, but a rule can remove information that later proves important. Apply trimming where the data's role is understood and verify its effect on task performance.

Pruning can also target retrieved documents rather than conversation history. Breunig points to [Provence](https://arxiv.org/abs/2501.16214), a research approach for reducing retrieved passages to the portions most relevant to a question. Any pruning component should be evaluated for both relevance and information loss.

LangGraph's documentation, linked from LangChain's article, describes utilities for summarizing and trimming message history. Those are implementation examples; the underlying decision—what to keep for the next invocation—applies to other orchestration systems as well.

### Isolate: separate information by task and access

Isolation keeps context in separate places and exposes it selectively. Isolation can reduce cross-task interference, limit what a model sees and let a system handle large data without placing all of it in the prompt.

**Separate agents** can investigate different subproblems with their own instructions, tools and context windows. A lead agent can then receive concise findings instead of every intermediate search result. Anthropic's multi-agent research system describes this pattern for broad research questions: subagents explore independently and return compressed findings for synthesis. The same article reports significant token costs and coordination challenges, so parallel agents are most suitable when subtasks are independent enough to benefit from separate work.

Breunig calls the use of separate threads **context quarantine**. Quarantined work can reduce unrelated history in each subtask, but it is a poor fit when workers need to share a large amount of evolving context. The system still needs to decide which findings cross each boundary and how the receiving agent checks their quality.

**Separate runtime state** can keep tool output or sensitive data in fields that are not automatically sent to the model. A workflow can expose only the fields required for a given step. State schemas make the distinction explicit: one field may hold conversation messages while another stores data for later processing.

**Separate execution environments** can hold token-heavy objects such as files, images or other data outside the model's prompt. The agent can use code or tools to operate on that data and return a small result to the model. LangChain points to Hugging Face's CodeAgent and sandbox examples as illustrations of this design.

Isolation is a context design technique, not a substitute for access control. Systems still need to enforce permissions when tools read or modify external data. The cited examples show ways to limit what enters a model invocation; they do not by themselves define a complete security model.

## Context engineering in a stateful agent

An agent's runtime state can serve as both a scratchpad and a boundary between information that is available and information that is currently exposed. LangGraph uses state passed between graph steps and supports checkpointing so state can persist across a thread. Its long-term memory supports information stored across sessions. Each node can decide which state fields or memories to use for its model call.

This pattern separates storage from presentation. The system can retain a large state object while building a smaller prompt for each step. It can also add nodes that summarize a tool result, retrieve memory, select tools or request human input before continuing. LangChain's article describes these as framework capabilities; teams using another framework can implement equivalent decisions in their own orchestration layer.

## Observe and evaluate context changes

Context engineering is iterative. First inspect what the agent receives and how context changes across calls. Track token use, tool inputs and outputs, retrieval results and the content passed between steps. Traces help identify whether a failure came from missing context, irrelevant retrieval, a misleading tool result or an ineffective summary.

Then evaluate whether a context change improves the task. Compare the original and revised design against representative examples. Measure outcome quality and relevant operational costs, such as latency and token use. For open-ended agents, evaluation should allow different valid paths to the same result; judging only whether an agent followed one fixed sequence can reject correct behavior. Anthropic's account of its research agent describes combining test cases, output rubrics and human review to assess accuracy, completeness, source quality and tool efficiency.

LangChain recommends using tracing and evaluations to find where context work is useful and to test its effects. Its own products are examples of those practices, not a requirement to use a particular vendor. The general loop is: inspect the context, identify a failure, change one context decision and compare results.

## A practical design sequence

For each agent step, make the context decisions explicit:

1. **Define the step's purpose.** State what the model must decide or produce at this point in the workflow.
2. **List required inputs.** Identify the instructions, knowledge, memories, tool descriptions and feedback needed for that decision.
3. **Choose where each item lives.** Keep durable information in an appropriate store or state field instead of relying on the full conversation history.
4. **Select the smallest relevant set.** Retrieve documents, memories and tools based on the current task.
5. **Set retention and compression rules.** Decide what to write, summarize, trim or expire, and preserve information needed for later verification.
6. **Define isolation boundaries.** Decide which agents, workflow steps and tools can access each part of the context.
7. **Trace and evaluate.** Inspect representative runs, test context changes and refine the design based on observed outcomes.

The central trade-off is between providing enough context to act correctly and limiting material that can distract, conflict or carry an error forward. More context is useful only when the additional information improves the result.

## Sources

- [Context Engineering — LangChain](https://www.langchain.com/blog/context-engineering-for-agents)
- [How Long Contexts Fail — Drew Breunig](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html)
- [How to Fix Your Context — Drew Breunig](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html)
- [Building Effective Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents)
- [How We Built Our Multi-Agent Research System — Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Evaluate a Complex Agent — LangSmith documentation](https://docs.langchain.com/langsmith/evaluate-complex-agent)
