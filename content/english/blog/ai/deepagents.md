+++
date = '2026-05-16T12:00:00+10:00'
draft = false
title = 'LangChain Deep Agents'
tags = ['LLM', 'AI', 'LangChain', 'Deep Agents', 'Agentic AI', 'Design Patterns']
summary = "Comprehensive reference for building long-running, multi-step AI agents using LangChain's Deep Agents SDK — covering planning, file systems, subagents, memory, skills, sandboxes, backends, context management, human-in-the-loop, and deployment."
+++

Deep Agents is LangChain's opinionated SDK for building agents that can plan, delegate, remember and act across complex multi-step tasks. It sits on top of LangChain core and the LangGraph runtime and ships with built-in tools for task planning, file management, subagent spawning, context compression and long-term memory.

---

![Class diagram of the LangChain Deep Agents SDK: the create_deep_agent factory, the middleware stack, subagents, filesystem backends and sandbox protocols, result types, harness profiles and filesystem permissions](../img/png/deepagents-class-diagram.png)

## Introduction

Before the Deep Agents SDK, building a production-grade agent required you to wire together context management, tool routing, summarization, filesystem access, human approvals and deployment yourself. Deep Agents packages all of that into a single harness so you can focus on what the agent does, not how it works.

The SDK is available at [pypi.org/project/deepagents](https://pypi.org/project/deepagents/) and the source is at [github.com/langchain-ai/deepagents](https://github.com/langchain-ai/deepagents).

---

## Concepts

### What is an Agent Harness?

- An **agent harness** is the scaffolding that wraps the core LLM tool-calling loop with production-grade capabilities.
- Deep Agents is an opinionated harness — it ships with defaults that work well out of the box (system prompts, planning, file tools, subagents, summarization) and lets you override each layer.
- Contrast with a **framework** (LangChain: building blocks) or a **runtime** (LangGraph: durable graph execution).

### The Tool-Calling Loop

- At its core, every agent is a loop: LLM generates a message → tools are called → results are appended → LLM generates again.
- Deep Agents uses LangGraph's compiled state graph as the runtime engine for this loop, giving it durable execution, checkpointing, and streaming for free.
- The harness adds built-in tools, context management, and middleware hooks on top of this core loop.

### Context Window Management

- LLMs have a finite context window — everything (system prompt, conversation history, tool results) must fit inside it.
- **Context bloat** is the primary failure mode of long-running agents: tool results accumulate and eventually exceed the window.
- Deep Agents addresses this with two main strategies: **offloading** (writing large content to disk) and **summarization** (compressing old history). Both are automatic.

### Virtual Filesystem

- Instead of working directly with disk, Deep Agents gives agents a **virtual filesystem** backed by a pluggable backend.
- Agents write, read, search and edit files as part of their normal work — this is how they offload large tool results and share state between tasks.
- The same agent code works whether the backend is in-memory state, local disk, a cloud sandbox, or a custom store.

---

## Core Architecture

```
create_deep_agent(
    model,              # LLM provider and model
    tools,              # Your custom tools
    system_prompt,      # Your role/behavior instructions
    middleware,         # Cross-cutting hooks (summarization, caching, HITL, etc.)
    subagents,          # Specialized child agents for task delegation
    skills,             # SKILL.md files for progressive domain expertise
    memory,             # AGENTS.md files for persistent context
    response_format,    # Structured output schema (Pydantic)
    backend,            # Virtual filesystem backend
    interrupt_on,       # Human-in-the-loop configuration per tool
)
→ CompiledStateGraph    # LangGraph compiled graph; call .invoke() or .ainvoke()
```

---

## Installation and Setup

```bash
# Core install
pip install deepagents

# With a model provider
pip install deepagents langchain-anthropic   # Anthropic Claude
pip install deepagents langchain-openai      # OpenAI GPT
pip install deepagents langchain-google-genai  # Google Gemini

# With a search tool (for the quickstart)
pip install deepagents tavily-python
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
    model="anthropic:claude-sonnet-4-6",
    tools=[internet_search],
    system_prompt=research_instructions,
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What is LangGraph?"}]
})

print(result["messages"][-1].content)
```

**What happens automatically:**

1. Agent calls `write_todos` to break the task into steps.
2. Calls `internet_search` to gather information.
3. Calls `write_file` / `read_file` to offload large search results.
4. Spawns a subagent if a subtask needs deep context isolation.
5. Synthesizes and returns a final report.

---

## Supported Models

Deep Agents uses a `provider:model` format string. Any LangChain chat model that supports tool calling works.

| Provider     | Example Model String                                                | Notes                      |
| ------------ | ------------------------------------------------------------------- | -------------------------- |
| Anthropic    | `anthropic:claude-sonnet-4-6`                                       | Default; best tested       |
| OpenAI       | `openai:gpt-5.4`                                                    | Strong for numerical tasks |
| Google       | `google_genai:gemini-3.1-pro-preview`                               | Large context window       |
| Azure OpenAI | `azure_openai:gpt-5.4`                                              | Enterprise Azure           |
| AWS Bedrock  | `anthropic.claude-sonnet-4-6` + `model_provider="bedrock_converse"` | AWS-managed                |
| HuggingFace  | `microsoft/Phi-3-mini-4k-instruct` + `model_provider="huggingface"` | Open source                |
| OpenRouter   | `openrouter:anthropic/claude-sonnet-4-6`                            | Proxy across providers     |
| Fireworks    | `fireworks:accounts/fireworks/models/qwen3p5-397b-a17b`             | Fast inference             |
| Baseten      | `baseten:zai-org/GLM-5`                                             | Custom deployments         |
| Ollama       | `ollama:devstral-2`                                                 | Local models               |

You can also pass an initialized model instance directly:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    model="claude-sonnet-4-6",
    max_retries=10,   # Increase for unreliable networks (default: 6)
    timeout=120,      # Increase for slow connections
)
agent = create_deep_agent(model=model, ...)
```

**Connection resilience:** LangChain models automatically retry with exponential backoff. Default is 6 retries for network errors, rate limits (429), and server errors (5xx). Client errors (401, 404) are not retried.

---

## Built-in Tools (Harness Capabilities)

### Planning — `write_todos`

- Gives the agent a structured task list it can update during execution.
- Tasks have statuses: `pending`, `in_progress`, `completed`.
- Persisted in agent state across the conversation.
- Helps agents decompose complex tasks and track progress without holding everything in the prompt.

### Virtual Filesystem Tools

| Tool         | Description                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| `ls`         | List files in a directory with size and modified time                                                          |
| `read_file`  | Read file contents with line numbers; supports offset/limit for large files; reads images as multimodal blocks |
| `write_file` | Create new files                                                                                               |
| `edit_file`  | Exact string replacement in files (with global replace mode)                                                   |
| `glob`       | Find files matching patterns (e.g. `**/*.py`)                                                                  |
| `grep`       | Search file contents — files-only, content-with-context, or count modes                                        |
| `execute`    | Run shell commands (only available with sandbox backends)                                                      |

### Task Delegation — `task`

- The main agent's tool for spawning subagents.
- Accepts a subagent name and task description.
- Blocks (synchronously) until the subagent finishes, then returns a final report.
- All subagent intermediate tool calls are invisible to the main agent — only the result comes back.

---

## Context Management in Depth

### Prompt Composition Order

When you create a deep agent, the final system prompt is assembled in this order:

1. Your custom `system_prompt` (prepended)
2. Base agent prompt (planning, filesystem, general guidance)
3. To-do list instructions
4. Memory prompt (AGENTS.md + usage guidelines, if `memory` provided)
5. Skills prompt (skill locations + frontmatter, if `skills` provided)
6. Virtual filesystem tool docs
7. Subagent `task` tool usage
8. Custom middleware prompts (if any)
9. Human-in-the-loop instructions (if `interrupt_on` set)
10. Local context (current directory, project info — CLI only)

### Offloading Large Content

Deep Agents automatically offloads large content in two scenarios:

**Large tool inputs (>20,000 tokens):**

- File write/edit operations leave the full file content in conversation history.
- Once the session context crosses 85% of the model's context window, Deep Agents truncates older tool calls and replaces them with a file path pointer.
- The file is already on disk — truncating the tool call loses nothing.
- Threshold is configurable via `tool_token_limit_before_evict`.

**Large tool results (>20,000 tokens):**

- When a tool returns more than 20,000 tokens, Deep Agents writes the result to the backend and substitutes a reference + a preview of the first 10 lines.
- The agent can re-read or search the file as needed.
- Threshold is also configurable via `tool_token_limit_before_evict`.

### Summarization

When the context crosses the 85% threshold and there is no more content eligible for offloading, Deep Agents summarizes the message history:

- **In-context summary:** An LLM generates a structured summary (session intent, artifacts created, next steps) that replaces the full conversation history.
- **Filesystem preservation:** The complete original messages are written to the backend as a canonical record.
- This means the agent stays aware of its goals (summary) but can recover details if needed (filesystem search).

**Summarization triggers and defaults:**

```
Trigger:      85% of model's max_input_tokens (from model profile)
Context kept: 10% of tokens as recent context
Fallback:     170,000-token trigger, 6 messages kept (if model profile unavailable)
```

You can also trigger summarization manually between tasks using `create_summarization_tool_middleware`, instead of waiting for the fixed token threshold.

---

## Filesystem Backends

The virtual filesystem is powered by pluggable backends. Choose based on your persistence and execution needs.

### StateBackend (Default)

- Stores files in LangGraph state.
- Ephemeral — persists only within a single thread.
- Good for development and stateless agents.

```python
from deepagents.backends import StateBackend
agent = create_deep_agent(model="openai:gpt-5.4", backend=StateBackend())
```

### FilesystemBackend (Local Disk)

- Maps the virtual filesystem to real local disk.
- Agents can read and write files directly on your machine.
- Use with caution — agents can modify real files.

```python
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    model="openai:gpt-5.4",
    backend=FilesystemBackend(root_dir=".", virtual_mode=True)
)
```

### LocalShellBackend (Local Shell)

- Like FilesystemBackend, but adds the `execute` tool for running shell commands.
- Agents can install packages, run tests, use git, etc.
- **Unrestricted shell access** — only use in safe, isolated environments.

```python
from deepagents.backends import LocalShellBackend
agent = create_deep_agent(
    model="openai:gpt-5.4",
    backend=LocalShellBackend(root_dir=".", env={"PATH": "/usr/bin:/bin"})
)
```

### StoreBackend (Cross-thread Persistence)

- Backed by a LangGraph Store — files persist across threads and conversations.
- Use for long-term memory and knowledge accumulation.
- When deploying to LangSmith, omit the `store` parameter — the platform provisions one automatically.

```python
from langgraph.store.memory import InMemoryStore
from deepagents.backends import StoreBackend

agent = create_deep_agent(
    model="openai:gpt-5.4",
    backend=StoreBackend(
        namespace=lambda ctx: (ctx.runtime.context.user_id,),
    ),
    store=InMemoryStore()
)
```

**Important:** The `namespace` parameter controls data isolation. In multi-user deployments, always set a namespace factory to isolate data per user or tenant.

### CompositeBackend (Hybrid Routing)

- Routes different virtual paths to different backends.
- Example: session-scoped state for most files, durable store for `/memories/`.
- Enables long-term memory alongside ephemeral working files.

```python
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    model="openai:gpt-5.4",
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": StoreBackend()}
    ),
    store=InMemoryStore()
)
```

### Sandbox Backends (Isolated Code Execution)

Sandboxes give agents an `execute` tool inside an isolated environment. Agents can install packages, run tests and execute scripts without touching your host system.

| Sandbox   | Package              | Notes                       |
| --------- | -------------------- | --------------------------- |
| Modal     | `langchain-modal`    | Cloud-based containers      |
| Runloop   | `langchain-runloop`  | Dev environments (devboxes) |
| Daytona   | `langchain-daytona`  | Cloud dev environments      |
| LangSmith | `langsmith[sandbox]` | Private beta                |

```python
# Example: Modal Sandbox
import modal
from langchain_modal import ModalSandbox
from deepagents import create_deep_agent

app = modal.App.lookup("your-app")
modal_sandbox = modal.Sandbox.create(app=app)

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
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

---

## Subagents

Subagents solve the **context bloat problem** — when a subtask requires many tool calls, those results would otherwise fill the main agent's context. Subagents isolate that work; the main agent receives only the final summary.

### When to Use Subagents

| Use                                             | Avoid                                       |
| ----------------------------------------------- | ------------------------------------------- |
| Multi-step tasks that clutter main context      | Simple, single-step tasks                   |
| Specialized domains needing custom instructions | When intermediate context must be preserved |
| Tasks requiring a different model               | When overhead outweighs benefits            |
| Context isolation for parallel workstreams      |                                             |

### Default General-Purpose Subagent

Every deep agent automatically gets a `general-purpose` subagent unless disabled. It has the same system prompt, tools and model as the main agent. The main agent can delegate any task to it without you defining it.

To replace it, provide a subagent with `name="general-purpose"`. To disable it entirely, set `general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False)` on the active harness profile.

### Dictionary-Based Subagents (Most Common)

```python
research_subagent = {
    "name": "research-agent",              # Required; used in task() calls
    "description": "Conducts in-depth research using web search and synthesizes findings",  # Required
    "system_prompt": "You are a thorough researcher...",  # Required; does NOT inherit from main
    "tools": [internet_search],            # Optional; inherits from main if omitted
    "model": "openai:gpt-5.4",            # Optional; inherits from main if omitted
    "middleware": [],                      # Optional; does NOT inherit from main
    "interrupt_on": {},                    # Optional; inherits from main if omitted
    "skills": ["/skills/research/"],      # Optional; does NOT inherit (except general-purpose)
    "response_format": ResearchFindings,  # Optional; Pydantic model for structured output
    "permissions": [],                     # Optional; replaces parent permissions entirely
}

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[research_subagent]
)
```

### CompiledSubAgent (Custom LangGraph Graphs)

For complex workflows, wrap any compiled LangGraph graph as a subagent:

```python
from deepagents import create_deep_agent, CompiledSubAgent
from langchain.agents import create_agent

custom_graph = create_agent(
    model=your_model,
    tools=specialized_tools,
    prompt="You are a specialized data analysis agent..."
).compile()

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[
        CompiledSubAgent(
            name="data-analyzer",
            description="Specialized agent for complex data analysis tasks",
            runnable=custom_graph
        )
    ]
)
```

### Structured Output from Subagents

Subagents can return structured JSON to the parent instead of free-form text:

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
    model="google_genai:gemini-3.1-pro-preview",
    system_prompt="You coordinate data analysis. Delegate to subagents for specialized work.",
    subagents=subagents
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
    model="google_genai:gemini-3.1-pro-preview",
    subagents=[{"name": "researcher", "tools": [get_user_data], ...}],
    context_schema=Context,
)

# Context flows to the researcher subagent's tools automatically
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Look up my recent activity"}]},
    context=Context(user_id="user-123"),
)
```

---

## Middleware

Middleware intercepts the agent loop to add cross-cutting behaviour — logging, caching, prompt injection, PII detection, or custom hooks.

### Default Middleware (Always Present)

| Middleware                         | Purpose                                                   |
| ---------------------------------- | --------------------------------------------------------- |
| `TodoListMiddleware`               | Provides the `write_todos` planning tool                  |
| `FilesystemMiddleware`             | Provides ls, read_file, write_file, edit_file, glob, grep |
| `SubAgentMiddleware`               | Provides the `task` delegation tool                       |
| `SummarizationMiddleware`          | Condenses history when context grows long                 |
| `AnthropicPromptCachingMiddleware` | Reduces redundant token processing for Anthropic models   |
| `PatchToolCallsMiddleware`         | Fixes message history when tool calls are interrupted     |

### Conditional Middleware (Added When Feature is Used)

| Middleware                 | Activated When                      |
| -------------------------- | ----------------------------------- |
| `MemoryMiddleware`         | `memory` argument is provided       |
| `SkillsMiddleware`         | `skills` argument is provided       |
| `HumanInTheLoopMiddleware` | `interrupt_on` argument is provided |

### Custom Middleware

```python
from langchain.agents.middleware import wrap_tool_call
from deepagents import create_deep_agent

call_count = [0]  # Use list, not attribute mutation, to avoid race conditions

@wrap_tool_call
def log_tool_calls(request, handler):
    """Log every tool call."""
    call_count[0] += 1
    print(f"[Tool #{call_count[0]}] {request.name}: {request.args}")
    result = handler(request)
    print(f"[Tool #{call_count[0]}] completed")
    return result

agent = create_deep_agent(
    model="openai:gpt-5.4",
    tools=[my_tool],
    middleware=[log_tool_calls],
)
```

**Critical warning:** Do NOT mutate `self` attributes in middleware hooks — it causes race conditions when subagents, parallel tools, or concurrent threads run simultaneously. Use graph state instead.

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

Memory files are always loaded into the agent's context at startup. They provide persistent, session-independent background that the agent uses to stay consistent across conversations.

- Uses the [AGENTS.md standard](https://agents.md/).
- Typically contains: coding style, project conventions, user preferences, domain guidelines.
- Agents can update their memory based on interactions and feedback.
- Unlike skills, memory is always loaded (not progressive disclosure).

```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langgraph.checkpoint.memory import MemorySaver

with open("./AGENTS.md") as f:
    agents_md = f.read()

agent = create_deep_agent(
    model="openai:gpt-5.4",
    memory=["/AGENTS.md"],
    checkpointer=MemorySaver(),
)

result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "What are my coding preferences?"}],
        "files": {"/AGENTS.md": create_file_data(agents_md)},
    },
    config={"configurable": {"thread_id": "thread-1"}},
)
```

### Skills (SKILL.md Files)

Skills are directories containing a `SKILL.md` file with instructions and metadata. They follow the [Agent Skills standard](https://agentskills.io/).

- Skills use **progressive disclosure** — only the frontmatter of each skill is loaded at startup; the full content is loaded only when the agent determines it's relevant.
- This reduces token usage: skills don't pollute the context unless they're needed.
- A skill directory can include additional scripts, templates, reference docs, and other assets.
- Custom subagents do NOT inherit skills from the main agent by default — only the general-purpose subagent inherits them.

```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data

skill_content = open("./skills/langgraph-docs/SKILL.md").read()

agent = create_deep_agent(
    model="openai:gpt-5.4",
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
| Agent can update     | Yes                                              | Typically read-only                |
| Subagent inheritance | Only general-purpose                             | Only general-purpose               |

---

## Human-in-the-Loop

Human-in-the-loop (HITL) pauses agent execution before specific tool calls and waits for human approval, modification, or rejection. It uses LangGraph's interrupt mechanism.

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
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email."""
    return f"Sent email to {to}"

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[delete_file, send_email],
    interrupt_on={
        "delete_file": True,   # Allow: approve, edit, reject
        "send_email": {"allowed_decisions": ["approve", "reject"]},  # No editing
    },
    checkpointer=MemorySaver()  # Required!
)
```

**HITL decision options:**

- `approve`: Execute the tool as-is.
- `edit`: Modify the tool's input arguments before execution.
- `reject`: Skip the tool call entirely.

**Use cases:**

- Safety gates for destructive operations (delete, overwrite).
- User verification before expensive API calls (email, payment).
- Compliance workflows requiring human sign-off.
- Interactive debugging — inspect and guide agent reasoning.

HITL can be configured for both the main agent and individual subagents. Subagent config overrides the main agent's config for that subagent.

---

## Filesystem Permissions

Declare permission rules to control which files and directories agents can read or write. Subagents can inherit or override parent permissions.

```python
from deepagents import create_deep_agent
from deepagents.permissions import FilesystemPermission

agent = create_deep_agent(
    model="openai:gpt-5.4",
    permissions=[
        FilesystemPermission(path="/data/", read=True, write=True),
        FilesystemPermission(path="/config/", read=True, write=False),
        FilesystemPermission(path="/secrets/", read=False, write=False),
    ]
)
```

When specified on a subagent, the subagent's permissions **replace** the parent's entirely — they are not merged or additive.

---

## Structured Output

Agents can return structured, validated data instead of free-form text using Pydantic models as the `response_format`:

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
    model="openai:gpt-5.4",
    response_format=WeatherReport,
    tools=[internet_search]
)

result = agent.invoke({"messages": [
    {"role": "user", "content": "What's the weather like in Sydney?"}
]})

# Structured output in result["structured_response"]
report = result["structured_response"]
print(report.temperature, report.condition)
```

The structured response is validated, type-safe, and returned in the `structured_response` key of the agent state.

---

## Streaming

Deep Agents support real-time streaming using LangGraph's streaming interface. This lets you observe output progressively — tool calls, tool results, LLM responses — as the agent runs.

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Research quantum computing"}]},
    stream_mode="updates",
):
    print(chunk)
```

When using subagents, each agent's output is tagged with `lc_agent_name` in the metadata, so you can differentiate which agent produced which chunk:

```python
for event in agent.stream(..., stream_mode="events"):
    agent_name = event.get("metadata", {}).get("lc_agent_name", "main")
    print(f"[{agent_name}] {event}")
```

---

## System Prompt Composition

Each deep agent should have a custom system prompt tailored to its use case. The custom prompt is **prepended** to the built-in harness prompt — you layer on top of the defaults.

```python
agent = create_deep_agent(
    model="openai:gpt-5.4",
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

**What the built-in prompt covers:** how to use the planning tool, filesystem tools, subagents, context management and output formatting. You don't need to re-explain these — just describe your agent's role and domain.

---

## Deployment

### CLI Deployment

```bash
deepagents deploy
```

Deploys to LangSmith managed cloud. The agent server, streaming endpoints, thread management, run history, webhooks and authentication are all included.

### Self-Hosted (Docker)

```bash
langgraph build   # Produces a standalone Docker image
```

Deploy the image anywhere — your own infrastructure, Kubernetes, cloud VMs.

**Both modes run the same code without changes.** You pick managed vs. self-hosted at deploy time, not at build time.

### Tracing and Observability

Connect to [LangSmith](https://smith.langchain.com) for tracing, debugging and evaluation:

```bash
export LANGSMITH_API_KEY="your-key"
export LANGSMITH_TRACING=true
```

The LangSmith Engine detects issues in agent traces and proposes fixes. You can open a pull request with the fix directly from the Issues tab.

---

## Protocols and Interoperability

### Agent Client Protocol (ACP)

ACP is a connector that lets Deep Agents work inside code editors like Zed, VSCode and others that support the protocol. It exposes your agent over a standard interface that editors can call.

### MCP (Model Context Protocol)

Deep Agents integrates with MCP servers for tool access. Pass MCP tool sets alongside your regular tools to give agents access to third-party services.

### A2A (Agent-to-Agent)

A2A enables communication between different agent systems. Via LangSmith, deep agents can interoperate with agents built on other frameworks.

---

## Deep Agents vs. Claude Agent SDK

Both are agent harnesses, but they make different tradeoffs.

| Feature               | Deep Agents                                        | Claude Agent SDK                                   |
| --------------------- | -------------------------------------------------- | -------------------------------------------------- |
| **Where agent runs**  | Inside sandbox, or outside using sandbox as a tool | Inside the sandbox                                 |
| **Execution backend** | Pluggable (local, virtual, sandbox, custom)        | Local filesystem of sandbox                        |
| **Model provider**    | Any (Anthropic, OpenAI, Google, 100+ others)       | Claude only (Anthropic, Bedrock, Vertex, Azure)    |
| **Deployment**        | LangSmith managed cloud or self-hosted Docker      | Self-hosted; you build the server, auth, streaming |
| **Multi-tenancy**     | Built-in: scoped threads, per-user sandboxes, RBAC | Build it yourself                                  |
| **Agent server**      | Included (streaming, thread mgmt, history, auth)   | Build it yourself                                  |
| **License**           | MIT                                                | MIT (Claude Code itself is proprietary)            |

**Choose Deep Agents if:** you want model and infrastructure flexibility, built-in multi-tenant deployment, and the option to run managed or self-hosted without code changes.

**Choose Claude Agent SDK if:** you're committed to Anthropic's Claude and want to self-host and build the server and auth layers yourself.

---

## Comparison: Deep Agents vs. LangChain create_agent vs. LangGraph

|                         | Deep Agents                          | LangChain `create_agent`     | LangGraph                          |
| ----------------------- | ------------------------------------ | ---------------------------- | ---------------------------------- |
| **Level**               | High-level harness                   | Mid-level factory            | Low-level runtime/graph            |
| **Planning**            | Built-in `write_todos`               | Manual                       | Manual                             |
| **Filesystem**          | Built-in virtual FS                  | Not included                 | Not included                       |
| **Subagents**           | Built-in `task` tool                 | Manual                       | Manual                             |
| **Context compression** | Automatic (offload + summarize)      | Manual                       | Manual                             |
| **Memory**              | AGENTS.md, built-in                  | Manual                       | Via Store                          |
| **Skills**              | SKILL.md, progressive                | Not included                 | Not included                       |
| **HITL**                | `interrupt_on` parameter             | Manual                       | Manual via interrupt               |
| **Deployment**          | `deepagents deploy`                  | Manual                       | `langgraph build`                  |
| **Best for**            | Production agents with complex tasks | Simpler agents; full control | Custom graphs; maximum flexibility |

---

## Common Patterns and Use Cases

### Pattern 1: Research and Report Agent

Use internet search + filesystem for offloading + subagent for deep-dive research.

- Main agent: plans, coordinates, writes final report.
- Research subagent: does multi-search deep research, returns concise summary.

### Pattern 2: Coding Agent with Sandbox

Use sandbox backend + execute tool for full software development lifecycle.

- Agent can install packages, run tests, fix code, commit via git.
- Human-in-the-loop on `write_file` and `execute` for sensitive operations.

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
- Add to the main agent's system prompt: `"For complex tasks, always delegate to your subagents using the task() tool."`

### Context still getting bloated despite subagents

- Instruct subagents to return summaries under 500 words.
- Have subagents use the filesystem for large data: save raw results, return only analysis.

### Wrong subagent selected

- Differentiate descriptions clearly: `"quick-researcher"` vs `"deep-researcher"` with explicit criteria for which to use.

### Context window errors

- Lower `tool_token_limit_before_evict` to trigger offloading earlier.
- Use a model with a larger context window (e.g. Gemini 1.5 Pro).
- Add `create_summarization_tool_middleware` for explicit summarization between tasks.

### Mutation bugs in middleware

- Never mutate `self` in middleware hooks.
- Use graph state or external stores for counters and shared values.

---

## Summary of Key APIs

```python
# Create a deep agent
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[my_tool],
    system_prompt="...",
    subagents=[...],
    skills=["/skills/"],
    memory=["/AGENTS.md"],
    response_format=MySchema,
    backend=CompositeBackend(...),
    interrupt_on={"edit_file": True},
    checkpointer=MemorySaver(),
    middleware=[my_middleware],
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

- [Deep Agents GitHub](https://github.com/langchain-ai/deepagents)
- [LangSmith Docs](https://docs.langchain.com/langsmith)
- [LangGraph Docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [Agent Skills Standard](https://agentskills.io/)
- [AGENTS.md Standard](https://agents.md/)
- [Examples Repo](https://github.com/langchain-ai/deepagents/tree/main/examples)
- [API Reference](https://reference.langchain.com/python/deepagents)
