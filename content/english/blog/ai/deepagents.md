+++
date = '2026-11-07T12:00:00+10:00'
draft = false
title = 'LangChain Deep Agents'
tags = ['LLM', 'AI', 'LangChain', 'Deep Agents', 'Agentic AI', 'Design Patterns']
summary = "Comprehensive reference for building long-running, multi-step AI agents using LangChain's Deep Agents SDK — covering planning, file systems, subagents, memory, skills, sandboxes, backends, context management, human-in-the-loop, and deployment."
+++

Deep Agents is LangChain's opinionated agent harness for building agents that can plan, delegate, remember and act across complex multi-step tasks. It sits on top of LangChain core and the LangGraph runtime and ships with built-in tools for file management, subagent spawning, context compression and long-term memory, plus optional task planning.

---

![Class diagram of the LangChain Deep Agents SDK: the create_deep_agent factory, the middleware stack, subagents, filesystem backends and sandbox protocols, result types, harness profiles and filesystem permissions](../img/png/deepagents-class-diagram.png)

## Introduction

Before the Deep Agents SDK, building a production-grade agent required you to wire together context management, tool routing, summarization, filesystem access, human approvals and deployment yourself. Deep Agents packages all of that into a single harness so you can focus on what the agent does, not how it works.

You can run the same harness two ways:

- **Deep Agents SDK** (`deepagents` on [PyPI](https://pypi.org/project/deepagents/)) — open source, you host the agent yourself. Source: [github.com/langchain-ai/deepagents](https://github.com/langchain-ai/deepagents).
- **Managed Deep Agents** — LangSmith runs the harness, Agent Server, sandboxes and schedules for you. This is the path the docs now recommend first for production.

This post focuses on the SDK, with a section on managed deployment near the end.

> **Version note.** Several features below depend on a minimum `deepagents` version. See [Version requirements](#version-requirements) at the end of the post and pin your version before copying code.

---

## Concepts

### What is an Agent Harness?

- An **agent harness** is the scaffolding that wraps the core LLM tool-calling loop with production-grade capabilities.
- Deep Agents is an opinionated harness — it ships with defaults that work well out of the box (system prompts, file tools, subagents, summarization) and lets you override each layer.
- Contrast with a **framework** (LangChain: building blocks) or a **runtime** (LangGraph: durable graph execution).

### The Tool-Calling Loop

- At its core, every agent is a loop: LLM generates a message → tools are called → results are appended → LLM generates again.
- Deep Agents uses LangGraph's compiled state graph as the runtime engine for this loop, giving it durable execution, checkpointing, and streaming for free.
- The harness adds built-in tools, context management, and middleware hooks on top of this core loop.

### Context Window Management

- LLMs have a finite context window — everything (system prompt, conversation history, tool results) must fit inside it.
- **Context bloat** is the primary failure mode of long-running agents: tool results accumulate and eventually exceed the window.
- Deep Agents addresses this with two built-in compression strategies: **offloading** (writing large content to the filesystem) and **summarization** (compressing old history). Both are automatic. Subagents add a third lever: **isolation**.

### Virtual Filesystem

- Instead of working directly with disk, Deep Agents gives agents a **virtual filesystem** backed by a pluggable backend.
- Agents write, read, search and edit files as part of their normal work — this is how they offload large tool results and share state between tasks.
- The same agent code works whether the backend is in-memory state, local disk, a cloud sandbox, or a custom store.

---

## Core Architecture

`create_deep_agent` is the single entry point. Its real signature (from the docs) is:

```python
create_deep_agent(
    model=None,              # "provider:model" string or a chat model instance
    tools=None,              # Your custom tools (functions, LangChain tools, MCP tools)
    *,
    system_prompt=None,      # str or SystemMessage; prepended to the built-in prompt
    middleware=(),           # Extra middleware merged into the Deep Agents stack
    subagents=None,          # SubAgent dicts, CompiledSubAgent, or AsyncSubAgent
    skills=None,             # Skill source directories (SKILL.md)
    memory=None,             # AGENTS.md files, always loaded
    permissions=None,        # FilesystemPermission rules
    backend=None,            # Filesystem backend (StateBackend by default)
    interrupt_on=None,       # Human-in-the-loop config per tool
    response_format=None,    # Structured output schema
    state_schema=None,       # Custom graph state (subclass DeepAgentState)
    context_schema=None,     # Per-run runtime context schema
    checkpointer=None,       # Persists state between runs
    store=None,              # Persistent store (required for StoreBackend)
    debug=False,
    name=None,
    cache=None,
)
→ CompiledStateGraph         # LangGraph compiled graph; call .invoke() / .ainvoke()
```

---

## Installation and Setup

```bash
# Core install
pip install deepagents

# Model provider packages come from LangChain extras
pip install -U "langchain[anthropic]"      # Anthropic Claude
pip install -U "langchain[openai]"         # OpenAI GPT (also used for Azure OpenAI)
pip install -U "langchain[google-genai]"   # Google Gemini
pip install -U "langchain[aws]"            # AWS Bedrock
pip install -U "langchain[huggingface]"    # HuggingFace

# With a search tool (for the quickstart)
pip install tavily-python
```

**Environment variables:**

```bash
export ANTHROPIC_API_KEY="your-key"
export OPENAI_API_KEY="your-key"
export GOOGLE_API_KEY="your-key"
export TAVILY_API_KEY="your-key"
```

---

## Quickstart — Research Agent

```python
import os
from typing import Literal

from tavily import TavilyClient
from langchain.agents.middleware import TodoListMiddleware
from deepagents import create_deep_agent

tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def internet_search(
    query: str,
    max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """Run a web search"""
    return tavily_client.search(
        query,
        max_results=max_results,
        include_raw_content=include_raw_content,
        topic=topic,
    )

research_instructions = """You are an expert researcher. Your job is to conduct
thorough research and then write a polished report.

Use internet_search to gather information. Save intermediate results to files
to manage your context window. Write a final report when done."""

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    tools=[internet_search],
    system_prompt=research_instructions,
    middleware=[TodoListMiddleware()],  # opt-in since v0.7: adds write_todos
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What is LangGraph?"}]
})

print(result["messages"][-1].content)
```

**What typically happens:**

1. The agent calls `write_todos` (because `TodoListMiddleware` was added) to break the task into steps.
2. It calls `internet_search` to gather information.
3. It calls `write_file` / `read_file` to offload large search results.
4. It may spawn a subagent if a subtask needs deep context isolation.
5. It synthesizes and returns a final report.

---

## Supported Models

Deep Agents uses a `provider:model` format string. Any LangChain chat model that supports tool calling works. The strings below are the ones used in the official docs.

| Provider     | Example Model String                                                | Notes                      |
| ------------ | ------------------------------------------------------------------- | -------------------------- |
| Anthropic    | `anthropic:claude-sonnet-5`                                         | Install `langchain[anthropic]` |
| OpenAI       | `openai:gpt-5.5`                                                    | Install `langchain[openai]` |
| Google       | `google_genai:gemini-3.6-flash`                                     | Install `langchain[google-genai]` |
| Azure OpenAI | `azure_openai:gpt-5.5`                                              | Needs Azure endpoint and key env vars |
| AWS Bedrock  | `anthropic.claude-sonnet-4-6` + `model_provider="bedrock_converse"` | Install `langchain[aws]` |
| HuggingFace  | `microsoft/Phi-3-mini-4k-instruct` + `model_provider="huggingface"` | Open source models |
| OpenRouter   | `openrouter:z-ai/glm-5.2`                                           | Proxy across providers     |
| Fireworks    | `fireworks:accounts/fireworks/models/glm-5p2`                       | Fast inference             |
| Baseten      | `baseten:zai-org/GLM-5.2`                                           | Custom deployments         |
| Ollama       | `ollama:north-mini-code-1.0`                                        | Local models               |

You can also pass an initialized model instance directly:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    model="claude-sonnet-5",
    max_retries=10,   # Increase for unreliable networks
    timeout=120,      # Increase for slow connections
)
agent = create_deep_agent(model=model, ...)
```

**Connection resilience:** LangChain chat models automatically retry transient API failures with exponential backoff. For the exact defaults and tuning options, see the *Connection resilience* section of the LangChain Models docs.

---

## Built-in Tools (Harness Capabilities)

### Planning — `write_todos` (opt-in)

- Gives the agent a structured task list it can update during execution.
- **Opt-in since v0.7:** add `TodoListMiddleware` from `langchain.agents.middleware` to `middleware=`. Earlier versions included it by default.
- Tasks have statuses: `pending`, `in_progress`, `completed`, and are persisted in agent state.
- Useful for long multi-step tasks, less capable models that benefit from an explicit accountability tool, and UIs that stream progress from agent state.

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    middleware=[TodoListMiddleware()],
)
```

### Virtual Filesystem Tools

| Tool         | Description                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| `ls`         | List files in a directory with metadata (size, modified time)                                                  |
| `read_file`  | Read file contents with line numbers; supports offset/limit; returns multimodal blocks for images, video, audio, PDF and PPT/PPTX |
| `write_file` | Create a new file, or overwrite an existing one                                                                |
| `edit_file`  | Exact string replacement in files (with global replace mode)                                                   |
| `delete`     | Delete a file, or a directory and its contents recursively (`deepagents>=0.7`; hidden on backends that don't support deletion) |
| `glob`       | Find files matching patterns (e.g. `**/*.py`)                                                                  |
| `grep`       | Search file contents — files-only, content-with-context, or count modes                                        |
| `execute`    | Run shell commands (only available with sandbox backends)                                                      |

**Hiding or restricting filesystem tools:**

- To hide tools from the model, register a harness profile with `excluded_tools` (this shrinks the baseline prompt, since unused tool schemas are sent every turn).
- To expose only a subset, pass a `tools` allowlist to your own `FilesystemMiddleware` instance (`deepagents>=0.7`). `read_file` must always be in the list.

```python
from deepagents import create_deep_agent
from deepagents.backends import StateBackend
from deepagents.middleware import FilesystemMiddleware

backend = StateBackend()

# Read-only agent: write_file, edit_file, delete, and execute are never shown
agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    backend=backend,
    middleware=[
        FilesystemMiddleware(backend=backend, tools=["read_file", "ls", "glob", "grep"]),
    ],
)
```

`FilesystemMiddleware` itself cannot be removed via `excluded_middleware`; it is required scaffolding.

### Task Delegation — `task`

- The main agent's tool for spawning subagents.
- Accepts a subagent name and task description.
- Synchronous subagents block until the subagent finishes, then return a single final report.
- All subagent intermediate tool calls are invisible to the main agent — only the result comes back.
- Subagents are stateless: they cannot send multiple messages back.

---

## Context Management in Depth

### Prompt Composition Order

When you create a deep agent, the final system prompt is assembled in this order:

1. Your custom `system_prompt` (prepended)
2. Base agent prompt
3. Memory prompt (`AGENTS.md` + usage guidelines, only when `memory` is provided)
4. Skills prompt (skill locations + frontmatter, only when `skills` is provided)
5. Virtual filesystem prompt (filesystem + `execute` tool docs where applicable)
6. Subagent prompt (`task` tool usage)
7. User-provided middleware prompts (if any)
8. Human-in-the-loop prompt (when `interrupt_on` is set)

Deep Agents Code (the terminal agent) additionally injects a local-context prompt with the current directory and project info.

### Offloading Large Content

Deep Agents automatically offloads large content when tool inputs or results exceed a token threshold (default **20,000 tokens**):

**Large tool inputs:**

- File write/edit operations leave the full file content in conversation history.
- As the session context crosses 85% of the model's available window, older tool calls are truncated and replaced with a pointer to the file on disk.
- The file is already persisted, so nothing is lost.

**Large tool results:**

- The result is written to the backend and replaced with a file path reference plus a preview of the first 10 lines.
- The agent can re-read or search the file as needed.

Built-in compression does not resize images or lower image resolution; see the Multimodal docs for how compression interacts with media.

### Summarization

When the context crosses the limit (default 85% of the model's `max_input_tokens`) and there is no more content eligible for offloading, Deep Agents summarizes the message history:

- **In-context summary:** An LLM generates a structured summary (session intent, artifacts created, next steps) that replaces the full conversation history in working memory.
- **Filesystem preservation:** A text rendering of the original messages is written to the filesystem as a canonical record.
- The agent stays aware of its goals (summary) but can recover details (filesystem search).

**Defaults:**

```
Trigger:      85% of model's max_input_tokens (from the model profile)
Context kept: 10% of tokens as recent context
Fallback:     170,000-token trigger, 6 messages kept (if model profile unavailable)
```

If any model call raises a standard `ContextOverflowError`, the deep agent immediately falls back to summarization and retries with the summary plus recent preserved messages.

**Customizing thresholds** — override the default `SummarizationMiddleware` by passing an instance (matching is done by `.name`; requires `deepagents>=0.7`). An override *replaces* the default, so it must be fully configured:

```python
from deepagents import create_deep_agent
from deepagents.backends import StateBackend
from deepagents.middleware import SummarizationMiddleware

backend = StateBackend()
model = "anthropic:claude-sonnet-5"

agent = create_deep_agent(
    model=model,
    middleware=[
        SummarizationMiddleware(
            model=model,
            backend=backend,
            trigger=("tokens", 100000),  # summarize once the conversation exceeds 100k tokens
            keep=("messages", 20),       # keep the most recent 20 messages verbatim
        ),
    ],
)
```

`trigger` also accepts `("fraction", ...)` and a list of thresholds (combined with OR).

**On-demand compaction** — give the agent a `compact_conversation` tool so it can compact between tasks instead of waiting for the threshold. Automatic summarization stays on:

```python
from deepagents import create_deep_agent
from deepagents.backends import StateBackend
from deepagents.middleware.summarization import create_summarization_tool_middleware

backend = StateBackend  # if using the default backend
model = "anthropic:claude-sonnet-5"

agent = create_deep_agent(
    model=model,
    middleware=[create_summarization_tool_middleware(model, backend)],
)
```

**Streaming tip:** tokens generated by the summarization step show up in message streams. Filter them with `metadata.get("lc_source") == "summarization"`.

### Prompt Caching

For Anthropic and Amazon Bedrock models (Claude or Nova), `create_deep_agent` automatically applies prompt caching to the static parts of the system prompt (base instructions, memory, skill content). No configuration is needed. To change the Anthropic cache TTL, override the middleware:

```python
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    middleware=[AnthropicPromptCachingMiddleware(ttl="1h")],  # default is 5m
)
```

### Runtime Context and Custom State

- **Runtime context** (`context_schema=` + `context=` at invoke time) is per-run configuration such as user IDs or API keys. It is not shown to the model unless a tool or middleware adds it. It propagates to all subagents.
- **Custom state** (`state_schema=`, `deepagents>=0.6.6`) is for data that must live in checkpointed graph state. Subclass `DeepAgentState` so the built-in `messages` reducer is preserved.

---

## Filesystem Backends

The virtual filesystem is powered by pluggable backends. Choose based on your persistence and execution needs.

### StateBackend (Default)

- Thread-scoped filesystem stored in LangGraph state.
- Files persist across turns within a thread (via your checkpointer) but are not shared across threads.
- Good for development and stateless agents.

```python
from deepagents.backends import StateBackend
agent = create_deep_agent(model="openai:gpt-5.5", backend=StateBackend())
```

### FilesystemBackend (Local Disk)

- Maps the virtual filesystem to real local disk.
- Agents can read and write files directly on your machine — use with caution.
- Tip: wrap it in a `CompositeBackend` so internal agent data (offloaded results, conversation history) isn't written next to your project files.

```python
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=FilesystemBackend(root_dir=".", virtual_mode=True),
)
```

### LocalShellBackend (Local Shell)

- Like FilesystemBackend, but adds the `execute` tool for running shell commands on the host.
- **Unrestricted shell access** — only use in safe, isolated environments.

```python
from deepagents.backends import LocalShellBackend
agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=LocalShellBackend(root_dir=".", virtual_mode=True, env={"PATH": "/usr/bin:/bin"}),
)
```

### StoreBackend (Cross-thread Persistence)

- Backed by a LangGraph Store — files persist across threads and conversations.
- When deploying to LangSmith, omit the `store` parameter — the platform provisions one automatically.

```python
from langgraph.store.memory import InMemoryStore
from deepagents.backends import StoreBackend

agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=StoreBackend(
        namespace=lambda rt: (rt.server_info.user.identity,),
    ),
    store=InMemoryStore(),  # fine for local dev; omit for LangSmith Deployment
)
```

**Important:** The `namespace` parameter controls data isolation. In multi-user deployments, always set a namespace factory to isolate data per user or tenant.

### ContextHubBackend

- Durable filesystem storage in a LangSmith Hub repo.

```python
from deepagents.backends import ContextHubBackend

agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=ContextHubBackend("my-agent"),
)
```

### CompositeBackend (Hybrid Routing)

- Routes different virtual paths to different backends.
- Example: thread-scoped state for most files, durable store for `/memories/`.
- This is the documented way to get long-term memory alongside ephemeral working files.

```python
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": StoreBackend(namespace=lambda _rt: ("memories",))},
    ),
    store=InMemoryStore(),  # passed to create_deep_agent, not to the backend
)
```

### Sandbox Backends (Isolated Code Execution)

Sandboxes are backends that also give the agent an `execute` tool inside an isolated environment. Agents can install packages, run tests and execute scripts without touching your host. Only the `execute()` method is required of a provider; the base class builds all other file operations on top of it.

| Sandbox          | Package                                   | Notes                                |
| ---------------- | ----------------------------------------- | ------------------------------------ |
| LangSmith        | `langsmith` (`langsmith.sandbox.SandboxClient`) | First-party managed sandboxes; `LangSmithSandbox` is importable from `deepagents.backends` |
| Daytona          | `langchain-daytona`                       | Cloud dev environments               |
| E2B              | `langchain-e2b`                           | Cloud sandboxes                      |
| Modal            | `langchain-modal`                         | Cloud-based containers               |
| Runloop          | `langchain-runloop`                       | Dev environments (devboxes)          |
| Vercel           | `langchain-vercel-sandbox`                | Vercel Sandbox                       |
| AgentCore        | `langchain-agentcore-codeinterpreter`     | AWS AgentCore code interpreter       |
| NVIDIA OpenShell | `langchain-nvidia-openshell`              | NVIDIA OpenShell                     |

See the sandbox integrations docs for provider-specific setup.

```python
# Example: Modal Sandbox
import modal
from langchain_anthropic import ChatAnthropic
from langchain_modal import ModalSandbox
from deepagents import create_deep_agent

app = modal.App.lookup("your-app")
modal_sandbox = modal.Sandbox.create(app=app)

agent = create_deep_agent(
    model=ChatAnthropic(model="claude-sonnet-4-6"),
    system_prompt="You are a Python coding assistant with sandbox access.",
    backend=ModalSandbox(sandbox=modal_sandbox),
)

try:
    result = agent.invoke({"messages": [
        {"role": "user", "content": "Create a Python package and run pytest"}
    ]})
finally:
    modal_sandbox.terminate()
```

**Why sandboxes:**

- Security: code runs in isolation, protecting your host.
- Clean environments: specific dependency versions without local setup.
- Reproducibility: consistent environments across teams.

**Two integration patterns:**

- **Sandbox as tool** (the pattern used in the docs' examples): the agent runs on your server and calls the sandbox through its backend. API keys stay outside the sandbox, and you can update agent code without rebuilding images.
- **Agent in sandbox:** the agent runs inside the sandbox and you talk to it over the network. This mirrors local development but puts API keys inside the sandbox and needs extra communication infrastructure.

**Security notes:**

- Sandboxes do not protect against context injection or network exfiltration. **Never put secrets inside a sandbox.** Keep authenticated calls in tools that run outside it.
- Shut sandboxes down when finished; they cost money until you do. Set a TTL for idle environments.
- Use `upload_files()` / `download_files()` from application code to move files across the sandbox boundary.

---

## Subagents

Subagents solve the **context bloat problem** — when a subtask requires many tool calls, those results would otherwise fill the main agent's context. Subagents isolate that work; the main agent receives only the final result.

### When to Use Subagents

| Use                                             | Avoid                                       |
| ----------------------------------------------- | ------------------------------------------- |
| Multi-step tasks that clutter main context      | Simple, single-step tasks                   |
| Specialized domains needing custom instructions | When intermediate context must be preserved |
| Tasks requiring a different model               | When overhead outweighs benefits            |
| Keeping the main agent focused on coordination  |                                             |

### Default General-Purpose Subagent

Every deep agent automatically gets a synchronous `general-purpose` subagent unless you provide your own with that name. It has filesystem tools, the main agent's model by default, and **inherits the main agent's skills**.

- To **replace** it, pass a subagent named `general-purpose`.
- To **re-prompt or rename** the auto-added one, set `general_purpose_subagent=GeneralPurposeSubagentProfile(...)` on the active harness profile.
- To **disable** it, set `GeneralPurposeSubagentProfile(enabled=False)` on the harness profile *and* pass no synchronous subagents. Without any synchronous subagent, `SubAgentMiddleware` (and the `task` tool) is not attached.

### Dictionary-Based Subagents (Most Common)

```python
research_subagent = {
    "name": "research-agent",              # Required; used in task() calls
    "description": "Conducts in-depth research using web search and synthesizes findings",  # Required
    "system_prompt": "You are a thorough researcher...",  # Required in isolated mode; does NOT inherit from main
    "mode": "isolated",                    # Optional; "isolated" (default) or "fork"
    "tools": [internet_search],            # Optional; inherits from main if omitted; overrides entirely when set
    "model": "openai:gpt-5.5",            # Optional; inherits from main if omitted
    "middleware": [],                      # Optional; isolated subagents do NOT inherit custom middleware
    "interrupt_on": {},                    # Optional; inherits from main; subagent value overrides
    "skills": ["/skills/research/"],      # Optional; does NOT inherit (only general-purpose does)
    "response_format": ResearchFindings,  # Optional; Pydantic model for structured output
    "permissions": [],                     # Optional; inherits by default; when set, REPLACES parent rules entirely
}

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    subagents=[research_subagent],
)
```

### CompiledSubAgent (Custom LangGraph Graphs)

For complex workflows, wrap a prebuilt, already-compiled LangGraph graph. `create_agent` already returns a compiled graph, so **do not call `.compile()` on it**. A custom graph must have a `"messages"` state key.

```python
from deepagents import create_deep_agent, CompiledSubAgent
from langchain.agents import create_agent

custom_graph = create_agent(
    model=your_model,
    tools=specialized_tools,
    system_prompt="You are a specialized data analysis agent...",
)

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    subagents=[
        CompiledSubAgent(
            name="data-analyzer",
            description="Specialized agent for complex data analysis tasks",
            runnable=custom_graph,
        )
    ],
)
```

Note: `CompiledSubAgent` and remote `AsyncSubAgent` specs do not inherit the parent's `state_schema` because their graphs are already compiled or hosted separately.

### Forked Subagents (Beta)

By default a subagent is **isolated**: it sees only the task description. With `mode: "fork"` it inherits the parent's conversation history and state instead, which is useful when the subagent continues work the parent already started (for example, drafting review comments for issues the parent found).

```python
comment_writer = {
    "name": "comment-writer",
    "description": "Continues an in-progress PR review and drafts review comments",
    "mode": "fork",
    "tools": [read_diff],
}
```

- Requires `deepagents>=0.7.13` and is in beta.
- `skills` is rejected on forks; `system_prompt` is appended to the parent's prompt as an addendum.
- Forks cannot call `task`; they must finish the work themselves.
- A fork can reuse the parent's prompt cache for the matching prefix, so avoid rewriting the shared prompt or tools if you want cache hits.

### Structured Output from Subagents

Subagents can return structured JSON to the parent instead of free-form text (`deepagents>=0.5.3`):

```python
from pydantic import BaseModel, Field

class ResearchFindings(BaseModel):
    summary: str = Field(description="Summary of findings")
    confidence: float = Field(description="Confidence score 0-1")
    sources: list[str] = Field(description="List of source URLs")

research_subagent = {
    "name": "researcher",
    "description": "Researches topics and returns structured findings",
    "system_prompt": "Research the given topic thoroughly.",
    "tools": [web_search],
    "response_format": ResearchFindings,
}
```

The parent receives the JSON-serialized result as the `ToolMessage` content.

### Multiple Specialized Subagents Pattern

```python
subagents = [
    {
        "name": "data-collector",
        "description": "Gathers raw data from various sources",
        "system_prompt": "Collect comprehensive data on the given topic.",
        "tools": [web_search, api_call, database_query],
    },
    {
        "name": "data-analyzer",
        "description": "Analyzes collected data for insights",
        "system_prompt": "Analyze data and extract key insights.",
        "tools": [statistical_analysis],
    },
    {
        "name": "report-writer",
        "description": "Writes polished reports from analysis",
        "system_prompt": "Create professional reports from insights.",
        "tools": [format_document],
    },
]

agent = create_deep_agent(
    model="google_genai:gemini-3.6-flash",
    system_prompt="You coordinate data analysis and reporting. Use subagents for specialized tasks.",
    subagents=subagents,
)
```

**Workflow:** Main agent plans → delegates collection → passes results to analyzer → sends insights to writer → compiles final output. Each subagent works with clean, focused context.

### Subagent Best Practices

**Write clear, specific descriptions** — the main agent uses descriptions to choose which subagent to call:

```
✅  "Analyzes financial data and generates investment insights with confidence scores"
❌  "Does finance stuff"
```

**Detailed system prompts with output format guidance:**

```python
system_prompt="""You are a thorough researcher.
1. Break the question into searchable queries
2. Run internet_search for each
3. Synthesize into a concise summary with citations

Output format:
- Summary (2-3 paragraphs)
- Key findings (bullet points)
- Sources (with URLs)
Keep response under 500 words."""
```

**Minimal tool sets** — only give subagents what they need:

```python
# ✅ Focused
{"tools": [send_email, validate_email]}

# ❌ Unfocused
{"tools": [send_email, web_search, database_query, file_upload]}
```

**Concise return values** — instruct subagents to summarize, not dump:

```python
system_prompt="""Return:
- Key insights (3-5 bullets)
- Confidence score
- Recommended next actions
Do NOT include raw data or intermediate tool outputs. Under 300 words."""
```

### Context Propagation to Subagents

Runtime context passed on `.invoke()` automatically propagates to all subagents:

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime

@dataclass
class Context:
    user_id: str

@tool
def get_user_data(query: str, runtime: ToolRuntime[Context]) -> str:
    """Fetch data for the current user."""
    return f"Data for user {runtime.context.user_id}: {query}"

agent = create_deep_agent(
    model="google_genai:gemini-3.6-flash",
    subagents=[{
        "name": "researcher",
        "description": "Conducts research for the current user",
        "system_prompt": "You are a research assistant.",
        "tools": [get_user_data],
    }],
    context_schema=Context,
)

# Context flows to the researcher subagent's tools automatically
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Look up my recent activity"}]},
    context=Context(user_id="user-123"),
)
```

All subagents receive the same context. For per-subagent settings, use namespaced fields (for example `fact_checker_strict_mode`) or read `runtime.config["metadata"]["lc_agent_name"]` inside a tool to branch on which agent called it.

---

## Async Subagents

Deep Agents also support **async subagents** (`AsyncSubAgent`) for long-running tasks, parallel workstreams, and mid-flight steering. Unlike synchronous subagents (which block the supervisor until they return a single report), async subagents run in parallel and expose launch, check, update, and cancel tools so the main agent can monitor progress and intervene while a task is running.

Async subagents flow through their own middleware (`AsyncSubAgentMiddleware`), independent of the synchronous `task` stack, so you can use both in the same agent.

| | Synchronous (`task`) | Async (`AsyncSubAgent`) |
|---|---|---|
| Execution | Blocks until complete | Runs in parallel |
| Steering | None mid-flight | Launch, check, update, cancel |
| Best for | Short, focused subtasks | Long-running work, parallel batches |

---

## Interpreters and Dynamic Subagents

An **interpreter** adds an `eval` tool that runs JavaScript in a scoped QuickJS runtime. It lets the agent compose tools programmatically — loops, batching, deterministic data transformations — without shell access. Interpreters do **not** provide shell access, package installs, or filesystem and network access; use a sandbox backend for those.

```bash
pip install -U "deepagents[quickjs]"
```

```python
from deepagents import create_deep_agent
from langchain_quickjs import CodeInterpreterMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    middleware=[CodeInterpreterMiddleware()],
)
```

Requires `langchain-quickjs>=0.2.0` and Python 3.11+.

**Dynamic subagents (beta):** when an agent has both subagents and the interpreter middleware, it can dispatch subagents *from code* — fanning work out across many items in loops and parallel batches, then synthesizing results. This is on by default; pass `CodeInterpreterMiddleware(subagents=False)` to require the normal `task` path. The built-in interpreter prompt treats the word "workflow" in a request as a cue to orchestrate through code.

---

## Middleware

Middleware intercepts the agent loop to add cross-cutting behaviour — logging, caching, prompt injection, PII detection, or custom hooks.

### Default Middleware (the "bare" stack)

| Middleware                         | Purpose                                                   |
| ---------------------------------- | --------------------------------------------------------- |
| `FilesystemMiddleware`             | Provides ls, read_file, write_file, edit_file, delete, glob, grep |
| `SubAgentMiddleware`               | Provides the `task` tool (attached when at least one synchronous subagent exists, which the default general-purpose subagent satisfies) |
| `SummarizationMiddleware`          | Condenses history when context grows long                 |
| `PatchToolCallsMiddleware`         | Repairs dangling tool calls when a run resumes after an interruption |
| Prompt caching middleware          | `AnthropicPromptCachingMiddleware` and `BedrockPromptCachingMiddleware` are always registered and no-op on unsupported models |

### Conditional Middleware (Added When a Feature is Used)

| Middleware                 | Activated When                      |
| -------------------------- | ----------------------------------- |
| `TodoListMiddleware`       | Passed explicitly via `middleware=` (opt-in in v0.7+) |
| `AsyncSubAgentMiddleware`  | Async subagents are configured      |
| `SkillsMiddleware`         | `skills` argument is provided       |
| `MemoryMiddleware`         | `memory` argument is provided       |
| `HumanInTheLoopMiddleware` | `interrupt_on` (or an `interrupt` permission rule) is provided |

### Where Your Middleware Lands

Middleware you pass is merged into the stack by `.name`: an instance whose name matches a built-in replaces it in place (`deepagents>=0.7`); anything else is inserted after `PatchToolCallsMiddleware` and before the profile extras, `SkillsMiddleware`, prompt caching and memory.

Declarative subagents do **not** inherit the main agent's middleware customization; pass overrides in the subagent's own `middleware` field. The general-purpose subagent does inherit overrides of default middleware.

### Custom Middleware

```python
from langchain.agents.middleware import wrap_tool_call
from deepagents import create_deep_agent

call_count = [0]  # demo only: a list lets the nested function mutate it

@wrap_tool_call
def log_tool_calls(request, handler):
    """Log every tool call."""
    call_count[0] += 1
    print(f"[Tool #{call_count[0]}] {request.name}: {request.args}")
    result = handler(request)
    print(f"[Tool #{call_count[0]}] completed")
    return result

agent = create_deep_agent(
    model="openai:gpt-5.5",
    tools=[my_tool],
    middleware=[log_tool_calls],
)
```

**Concurrency warning:** Do NOT mutate `self` attributes (or other shared values) in middleware hooks — it causes race conditions when subagents, parallel tools, or concurrent threads run simultaneously. The counter above is fine for a single-run demo; for anything real, use graph state, which is scoped to a thread.

```python
# ✅ Safe: update graph state
class CustomMiddleware(AgentMiddleware):
    def before_agent(self, state, runtime):
        return {"x": state.get("x", 0) + 1}

# ❌ Unsafe: mutate self
class CustomMiddleware(AgentMiddleware):
    def __init__(self):
        self.x = 0
    def before_agent(self, state, runtime):
        self.x += 1  # Race condition under concurrency!
```

---

## Memory and Skills

### Memory (AGENTS.md Files)

Memory files are always loaded into the agent's context at startup. They provide persistent, session-independent background that keeps the agent consistent across conversations.

- Uses the [AGENTS.md standard](https://agents.md/).
- Typically contains: coding style, project conventions, user preferences, domain guidelines.
- Content is stored in the configured backend (`StateBackend`, `StoreBackend`, or `FilesystemBackend`), and the agent can update it based on interactions and feedback.
- Unlike skills, memory is always injected — there is no progressive disclosure. Keep it minimal.
- You must add the expected memory files to the backend before creating the agent (or seed them via `files` with the default `StateBackend`).

```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langgraph.checkpoint.memory import MemorySaver

with open("./AGENTS.md") as f:
    agents_md = f.read()

agent = create_deep_agent(
    model="openai:gpt-5.5",
    memory=["/AGENTS.md"],
    checkpointer=MemorySaver(),
)

result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "What are my coding preferences?"}],
        # Seed the default StateBackend's in-state filesystem (paths must start with "/")
        "files": {"/AGENTS.md": create_file_data(agents_md)},
    },
    config={"configurable": {"thread_id": "thread-1"}},
)
```

**Long-term memory across threads** uses a `CompositeBackend` that routes `/memories/` to a `StoreBackend` (see [Backends](#compositebackend-hybrid-routing)). The path starts empty; tell the agent in the system prompt what to save there and the agent creates files on demand. Use `permissions` to make memory read-only when only application code should update it.

### Skills (SKILL.md Files)

Skills are directories containing a `SKILL.md` file with instructions and metadata. They follow the [Agent Skills standard](https://agentskills.io/).

- Skills use **progressive disclosure** — only the frontmatter of each skill is loaded at startup; the full content is loaded only when the agent determines it's relevant.
- A skill directory can include scripts, templates, reference docs, and other assets.
- Custom subagents do **not** inherit skills from the main agent — only the general-purpose subagent does. Give a custom subagent its own via the `skills` field.
- Keep each skill focused on one workflow or domain; move detailed reference material into separate files.

```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data

skill_content = open("./skills/langgraph-docs/SKILL.md").read()

agent = create_deep_agent(
    model="openai:gpt-5.5",
    skills=["/skills/"],
)

result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "How do I build a LangGraph workflow?"}],
        "files": {"/skills/langgraph-docs/SKILL.md": create_file_data(skill_content)},
    },
    config={"configurable": {"thread_id": "thread-1"}},
)
```

### Memory vs Skills Comparison

|                      | Memory (AGENTS.md)                               | Skills (SKILL.md)                  |
| -------------------- | ------------------------------------------------ | ---------------------------------- |
| Loaded               | Always, at startup                               | On-demand (progressive disclosure) |
| Purpose              | Persistent background (preferences, conventions) | Domain expertise and workflows     |
| Token cost           | Always in context                                | Only when relevant                 |
| Subagent inheritance | See subagent docs                                | Only the general-purpose subagent  |

---

## Human-in-the-Loop

Human-in-the-loop (HITL) pauses agent execution before specific tool calls and waits for a human decision. It uses LangGraph's interrupt mechanism.

**Requires a checkpointer** (e.g. `MemorySaver`) to persist state during the pause.

```python
from langchain.tools import tool
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

@tool
def delete_file(path: str) -> str:
    """Delete a file from the filesystem."""
    return f"Deleted {path}"

@tool
def read_data(path: str) -> str:
    """Read a file from the filesystem."""
    return f"Contents of {path}"

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email."""
    return f"Sent email to {to}"

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    tools=[delete_file, read_data, send_email],
    interrupt_on={
        "delete_file": True,   # Default decisions: approve, edit, reject, respond
        "read_data": False,    # No interrupts needed
        "send_email": {"allowed_decisions": ["approve", "reject"]},  # No editing
    },
    checkpointer=MemorySaver(),  # Required!
)
```

**HITL decision options (when set to `True`):**

- `approve`: Execute the tool as-is.
- `edit`: Modify the tool's input arguments before execution.
- `reject`: Skip the tool call entirely.
- `respond`: Add guidance for the agent instead of running the tool.

**Use cases:**

- Safety gates for destructive operations (delete, overwrite).
- User verification before expensive API calls (email, payment).
- Compliance workflows requiring human sign-off.
- Interactive debugging — inspect and guide agent reasoning.

HITL can be configured for both the main agent and individual subagents; a subagent's `interrupt_on` overrides the inherited value. Interrupts can also come from filesystem permission rules (see below).

---

## Filesystem Permissions

Declare permission rules to control which files and directories agents can read or write.

```python
from deepagents import create_deep_agent, FilesystemPermission

agent = create_deep_agent(
    model="openai:gpt-5.5",
    permissions=[
        # Specific rules first: first match wins
        FilesystemPermission(operations=["read", "write"], paths=["/secrets/**"], mode="deny"),
        FilesystemPermission(operations=["read", "write"], paths=["/config/**"], mode="deny"),
        FilesystemPermission(operations=["read", "write"], paths=["/data/**"], mode="allow"),
    ],
)
```

Each `FilesystemPermission` has three fields:

| Field | Type | Description |
| - | - | - |
| `operations` | `list["read" \| "write"]` | `"read"` covers `ls`, `read_file`, `glob`, `grep`. `"write"` covers `write_file`, `edit_file`, `delete`. |
| `paths` | `list[str]` | Glob patterns (e.g. `["/workspace/**"]`). Supports `**` for recursive matching and `{a,b}` for alternation. |
| `mode` | `"allow" \| "deny" \| "interrupt"` | Allow, deny, or pause for human approval. Defaults to `"allow"`. |

**Evaluation:** rules are evaluated in declaration order and the first match wins. If no rule matches, the operation is **allowed** (permissive default). Put specific rules (like a deny on `.env`) before broad allows, and end with a catch-all deny if you want an allowlist.

**Mode `interrupt`** (`deepagents>=0.6.8`) pauses for human approval on write tools (`write_file`, `edit_file`, `delete`) instead of allowing or denying. It needs a checkpointer and merges with any `interrupt_on` you set. Anchor interrupt patterns with a literal leading segment (such as `/secrets/**`); fully unanchored patterns over-fire.

**Limits and caveats:**

- Requires `deepagents>=0.5.2`.
- Applies only to the built-in filesystem tools. Custom tools and MCP tools that touch the filesystem are not covered; use backend policy hooks for those.
- **Does not apply to sandbox backends**, because `execute` can run arbitrary commands.
- With a `CompositeBackend` whose default is a sandbox, every permission path must be scoped under a known route prefix, otherwise `NotImplementedError` is raised.
- Deleting a directory is all-or-nothing: the `write` permission is checked on the target and every descendant. Exact-match behavior for deleting a plain file needs `deepagents>=0.7.3`.

**Subagents** inherit the parent's permissions by default. When you set `permissions` on a subagent, they **replace** the parent's entirely — they are not merged.

---

## Structured Output

Agents can return structured, validated data instead of free-form text using a schema as `response_format`:

```python
from pydantic import BaseModel, Field
from deepagents import create_deep_agent

class WeatherReport(BaseModel):
    location: str = Field(description="The location for this weather report")
    temperature: float = Field(description="Current temperature in Celsius")
    condition: str = Field(description="Current weather condition")
    humidity: int = Field(description="Humidity percentage")
    wind_speed: float = Field(description="Wind speed in km/h")
    forecast: str = Field(description="Brief forecast for the next 24 hours")

agent = create_deep_agent(
    model="openai:gpt-5.5",
    response_format=WeatherReport,
    tools=[internet_search],
)

result = agent.invoke({"messages": [
    {"role": "user", "content": "What's the weather like in Sydney?"}
]})

# Structured output in result["structured_response"]
report = result["structured_response"]
print(report.temperature, report.condition)
```

---

## Streaming

Deep Agents support real-time streaming through LangGraph's streaming interface:

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Research quantum computing"}]},
    stream_mode="updates",
):
    print(chunk)
```

### Stream subagents

For typed projections (messages, tool calls, values, subagents), use `stream_events`. Deep Agents add `stream.subagents`, which gives each delegated `task` call its own handle with `.name`, `.messages`, `.tool_calls`, `.output`, and nested subagent streams:

```python
stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "Research one recent advance in quantum computing."}]},
    version="v3",
)

for name, item in stream.interleave("messages", "subagents"):
    if name == "messages":
        print("[coordinator]", item.text)
    else:
        print(f"[{item.name}] started")
        for message in item.messages:
            print(f"[{item.name}]", message.text)
        print(f"[{item.name}] status: {item.status}")
```

### Tracing subagents

Every run executed by a subagent or the coordinator carries its agent name in LangSmith metadata under `lc_agent_name`, so you can filter by subagent in the UI or the SDK:

```python
from langsmith import Client

client = Client()
runs = client.list_runs(
    project_name="<your-project>",
    filter='has(metadata, \'{"lc_agent_name": "research-agent"}\')',
)
```

---

## System Prompt Composition

Each deep agent should have a custom system prompt tailored to its use case. The custom prompt is **prepended** to the built-in harness prompt — you layer on top of the defaults.

```python
agent = create_deep_agent(
    model="openai:gpt-5.5",
    system_prompt="""You are a senior software engineer specializing in Python.
    When writing code, always include:
    - Type hints
    - Docstrings
    - Unit tests
    - Error handling

    Save all generated code to files before running it in the sandbox.
    """,
)
```

`system_prompt` accepts a string or a `SystemMessage` (structured content blocks are preserved for the main agent; subagent dictionary specs stay strings). It is static; for per-run prompts (for example, admin vs read-only access) use `@dynamic_prompt` middleware that reads `request.runtime.context` or `request.runtime.store`.

**What the built-in prompt covers:** how to use the filesystem tools, subagents, context management and output formatting. You don't need to re-explain these — describe your agent's role and domain. See [Prompt Composition Order](#prompt-composition-order) for the full assembly.

### Harness profiles (beta)

Harness profiles let you adjust the assembled prompt, tools, middleware and subagents for a specific provider or model. `base_system_prompt` replaces the base prompt outright, and `system_prompt_suffix` appends to it. The built-in Anthropic and OpenAI profiles ship only a suffix. Profiles can also set `excluded_tools` and `tool_description_overrides`.

```python
from deepagents import (
    GeneralPurposeSubagentProfile,
    HarnessProfile,
    register_harness_profile,
)

register_harness_profile(
    "anthropic",
    HarnessProfile(
        base_system_prompt="You are ACME's support orchestrator.",   # main agent
        general_purpose_subagent=GeneralPurposeSubagentProfile(
            system_prompt="You are a research subagent. Cite sources.",  # GP subagent
        ),
        system_prompt_suffix="Always think step by step.",
    ),
)
```

The general-purpose subagent resolves its base prompt as: `general_purpose_subagent.system_prompt` (if set) → profile `base_system_prompt` (if set) → SDK default, with the suffix layered on top.

---

## Deployment

### Managed Deep Agents (public beta)

Managed Deep Agents (MDA) lets LangSmith run the harness, Agent Server, sandboxes and schedules. You write the agent as a project folder — `agent.py`, `instructions.md`, `skills/`, `tools/`, `middleware/`, and optional `sandbox/`, `memory.py`, `identity.py`, `channels/`, `schedules/`, `evals/` — and the `mda` CLI deploys it. It is in public beta on LangSmith Cloud.

```bash
uvx --from managed-deepagents mda init research-assistant
cd research-assistant
uv sync
uv run mda dev      # local Agent Server + LangSmith Studio
uv run mda deploy   # hosted deployment on LangSmith Agent Server
```

```python
# agent.py
from managed_deepagents import define_deep_agent

agent = define_deep_agent(
    name="research-assistant",
    model="openai:gpt-5.5",
    tools=[{"type": "web_search"}],  # provider built-in search; no extra API key
)
```

`define_deep_agent` takes the same parameters as `create_deep_agent` minus the managed ones, plus a required static `name`. The difference is who compiles the agent: with `create_deep_agent` you own the backend, store, checkpointer and server; with `define_deep_agent` the managed runtime supplies them. Every deployment gets the Agent Server API, threads, runs, streaming, and an MCP endpoint.

### Self-hosted

```bash
langgraph build   # Produces a standalone Docker image
```

Deploy the image anywhere — your own infrastructure, Kubernetes, cloud VMs — or configure a custom LangSmith Deployment. Your tools, middleware and subagents carry over unchanged between managed and self-hosted modes.

### Tracing and Observability

Connect to [LangSmith](https://smith.langchain.com) for tracing, debugging and evaluation:

```bash
export LANGSMITH_API_KEY="your-key"
export LANGSMITH_TRACING=true
```

LangSmith Engine monitors traces, detects issues, and proposes fixes.

### Forms of Deep Agents

* **Deep Agents SDK** (`deepagents` PyPI package) — self-hosted, full control, described throughout this post.
* **Managed Deep Agents** — LangSmith-hosted runtime (public beta; `mda` CLI).
* **Deep Agents Code (`dcode`)** — a pre-built terminal coding agent built on Deep Agents, with the code interpreter enabled so dynamic subagents work out of the box. Install with `curl -LsSf https://langch.in/dcode | bash`.
* **Deep Agents.js** — a JavaScript/TypeScript SDK (`createDeepAgent`) at github.com/langchain-ai/deepagentsjs.

---

## Protocols and Interoperability

### Agent Client Protocol (ACP)

ACP lets Deep Agents work inside code editors that support the protocol, such as Zed. It exposes your agent over a standard interface that editors can call, and you can use dynamic subagents in the coding agent of your choice over ACP.

### MCP (Model Context Protocol)

Deep Agents fully support MCP. Load tools from any MCP server and pass them to `tools=`. Install `langchain[mcp]` and use the adapter:

```python
from deepagents import create_deep_agent
from langchain.mcp import MCPAdapter

async def main():
    config = {"mcpServers": {"my_server": {"url": "http://localhost:8000/mcp"}}}
    async with MCPAdapter(config) as adapter:
        tools = await adapter.list_tools()
        agent = create_deep_agent(model="anthropic:claude-sonnet-5", tools=tools)
        await agent.ainvoke(
            {"messages": [{"role": "user", "content": "Use the MCP server to help me."}]},
            config={"configurable": {"thread_id": "1"}},
        )
```

Deployed Managed Deep Agents also expose an MCP endpoint, so you can call an agent as a tool from Claude Code or another MCP client.

---

## Deep Agents vs. Claude Agent SDK

Both are agent harnesses, but they make different tradeoffs. The table below follows LangChain's own comparison page (drafted April 16, 2026), so treat it as a vendor-authored view and check the Claude Agent SDK docs for the other side.

| Feature               | Deep Agents                                        | Claude Agent SDK                                   |
| --------------------- | -------------------------------------------------- | -------------------------------------------------- |
| **Where agent runs**  | Inside a sandbox, or outside executing commands remotely | Inside a sandbox                             |
| **Execution backend** | Pluggable in the SDK (local, virtual, remote sandbox, custom); Managed Deep Agents uses one managed sandbox | Local filesystem of the sandbox |
| **Model provider**    | Any (Anthropic, OpenAI, Google, 100+ others)       | Claude (Anthropic, Bedrock, Vertex, Azure)         |
| **Per-model tuning**  | Harness profiles (beta)                            | Configure in code at each model call site          |
| **Deployment**        | Default: Managed Deep Agents (`mda deploy`); or self-host a standalone image via `langgraph build` | Self-hosted; you build the server, auth, streaming |
| **Multi-tenancy**     | Built-in: scoped threads, per-user sandboxes, RBAC | Build it yourself                                  |
| **License**           | MIT                                                | MIT (Claude Code itself is proprietary)            |

**Choose Deep Agents if:** you want model and infrastructure flexibility, built-in multi-tenant deployment, and the option to run managed or self-hosted with the same agent code.

**Choose Claude Agent SDK if:** you are already invested in the Anthropic ecosystem and want to self-host and build the API, auth and multi-tenant layers yourself.

---

## Comparison: Deep Agents vs. LangChain create_agent vs. LangGraph

|                         | Deep Agents                          | LangChain `create_agent`     | LangGraph                          |
| ----------------------- | ------------------------------------ | ---------------------------- | ---------------------------------- |
| **Level**               | High-level harness                   | Mid-level factory            | Low-level runtime/graph            |
| **Planning**            | Opt-in `write_todos` via `TodoListMiddleware` | Manual              | Manual                             |
| **Filesystem**          | Built-in virtual FS                  | Not included                 | Not included                       |
| **Subagents**           | Built-in `task` tool                 | Manual                       | Manual                             |
| **Context compression** | Automatic (offload + summarize)      | Manual                       | Manual                             |
| **Memory**              | AGENTS.md, built-in                  | Manual                       | Via Store                          |
| **Skills**              | SKILL.md, progressive                | Not included                 | Not included                       |
| **HITL**                | `interrupt_on` parameter             | Manual                       | Manual via interrupt               |
| **Deployment**          | Managed Deep Agents (`mda deploy`) or `langgraph build` | Manual     | `langgraph build`                  |
| **Best for**            | Production agents with complex tasks | Simpler agents; full control | Custom graphs; maximum flexibility |

---

## Common Patterns and Use Cases

### Pattern 1: Research and Report Agent

Use internet search + filesystem for offloading + subagent for deep-dive research.

- Main agent: plans, coordinates, writes final report.
- Research subagent: does multi-search deep research, returns concise summary.

### Pattern 2: Coding Agent with Sandbox

Use a sandbox backend + the `execute` tool for the full software development lifecycle.

- Agent can install packages, run tests, fix code, commit via git.
- Human-in-the-loop on `write_file` and `execute` for sensitive operations.
- Keep secrets out of the sandbox; put authenticated calls in host-side tools.

### Pattern 3: Enterprise Document Processor

Use AGENTS.md for company style guides + skills for domain-specific workflows + StoreBackend for persistence.

- Memory keeps corporate tone and format consistent across sessions.
- Skills load domain expertise (e.g. legal review, financial analysis) only when relevant.

### Pattern 4: Multi-Tenant SaaS Agent

Use CompositeBackend (StateBackend for session, StoreBackend for user memory) + namespace factory for data isolation.

- Each user gets isolated session state.
- User preferences and history persist across conversations in the store.

### Pattern 5: Data Pipeline Agent

Use multiple specialized subagents (collector → analyzer → formatter) with structured output at each step.

- Each subagent returns a validated Pydantic model.
- Main agent orchestrates the pipeline without context bloat.

---

## Troubleshooting

### Subagent not being called

- Make descriptions specific and action-oriented.
- Add to the main agent's system prompt: `"For complex tasks, delegate to your subagents using the task() tool."`

### Context still getting bloated despite subagents

- Instruct subagents to return summaries under 500 words.
- Have subagents use the filesystem for large data: save raw results, return only analysis.

### Wrong subagent selected

- Differentiate descriptions clearly: `"quick-researcher"` vs `"deep-researcher"` with explicit criteria for which to use.

### Context window errors

- Override `SummarizationMiddleware` with a lower `trigger` so compaction happens earlier.
- Use a model with a larger context window.
- Add `create_summarization_tool_middleware` so the agent can compact between tasks.
- Remove unused built-in tools with a harness profile's `excluded_tools` to shrink the baseline prompt.

### Mutation bugs in middleware

- Never mutate `self` (or other shared state) in middleware hooks.
- Use graph state or external stores for counters and shared values.

### `write_todos` missing

- Since v0.7, task planning is opt-in. Add `TodoListMiddleware()` to `middleware=`.

---

## Version Requirements

| Feature | Minimum `deepagents` |
| - | - |
| Filesystem permissions | 0.5.2 |
| Subagent `response_format` (structured output) | 0.5.3 |
| Custom state schema (`state_schema`) | 0.6.6 |
| `interrupt` permission mode | 0.6.8 |
| `delete` tool, opt-in `TodoListMiddleware`, middleware override by name, `FilesystemMiddleware(tools=...)` allowlist | 0.7 |
| Exact-match `delete` permission evaluation for plain files | 0.7.3 |
| Forked subagents (beta) | 0.7.13 |
| Interpreter (`CodeInterpreterMiddleware`) | `langchain-quickjs>=0.2.0`, Python 3.11+ |

---

## Summary of Key APIs

```python
# Create a deep agent
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    tools=[my_tool],
    system_prompt="...",
    subagents=[...],
    skills=["/skills/"],
    memory=["/AGENTS.md"],
    permissions=[...],
    response_format=MySchema,
    backend=CompositeBackend(...),
    interrupt_on={"edit_file": True},
    checkpointer=MemorySaver(),
    middleware=[TodoListMiddleware(), my_middleware],
)

# Invoke (blocking)
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})

# Async invoke
result = await agent.ainvoke({"messages": [...]})

# Streaming
for chunk in agent.stream({"messages": [...]}, stream_mode="updates"):
    print(chunk)

# With thread persistence
result = agent.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "my-thread"}},
)
```

---

## Further Reading

- [Deep Agents Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [Deep Agents GitHub](https://github.com/langchain-ai/deepagents)
- [Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview)
- [LangSmith Docs](https://docs.langchain.com/langsmith)
- [LangGraph Docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [Agent Skills Standard](https://agentskills.io/)
- [AGENTS.md Standard](https://agents.md/)
- [Examples Repo](https://github.com/langchain-ai/deepagents/tree/main/examples)
- [API Reference](https://reference.langchain.com/python/deepagents)