+++
date = '2026-11-07T12:00:00+10:00'
draft = false
title = 'LangChain Deep Agents'
tags = ['LLM', 'AI', 'LangChain', 'Deep Agents', 'Agentic AI', 'Design Patterns']
summary = "Comprehensive reference for building long-running, multi-step AI agents using LangChain's Deep Agents SDK — covering planning, file systems, subagents, memory, skills, sandboxes, backends, context management, human-in-the-loop, and deployment."
+++

Deep Agents is a LangChain library for building agents that handle tasks with several steps. It combines LangChain components with the LangGraph runtime. It provides tools for working with files, delegating tasks to subagents and managing long conversations. Planning tools are optional. Persistent memory requires a suitable backend and configuration.

---

![Class diagram of the LangChain Deep Agents SDK: the create_deep_agent factory, the middleware stack, subagents, filesystem backends and sandbox protocols, result types, harness profiles and filesystem permissions](../img/png/deepagents-class-diagram.png)

## Chapter: The Problem Deep Agents Solves

An agent that handles long tasks needs more than a model and a tool loop. It may also need context management, file access, task delegation, human approval and a deployment environment. Deep Agents provides components for these needs in one agent harness. You still choose and configure the components that your application requires.

You can run the same harness two ways:

- **Deep Agents SDK** (`deepagents` on [PyPI](https://pypi.org/project/deepagents/)) — an open-source package that you run and host. Source: [github.com/langchain-ai/deepagents](https://github.com/langchain-ai/deepagents).
- **Managed Deep Agents** — a LangSmith service that hosts the harness and provides related deployment features. Check the current [Managed Deep Agents documentation](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview) for available features and requirements.

This post focuses on the SDK, with a section on managed deployment near the end.

## How to Read This Book

An **agent harness** adds the components needed to run an agent around a model's tool-calling loop. Deep Agents combines that loop with a LangGraph runtime, file tools, context management, optional planning, subagents and control mechanisms. The runtime runs the agent. The filesystem stores working files. Context management keeps conversations within model limits. Subagents handle delegated tasks. Approval and permission settings let an application control selected actions.

The chapters build that picture in layers:

- **Foundation:** learn what an agent harness does and how Deep Agents is structured.
- **Build an agent:** install the SDK, select a model and follow a research task from input to result.
- **Agent operation:** add tools, manage context and choose where the agent stores files.
- **Coordination and control:** use subagents and middleware, provide reusable knowledge, and configure approvals and filesystem permissions.
- **Deployment and integration:** return structured results, stream progress, deploy the agent and connect it to other systems.
- **Reference:** compare related tools, review use cases, troubleshoot errors, check version requirements and find API examples.

Read the chapters in order to learn the concepts step by step. After that, use the later chapters to look up specific features.

> **Version note.** Several features below depend on a minimum `deepagents` version. See [Version requirements](#version-requirements) at the end of the post and pin your version before copying code.

---

## Chapter: The Agent Harness Mental Model

### What is an Agent Harness?

- An **agent harness** is software that runs an agent's model-and-tool loop and supplies related capabilities, such as file tools and context management.
- Deep Agents is a harness with built-in defaults, including a system prompt, filesystem tools and context summarization. You can configure or extend parts of the harness.
- **LangChain** provides components for building agents. **LangGraph** provides the runtime used to execute and persist graph-based agents. Deep Agents builds on both.

### The Tool-Calling Loop

- A tool-calling agent repeats these steps: the model returns a message, the application runs any requested tools, and the application sends the tool results back to the model.
- Deep Agents runs this loop with a compiled LangGraph state graph. LangGraph provides execution features such as checkpointing and streaming when configured and used through its supported interfaces.
- Deep Agents adds filesystem tools, context management and middleware to the model-and-tool loop.

### Context Window Management

- Each model has a context limit. The system prompt, conversation history and tool results all use part of that limit.
- Long conversations can exceed the model's context limit when tool results and earlier messages accumulate.
- Deep Agents can reduce context use by saving large content to its filesystem and by summarizing older messages. Subagents can also keep the details of a delegated task in a separate context.

### Virtual Filesystem

- Deep Agents provides a **virtual filesystem**. A backend determines where its files are stored and which operations are available.
- The agent can use filesystem tools to read, write, search and edit files. These files can hold work that would otherwise remain in conversation history.
- Depending on the backend, files can be stored in graph state, on local disk, in a sandbox or in a persistent store. Backend choice affects persistence and access.

---

## Chapter: How the Harness Fits Together

`create_deep_agent` creates a Deep Agents agent. The following signature lists its main parameters; check the [API reference](https://reference.langchain.com/python/deepagents) for the signature supported by your installed version.

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

## Chapter: Set Up the SDK

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

## Chapter: Build Your First Research Agent

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

**One possible execution:**

1. The agent can call `write_todos` because the example adds `TodoListMiddleware`. The tool lets the agent record task steps.
2. It calls `internet_search` to gather information.
3. It calls `write_file` / `read_file` to offload large search results.
4. If the agent has a suitable subagent configured, it can delegate a subtask to that agent.
5. The agent combines the available results and returns a report.

---

## Chapter: Choose a Model Provider

Deep Agents can select a model with a `provider:model` string or receive an initialized LangChain chat model. The model must support tool calling. The identifiers below are examples, not a promise that every provider still offers each model. Check the current [Deep Agents model guide](https://docs.langchain.com/oss/python/deepagents/models) and your provider's model list before using one.

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

**Retries:** LangChain chat models retry some network errors, rate limits and server errors with exponential backoff. The retry count and other behavior depend on the integration and its configuration. See [LangChain's connection resilience guide](https://docs.langchain.com/oss/python/langchain/models#connection-resilience) for current details.

---

## Chapter: Give the Agent Its Core Tools

### Planning — `write_todos` (opt-in)

- Adds a structured task list that the agent can update while it works.
- **Opt-in since v0.7:** add `TodoListMiddleware` from `langchain.agents.middleware` to `middleware=`. Earlier versions enabled this middleware by default.
- Each task has one of these statuses: `pending`, `in_progress` or `completed`. The task list is stored in agent state.
- A task list can help with long tasks that have several steps. An application can also display task status to show progress.

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

**Choose which filesystem tools are available:**

- To remove filesystem tools from the model's tool list, register a harness profile with `excluded_tools`. Fewer tool definitions can also reduce the prompt size.
- To give the agent only selected filesystem tools, pass a `tools` allowlist to a `FilesystemMiddleware` instance (`deepagents>=0.7`). The allowlist must include `read_file`.

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

`FilesystemMiddleware` is required by the harness. You cannot remove it with `excluded_middleware`.

### Task Delegation — `task`

- The `task` tool lets the main agent start a configured subagent.
- A task call identifies the subagent and describes the work.
- A synchronous task call waits for the subagent to finish, then returns its result.
- The main agent receives the subagent's result. It does not receive the subagent's intermediate tool calls as messages in its own conversation.
- A synchronous subagent returns one result for each task call. Use async subagents when the main agent needs to check or update work while it is running.

---

## Chapter: Keep Long Tasks Within Context

### Prompt Composition Order

When you create a deep agent, the harness assembles its system prompt from these parts, in this order:

1. Your custom `system_prompt` (prepended)
2. Base agent prompt
3. Memory prompt (`AGENTS.md` + usage guidelines, only when `memory` is provided)
4. Skills prompt (skill locations + frontmatter, only when `skills` is provided)
5. Virtual filesystem prompt (filesystem + `execute` tool docs where applicable)
6. Subagent prompt (`task` tool usage)
7. User-provided middleware prompts (if any)
8. Human-in-the-loop prompt (when `interrupt_on` is set)

Deep Agents Code, the terminal agent, also adds information about the current directory and project.

### Offloading Large Content

Deep Agents can move large tool inputs and results out of the conversation and into the filesystem. The documented default threshold is **20,000 tokens**; confirm the value for the SDK version you use.

**Large tool inputs:**

- File write and edit operations can add the full file content to conversation history.
- When the session reaches 85% of the model's available context window, the middleware can replace older tool-call content with a reference to the saved file.
- The saved file remains available through the configured backend. The conversation history is shortened, but the file content is retained there.

**Large tool results:**

- The middleware writes the result to the configured backend and replaces the in-context result with a file path and a preview of its first 10 lines.
- The agent can use filesystem tools to read or search the saved result.

This compression does not resize images or reduce their resolution. See the [Deep Agents multimodal guide](https://docs.langchain.com/oss/python/deepagents/multimodality) for current details about media handling.

### Summarization

When the context reaches its configured limit (documented default: 85% of the model's `max_input_tokens`) and offloading cannot reduce it enough, Deep Agents can summarize the message history:

- **Summary in context:** a model creates a summary of the session's goal, created files and next steps. The summary replaces older messages in the active context.
- **Original messages in the filesystem:** a text copy of the original messages is saved to the configured backend.
- The agent uses the summary to continue its work. It can search or read the saved messages when it needs earlier details.

**Defaults:**

```
Trigger:      85% of model's max_input_tokens (from the model profile)
Context kept: 10% of tokens as recent context
Fallback:     170,000-token trigger, 6 messages kept (if model profile unavailable)
```

If a model call raises `ContextOverflowError`, the deep agent can summarize the conversation and retry with the summary and the recent messages that remain in context.

**Set different thresholds** — pass a `SummarizationMiddleware` instance to replace the default middleware (`deepagents>=0.7`). The replacement must include the configuration it needs:

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

**Summarize on request** — add a `compact_conversation` tool so the agent can summarize between tasks. Automatic summarization remains enabled:

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

**Identify summary messages in a stream:** messages produced during summarization have `metadata["lc_source"] == "summarization"`. Use this metadata to distinguish them from the agent's other messages.

### Prompt Caching

For supported Anthropic and Amazon Bedrock models, `create_deep_agent` adds prompt-caching middleware for static prompt content such as base instructions, memory and skill content. The provider must support prompt caching. To change the Anthropic cache time-to-live (TTL), configure its middleware:

```python
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-5",
    middleware=[AnthropicPromptCachingMiddleware(ttl="1h")],  # default is 5m
)
```

### Runtime Context and Custom State

- **Runtime context** (`context_schema=` and `context=` at invoke time) carries per-run values, such as a user ID. The model cannot see these values unless a tool or middleware exposes them. Runtime context is passed to subagents.
- **Custom state** (`state_schema=`, `deepagents>=0.6.6`) adds data to the graph state saved by a checkpointer. Subclass `DeepAgentState` to retain the built-in handling of the `messages` field.

---

## Chapter: Choose Where Agent Files Live

The virtual filesystem uses a backend to store files and perform file operations. Choose a backend based on how long files must persist and where the agent must run.

### StateBackend (Default)

- Stores files in the LangGraph state for a thread.
- With a checkpointer, files persist between turns in the same thread. They are not shared with other threads.
- This backend is useful for development and agents that do not need to share files between threads.

```python
from deepagents.backends import StateBackend
agent = create_deep_agent(model="openai:gpt-5.5", backend=StateBackend())
```

### FilesystemBackend (Local Disk)

- Maps virtual paths to files on local disk.
- The agent can read and write files within the configured root directory. Set that directory carefully because the agent may change files there.
- A `CompositeBackend` can route internal agent files, such as offloaded results, to a different backend from your project files.

```python
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=FilesystemBackend(root_dir=".", virtual_mode=True),
)
```

### LocalShellBackend (Local Shell)

- Like `FilesystemBackend`, this backend maps files to local disk and adds an `execute` tool for shell commands on the host.
- Commands run with the permissions of the host process. Use this backend only in an environment where those permissions are appropriate.

```python
from deepagents.backends import LocalShellBackend
agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=LocalShellBackend(root_dir=".", virtual_mode=True, env={"PATH": "/usr/bin:/bin"}),
)
```

### StoreBackend (Cross-thread Persistence)

- Uses a LangGraph Store so files can persist across threads and conversations.
- In a LangSmith deployment, the platform may provide the store. Follow the deployment documentation for the configuration required by your setup.

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

**Data isolation:** the `namespace` parameter determines where the store keeps each set of files. In a multi-user application, configure namespaces so one user or tenant cannot access another user's stored data.

### ContextHubBackend

- Stores filesystem data in a LangSmith Hub repository. Check the current integration documentation for availability and setup requirements.

```python
from deepagents.backends import ContextHubBackend

agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=ContextHubBackend("my-agent"),
)
```

### CompositeBackend (Hybrid Routing)

- Routes different virtual paths to different backends.
- In this example, most files use thread-scoped state and files under `/memories/` use a persistent store.
- This arrangement separates temporary working files from files intended to persist between conversations.

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

Sandbox backends provide an `execute` tool that runs commands in a separate environment. The agent can install packages, run tests and execute scripts there without running those commands directly on the host. A sandbox provider implements `execute()`; the backend base class supplies the other file operations.

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

**Reasons to use a sandbox:**

- **Separate execution environment:** commands run in the sandbox instead of directly on the host. This does not guarantee that the host or other systems are protected from every risk.
- **Separate dependencies:** install the required package versions in the sandbox without changing the host environment.
- **Repeatable setup:** configure the same environment for multiple runs or team members.

**Two integration patterns:**

- **Sandbox as a tool:** the agent runs in your application and uses its backend to send commands to the sandbox. Credentials used by the application can stay outside the sandbox. The application and sandbox have separate deployment lifecycles.
- **Agent inside the sandbox:** the agent process runs in the sandbox, and your application communicates with it over a network connection. Any credentials the agent needs may be present in the sandbox, so configure credential access carefully.

**Security notes:**

- A sandbox does not by itself prevent an agent from being influenced by malicious input or from sending data over an allowed network connection. Limit network access and keep credentials outside the sandbox when possible. If credentials must be available inside it, apply the provider's security controls.
- Stop sandboxes when they are no longer needed. Some providers charge while a sandbox is running. Configure an idle timeout (TTL) if the provider supports one.
- Use `upload_files()` and `download_files()` in application code to transfer files between the application and the sandbox.

---

## Chapter: Delegate Work to Subagents

Subagents perform tasks delegated by a main agent. A subagent can keep its own tool calls and results in a separate context. The main agent receives the subagent's returned result, which can reduce how much detail the main conversation must retain.

### When to Use Subagents

| Use                                             | Avoid                                       |
| ----------------------------------------------- | ------------------------------------------- |
| Consider a subagent for...                       | Avoid a subagent when...                    |
| ----------------------------------------------- | ------------------------------------------- |
| A task with several steps and many tool results | The task is a single, simple operation      |
| Work that needs its own instructions or tools   | The main agent needs every intermediate step |
| Work that needs a different model               | Setup and coordination cost more than the task |
| Work that can run separately from coordination |                                             |

### Default General-Purpose Subagent

The default configuration adds a synchronous `general-purpose` subagent unless you provide a subagent with that name. It has filesystem tools, uses the main agent's model by default and **inherits the main agent's skills**.

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

For a more complex subagent, pass a prebuilt, compiled LangGraph graph. `create_agent` returns a compiled graph, so do not call `.compile()` on its result. A custom graph must include a `"messages"` key in its state.

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

By default, a subagent runs in **isolated** mode and receives the task description. With `mode: "fork"`, it also receives the parent's conversation history and state. Fork mode can help when a subagent must continue work that the parent has already started, such as drafting review comments about issues the parent found.

```python
comment_writer = {
    "name": "comment-writer",
    "description": "Continues an in-progress PR review and drafts review comments",
    "mode": "fork",
    "tools": [read_diff],
}
```

- Fork mode requires `deepagents>=0.7.13` and is a beta feature.
- Fork mode does not accept `skills`. Its `system_prompt` is added to the parent's prompt.
- A forked subagent cannot call `task` to delegate another task. It must complete its assigned work itself.
- A fork may reuse cached prompt content when its prompt begins with the same content as the parent's prompt. Changes to the shared prompt or tools can prevent that reuse.

### Structured Results from Subagents

Subagents can return JSON that follows a schema instead of an unstructured text result (`deepagents>=0.5.3`):

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

The parent receives the serialized result in the `ToolMessage` content.

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

**Workflow:** the main agent plans the work, then delegates data collection, analysis and report writing to the configured subagents. It combines their results into the final output. Each subagent receives the task assigned to it and its configured context.

### Subagent Best Practices

**Give each subagent a clear description.** The main agent uses these descriptions when choosing a subagent:

```
✅  "Analyzes financial data and generates investment insights with confidence scores"
❌  "Does finance stuff"
```

**Give each subagent specific instructions, including the required result format:**

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

**Give each subagent only the tools it needs:**

```python
# ✅ Focused
{"tools": [send_email, validate_email]}

# ❌ Unfocused
{"tools": [send_email, web_search, database_query, file_upload]}
```

**Ask subagents for concise results.** They should summarize findings instead of returning all raw data:

```python
system_prompt="""Return:
- Key insights (3-5 bullets)
- Confidence score
- Recommended next actions
Do NOT include raw data or intermediate tool outputs. Under 300 words."""
```

### Context Propagation to Subagents

Runtime context supplied to `.invoke()` is also available to subagents:

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

All subagents receive the same runtime context. To give each subagent different settings, use separate namespaced fields, such as `fact_checker_strict_mode`. A tool can also inspect `runtime.config["metadata"]["lc_agent_name"]` to identify the calling agent.

---

## Chapter: Run Subagents Asynchronously

Deep Agents also supports **asynchronous subagents** (`AsyncSubAgent`). They are designed for long-running or parallel tasks. A synchronous subagent makes the main agent wait for one result. An asynchronous subagent provides tools to start, check, update and cancel work while it runs.

Async subagents use `AsyncSubAgentMiddleware`, which is separate from the middleware for synchronous `task` calls. An agent can be configured to use both types.

| | Synchronous (`task`) | Async (`AsyncSubAgent`) |
|---|---|---|
| Execution | Blocks until complete | Runs in parallel |
| Steering | None mid-flight | Launch, check, update, cancel |
| Best for | Short, focused subtasks | Long-running work, parallel batches |

---

## Chapter: Create Specialized Agents at Runtime

An **interpreter** adds an `eval` tool that runs JavaScript in a restricted QuickJS runtime. The agent can use code to call tools in loops, process batches or transform data. The interpreter does not provide shell commands, package installation, filesystem access or network access. Use a sandbox backend when the agent needs those capabilities.

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

**Dynamic subagents (beta):** when both subagents and interpreter middleware are configured, the agent can start subagents from interpreter code. This supports work over multiple items or batches. Dynamic subagents are enabled by default in this configuration. Set `CodeInterpreterMiddleware(subagents=False)` to disable them and require the regular `task` tool. The built-in interpreter instructions use the word "workflow" as a signal that code-based orchestration may be appropriate.

---

## Chapter: Extend the Agent Pipeline

Middleware runs at defined points in the agent loop. It can add shared behavior such as logging, caching, prompt changes, personal information detection or custom hooks.

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

Deep Agents combines the middleware you provide with its built-in middleware. In `deepagents>=0.7`, an instance with the same `.name` as built-in middleware replaces that middleware. Other middleware is added after `PatchToolCallsMiddleware` and before profile extras, skills, prompt caching and memory middleware.

Declarative subagents do **not** inherit custom middleware from the main agent. Set middleware for a declarative subagent in that subagent's `middleware` field. The general-purpose subagent does inherit replacements for default middleware.

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

**Concurrent execution:** Do not change `self` attributes or other shared values from middleware hooks. Subagents, parallel tools or concurrent threads could access the same value at the same time and cause inconsistent results. The counter above is only a single-run example. For application state, use graph state or another storage mechanism that supports your concurrency requirements.

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

## Chapter: Give the Agent Reusable Knowledge

### Memory (AGENTS.md Files)

When configured, memory files are added to the agent's context at startup. They can provide background information across sessions, such as project conventions or user preferences.

- Uses the [AGENTS.md standard](https://agents.md/).
- A memory file can contain coding style, project conventions, user preferences or domain guidance.
- Memory files are read from the configured backend. The agent can update them if its tools and permissions allow it.
- Configured memory content is included at startup. Unlike skills, it is not loaded only when relevant, so keep the files concise.
- Add the configured memory files to the backend before invoking the agent. The example seeds `/AGENTS.md` in the default `StateBackend` through the `files` state.

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

**Memory shared across threads** requires persistent storage. This example uses a `CompositeBackend` that routes `/memories/` to a `StoreBackend` (see [Backends](#compositebackend-hybrid-routing)). The `/memories/` path is initially empty. Tell the agent what information it may save there; it can then create files using its filesystem tools. Use `permissions` to prevent the agent from changing memory files when only application code should update them.

### Skills (SKILL.md Files)

Each skill is a directory containing a `SKILL.md` file with instructions and metadata. Deep Agents follows the [Agent Skills standard](https://agentskills.io/).

- Skills use **progressive disclosure**: the agent receives the skill metadata first and loads the full instructions when the skill is relevant.
- A skill directory can also contain scripts, templates, reference documents and other files.
- Custom subagents do **not** inherit the main agent's skills. Set a subagent's `skills` field to give it skills. The general-purpose subagent inherits skills from the main agent.
- Keep each skill focused on one task or subject. Put detailed reference information in separate files when appropriate.

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

## Chapter: Put Human Approval at the Right Boundaries

Human-in-the-loop (HITL) pauses the agent before selected tool calls and waits for a human decision. Deep Agents uses LangGraph interrupts to pause and resume execution.

**A checkpointer is required** to save the agent's state while execution is paused. `MemorySaver` is used in this example.

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

- `approve`: run the tool with its current arguments.
- `edit`: change the tool's arguments before it runs.
- `reject`: do not run the tool.
- `respond`: send guidance to the agent without running the tool.

**Use cases:**

- Require approval before destructive operations, such as deleting or overwriting files.
- Ask the user to confirm actions such as sending an email or making a payment.
- Require sign-off for workflows that have an approval requirement.
- Pause execution so a person can inspect the request and provide guidance.

Configure HITL for the main agent or for individual subagents. A subagent's `interrupt_on` setting takes precedence over the inherited setting. Filesystem permission rules can also pause selected filesystem operations.

---

## Chapter: Constrain Filesystem Access

Use permission rules to specify which filesystem paths the agent may read or change.

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

**Rule evaluation:** Deep Agents checks rules in declaration order and applies the first matching rule. If no rule matches, the operation is allowed. Put specific deny rules, such as a rule for `.env` files, before broader allow rules. To allow only selected paths, add a final rule that denies all other paths.

**Mode `interrupt`** (`deepagents>=0.6.8`) pauses supported write operations (`write_file`, `edit_file`, `delete`) for human approval. It requires a checkpointer and works with `interrupt_on`. Start interrupt path patterns with a literal path segment, such as `/secrets/**`. Patterns without a fixed starting path can match more paths than intended.

**Limits and caveats:**

- Filesystem permissions require `deepagents>=0.5.2`.
- These rules apply to the built-in filesystem tools. They do not restrict custom tools or MCP tools that access files. Apply separate controls to those tools or to their backend.
- Filesystem permissions do not restrict shell commands run through a sandbox's `execute` tool.
- If a `CompositeBackend` uses a sandbox as its default backend, scope each permission path under a configured route prefix. Otherwise, Deep Agents raises `NotImplementedError`.
- Deleting a directory checks write permission for the target directory and its contents. Exact path matching when deleting a regular file requires `deepagents>=0.7.3`.

**Subagents** inherit the parent's permissions unless the subagent defines its own `permissions`. A subagent's permission list replaces the parent's list; the two lists are not combined.

---

## Chapter: Return Data Your Application Can Trust

Set `response_format` to a schema when the application needs a structured result instead of free-form text. The agent returns data that can be validated against that schema:

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

## Chapter: Stream Progress and Subagent Activity

Use LangGraph's streaming interface to receive updates while the agent runs:

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "Research quantum computing"}]},
    stream_mode="updates",
):
    print(chunk)
```

### Stream subagents

For structured stream events, use `stream_events`. Deep Agents adds `stream.subagents`: each delegated `task` call has a handle with fields such as `.name`, `.messages`, `.tool_calls` and `.output`. The handle can also expose nested subagent streams:

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

LangSmith run metadata includes the executing agent's name in `lc_agent_name`. Use that field to filter runs by agent in the LangSmith interface or SDK:

```python
from langsmith import Client

client = Client()
runs = client.list_runs(
    project_name="<your-project>",
    filter='has(metadata, \'{"lc_agent_name": "research-agent"}\')',
)
```

---

## Chapter: Shape the Agent’s Instructions

Write a system prompt that describes the agent's role and task. Deep Agents places the custom prompt before the built-in harness prompt.

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

`system_prompt` accepts a string or a `SystemMessage`. Structured content blocks are preserved for the main agent; dictionary-based subagent prompts must be strings. The `system_prompt` value does not change between runs. To create a prompt from per-run values, use `@dynamic_prompt` middleware and read `request.runtime.context` or `request.runtime.store`.

**Built-in instructions:** Deep Agents already explains how to use its filesystem tools, subagents, context management and output format. In your custom prompt, describe the agent's role and subject area. See [Prompt Composition Order](#prompt-composition-order) for how the prompts are assembled.

### Harness profiles (beta)

Harness profiles let you configure prompts, tools, middleware and subagents for a provider or model. `base_system_prompt` replaces the base prompt. `system_prompt_suffix` adds text to the end of the assembled prompt. Built-in Anthropic and OpenAI profiles provide a suffix. A profile can also set `excluded_tools` and `tool_description_overrides`.

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

The general-purpose subagent uses the first available base prompt in this order: `general_purpose_subagent.system_prompt`, the profile's `base_system_prompt`, then the SDK default. Deep Agents adds `system_prompt_suffix` after that base prompt.

---

## Chapter: Run the Agent in Production

### Managed Deep Agents (public beta)

Managed Deep Agents (MDA) is a LangSmith service for running and deploying agents. An agent project can contain files such as `agent.py`, `instructions.md`, `skills/`, `tools/` and `middleware/`, plus optional configuration for sandboxes, memory, identity, channels, schedules and evaluations. The `mda` command-line tool can run the project locally or deploy it. Check the [Managed Deep Agents documentation](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview) for current availability and setup.

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

`define_deep_agent` accepts most of the same agent configuration as `create_deep_agent`, plus a required static `name`. With `create_deep_agent`, your application configures and runs the backend, store, checkpointer and server. With `define_deep_agent`, the managed runtime supplies the managed services supported by the deployment. Check the service documentation for the current APIs and features included with a deployment.

### Self-hosted

```bash
langgraph build   # Produces a standalone Docker image
```

Deploy the image on infrastructure you manage, such as Kubernetes or a cloud virtual machine, or configure a LangSmith Deployment. Check each deployment environment for compatibility. Some configuration may need to change when moving between managed and self-hosted deployments.

### Tracing and Observability

Connect to [LangSmith](https://smith.langchain.com) to trace and inspect runs and to evaluate agent behavior:

```bash
export LANGSMITH_API_KEY="your-key"
export LANGSMITH_TRACING=true
```

LangSmith Engine can analyze traces and suggest fixes. See the [LangSmith Engine documentation](https://docs.langchain.com/langsmith) for current capabilities.

### Forms of Deep Agents

* **Deep Agents SDK** (`deepagents` on PyPI) — the Python package described in this article. You run and host it.
* **Managed Deep Agents** — a LangSmith-hosted runtime managed with the `mda` CLI. Check the service documentation for current availability.
* **Deep Agents Code (`dcode`)** — a terminal coding agent built with Deep Agents. Its installation and included features can change; use the [official project instructions](https://langch.in/dcode) before installing it.
* **Deep Agents.js** — a JavaScript and TypeScript SDK with a `createDeepAgent` API. See the [official repository](https://github.com/langchain-ai/deepagentsjs) for current status and usage.

---

## Chapter: Connect Agents to Other Systems

### Agent Client Protocol (ACP)

The Agent Client Protocol (ACP) lets compatible editors, such as Zed, communicate with an agent through a shared interface. Whether a specific Deep Agents setup supports ACP or dynamic subagents depends on its integration and configuration.

### MCP (Model Context Protocol)

You can connect MCP tools to a Deep Agents agent through a LangChain MCP adapter. The following example connects to an MCP server and passes its tools to `create_deep_agent`. Check the current [LangChain MCP documentation](https://docs.langchain.com/oss/python/langchain/mcp) for supported transports and setup:

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

Some Managed Deep Agents deployments provide an MCP endpoint. When available, an MCP client such as Claude Code can call the deployed agent through that endpoint. Check the deployment documentation for configuration details.

---

## Chapter: Compare Deep Agents with Other Harnesses

This table compares Deep Agents with the Claude Agent SDK. The comparison is based on [LangChain's documentation](https://docs.langchain.com/oss/python/deepagents/comparison), which describes both products from LangChain's perspective. Product features and deployment options can change; check the official documentation for current details before making a decision.

| Feature               | Deep Agents                                        | Claude Agent SDK                                   |
| --------------------- | -------------------------------------------------- | -------------------------------------------------- |
| **Where agent runs**  | Inside a sandbox, or outside executing commands remotely | Inside a sandbox                             |
| **Execution backend** | Pluggable in the SDK (local, virtual, remote sandbox, custom); Managed Deep Agents uses one managed sandbox | Local filesystem of the sandbox |
| **Model provider**    | Any (Anthropic, OpenAI, Google, 100+ others)       | Claude (Anthropic, Bedrock, Vertex, Azure)         |
| **Per-model tuning**  | Harness profiles (beta)                            | Configure in code at each model call site          |
| **Deployment**        | Default: Managed Deep Agents (`mda deploy`); or self-host a standalone image via `langgraph build` | Self-hosted; you build the server, auth, streaming |
| **Multi-tenancy**     | Built-in: scoped threads, per-user sandboxes, RBAC | Build it yourself                                  |
| **License**           | MIT                                                | MIT (Claude Code itself is proprietary)            |

**Consider Deep Agents if** you need to choose among model providers and execution backends, and want to evaluate both managed and self-hosted deployment options.

**Consider the Claude Agent SDK if** your application uses Claude and you want to build and operate your own server, authentication and multi-tenant features.

---

## Chapter: Choose the Right LangChain Abstraction

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

## Chapter: Apply the Architecture to Real Work

### Pattern 1: Research and Report Agent

Combine an internet search tool, filesystem storage for research results and a subagent for a separate research task.

- The main agent plans the work, assigns the research task and writes the final report.
- The research subagent searches the sources it can access and returns a concise summary.

### Pattern 2: Coding Agent with Sandbox

Use a sandbox backend and the `execute` tool to run development commands in the sandbox.

- The agent can install packages, run tests, change code and run Git commands in the sandbox, subject to its permissions.
- Configure human approval for `write_file` and `execute` if people must review those operations.
- Avoid placing secrets in the sandbox. Run authenticated operations through tools in the host application when possible.

### Pattern 3: Enterprise Document Processor

Use `AGENTS.md` for company-wide guidance, skills for specific workflows and `StoreBackend` when information must persist across threads.

- Memory can provide the same style and format guidance in each session.
- Skills can provide specialized instructions for work such as document review or financial analysis when the agent uses them.

### Pattern 4: Multi-Tenant SaaS Agent

Use `CompositeBackend` to route session files to `StateBackend` and persistent user files to `StoreBackend`. Configure a namespace factory to separate users' stored data.

- Each user's session state is stored separately when the application assigns separate threads.
- The store can preserve user preferences and history between conversations when the namespace and storage are configured for that purpose.

### Pattern 5: Data Pipeline Agent

Use separate subagents to collect data, analyze it and format the result. Define a structured output schema for each step that requires a specific data format.

- Each subagent can return data validated against a Pydantic model.
- The main agent coordinates the steps and receives each subagent's returned result.

---

## Chapter: Diagnose Common Failures

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

## Chapter: Check Compatibility Before Copying Examples {#version-requirements}

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

## Chapter: Keep the Core API at Hand

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

## Chapter: Continue Learning

- [Deep Agents Docs](https://docs.langchain.com/oss/python/deepagents/overview)
- [Deep Agents GitHub](https://github.com/langchain-ai/deepagents)
- [Managed Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview)
- [LangSmith Docs](https://docs.langchain.com/langsmith)
- [LangGraph Docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [Agent Skills Standard](https://agentskills.io/)
- [AGENTS.md Standard](https://agents.md/)
- [Examples Repo](https://github.com/langchain-ai/deepagents/tree/main/examples)
- [API Reference](https://reference.langchain.com/python/deepagents)
