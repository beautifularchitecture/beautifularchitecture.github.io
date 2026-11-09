+++
date = '2026-11-08T12:00:00+10:00'
draft = true
title = 'Forward Deployed Engineer'
tags = ['AI', 'AI Engineering', 'Forward Deployed Engineer']
summary = 'Forward-deployed engineer.'
+++

## What a GPU is

A GPU (Graphics Processing Unit) is a processor built to run many simple calculations at the same time. A CPU has a small number of powerful cores designed for sequential, general-purpose work. A GPU has thousands of smaller, simpler cores designed for parallel work.

The GPU originated to render graphics, where the same math must be applied to millions of pixels or triangles independently. That same property—thousands of independent, identical calculations running in parallel—is exactly what training and running AI models requires.

### Why GPUs help in AI engineering

AI models, especially large language models, spend almost all their time doing matrix multiplication: multiplying large grids of numbers together, over and over, across billions of parameters. A CPU can do this math, but one core at a time (or a few cores at a time), which is far too slow for models of that size.

A GPU runs thousands of those multiplications simultaneously. For a forward-deployed engineer, that speed difference matters in practice:

- **Local prototyping:** You can test model behavior, prompts, and fine-tuning runs on your own machine instead of waiting on cloud jobs.
- **Customer deployments:** On-site systems often need low-latency inference without round-tripping to a cloud API, so you size and tune GPU-backed hardware for the customer's environment.
- **Evaluation:** Running evals over a dataset means executing the model hundreds or thousands of times; GPU parallelism turns that from hours into minutes.

The forward-deployed engineer must understand GPU constraints—memory size, memory bandwidth, and compute capacity—to decide whether a model fits on a given machine and how fast it will run for the customer.

### What VRAM is

VRAM (Video RAM) is the memory physically located on the GPU itself, separate from the system's main RAM. It stores the model weights, the activations computed during a forward pass, and any intermediate tensors needed while the GPU is working.

VRAM matters because it sets a hard ceiling on what you can run:

- **Model size:** A model's weights must fit in VRAM. A 7-billion-parameter model at 4-bit quantization needs roughly 4–5 GB of VRAM just for weights; at 16-bit precision it needs roughly 14 GB.
- **Context length:** Longer inputs and longer generated outputs require more activation memory.
- **Batch size:** Processing multiple inputs at once multiplies memory use.

If a model plus its activations exceed available VRAM, the system either fails to load the model or must swap data to slower system memory, which destroys the speed advantage of using a GPU.

### Unified memory on Mac

Apple Silicon Macs (M1, M2, M3, M4, and later) use a **unified memory architecture**. The CPU and GPU share the same pool of memory; there is no separate VRAM and no dedicated GPU memory chip. The GPU can read and write directly to the system's RAM, and vice versa, without copying data between a CPU memory pool and a GPU memory pool.

Working on a Mac has practical consequences:

- **The entire system memory pool is available to the GPU.** A Mac with 64 GB of unified memory can allocate much of that to GPU workloads, so models that would not fit on a discrete GPU with 8–16 GB of VRAM may run on a Mac with 64 GB or more of unified memory.
- **No CPU-to-GPU copy overhead.** Because both processors address the same memory, data does not need to be copied back and forth, which can make smaller models feel responsive even without a discrete GPU.
- **Memory is still a finite limit.** Unified memory is shared with macOS and all running applications, so the portion available to the model is the total minus what the OS and other apps consume. It is still a hard ceiling: if the model plus activations exceed available unified memory, it will not run or will swap to disk and slow dramatically.

Unified memory removes the traditional VRAM bottleneck on Macs, but it does not remove the memory ceiling entirely.

## Proprietary and open models

AI models are distributed under different openness terms, which affects how a forward-deployed engineer can use, modify, and deploy them for customers. The three main categories are open source, open weight, and closed (proprietary). For an up-to-date comparison of how open source models rank against proprietary ones, see the open models leaderboard at https://llmleaderboard.com/open and the model pricing and performance data at https://artificialanalysis.ai/.

|                                                    | Open source                                                                                    | Open weight                                                                                                            | Closed / proprietary                                                       |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| What is released                                   | Model weights plus training code, training data, and full source under an OSI-approved license | Trained model weights, often with inference code                                                                       | Access only via API or a hosted product; nothing about the model is public |
| What is withheld                                   | Generally nothing significant                                                                  | Training code, training data, and sometimes fine-tuning or commercial-use rights are restricted by the model's license | Weights, training code, training data, architecture details                |
| Example models                                     | OLMo (AI2), BLOOM (BigScience)                                                                 | Llama (Meta), Mistral (Mistral AI), Qwen (Alibaba), Gemma (Google)                                                     | GPT-4o (OpenAI), Claude 3.5 Sonnet (Anthropic), Gemini (Google)            |
| Can run on customer hardware / air-gapped          | Yes                                                                                            | Yes                                                                                                                    | No                                                                         |
| Can fine-tune the base model yourself              | Yes                                                                                            | Often, subject to license                                                                                              | No; only provider-hosted fine-tuning if offered                            |
| Customer data leaves your network                  | No                                                                                             | No                                                                                                                     | Yes, sent to the provider's API                                            |
| Can redistribute modified versions                 | Yes                                                                                            | Depends on license                                                                                                     | No                                                                         |
| Typical deployment for a forward-deployed engineer | Self-hosted, full control                                                                      | Self-hosted on customer or your hardware                                                                               | API integration                                                            |

## Dense models vs Mixture of Experts (MoE)

**Dense models** activate every parameter for every token. If a model has 7 billion parameters, all 7 billion are used in every forward pass, regardless of the input. Compute and memory scale linearly with total parameter count.

**Mixture of Experts (MoE) models** split the network into multiple "expert" sub-networks. A router (gating network) selects only a few experts per token, so each token activates only a subset of the total parameters. An MoE model can have a very large total parameter count but a much smaller active (activated) parameter count per token.

|                            | Dense                    | Mixture of Experts (MoE)                                  |
| -------------------------- | ------------------------ | --------------------------------------------------------- |
| Parameters used per token  | All parameters           | A subset, chosen by a router                              |
| Total vs active parameters | Total = active           | Total can be much larger than active                      |
| Compute per token          | Scales with total size   | Scales with active size plus routing overhead             |
| Memory to load             | Must fit all weights     | Must fit all weights, but active compute is lower         |
| Training cost              | High for large models    | Lower compute for equivalent total size                   |
| Inference cost             | High for large models    | Can be cheaper per token than an equivalent dense model   |
| Examples                   | Qwen3-32B, Llama-3.1-70B | Qwen3.6-35B-A3B, DeepSeek-V4-Flash, Kimi K2, gpt-oss-120b |

### Naming conventions

Dense models are named by total parameter count, and every parameter is active:

- `Qwen3-32B` — 32 billion parameters, all active
- `Llama-3.1-70B` — 70 billion parameters, all active

MoE models use two naming conventions that mean different things.

**`NxM`: N experts of M parameters each.** Mixtral uses this form. The product of N and M is not the total parameter count, because the attention layers and embeddings are shared by all experts:

- `Mixtral-8x7B` — 8 experts of 7 billion parameters each, 46.7 billion total and 12.9 billion active per token, because the router selects 2 experts per token (https://mistral.ai/news/mixtral-of-experts)
- `Mixtral-8x22B` — 8 experts of 22 billion parameters each, 141 billion total and 39 billion active per token (https://docs.mistral.ai/models/mixtral-8x22b-0-1-0-3)

**`TOTAL-A_ACTIVE`: total parameters followed by active parameters.** The `A` prefix marks the activated count, so `A3B` means 3 billion activated parameters, not 3 experts:

- `Qwen3.6-35B-A3B` — 35 billion total and 3 billion active, with 256 experts, 8 routed experts and 1 shared expert per token (https://huggingface.co/Qwen/Qwen3.6-35B-A3B)
- `Qwen3-30B-A3B` — 30.5 billion total and 3.3 billion active (https://huggingface.co/Qwen/Qwen3-30B-A3B)
- `Qwen3-235B-A22B` — 235 billion total and 22 billion active (https://qwenlm.github.io/blog/qwen3/)
- `DeepSeek-V4-Flash` — 284 billion total and 13 billion active; `DeepSeek-V4-Pro` is 1.6 trillion total and 49 billion active (https://api-docs.deepseek.com/news/news260424)
- `Kimi K2` — 1 trillion total and 32 billion active (https://github.com/MoonshotAI/Kimi-K2)
- `gpt-oss-120b` — 117 billion total and 5.1 billion active (https://openai.com/index/introducing-gpt-oss/)

Not every MoE name encodes its size. `DeepSeek-V4-Flash`, `Kimi K2` and `gpt-oss-120b` either omit the active count or round the total count, so a forward-deployed engineer must check both numbers in the model card. Total parameters determine the disk and VRAM needed to load the model, while active parameters determine the approximate compute cost per token.

### What this means for deployment

For a forward-deployed engineer, MoE models can offer larger effective capacity at lower inference cost than an equivalently sized dense model, but they come with trade-offs:

- You still need enough VRAM/unified memory to hold **all** weights, even though only some are active per token, so loading an MoE model can require as much memory as a dense model of the same total size.
- The routing layer adds complexity and can introduce latency if not well optimized.
- MoE models can be harder to quantize and fine-tune than dense models.
- For customer deployments where hardware is fixed, an MoE model may let you run a larger, more capable model than a dense model of the same active compute budget.

## Cerebras

Cerebras is not a single model. It is a company (Cerebras Systems) that builds AI hardware and runs an inference cloud. Its best-known product is the Wafer-Scale Engine (WSE) — a single processor the size of an entire silicon wafer, with hundreds of thousands of cores and on-chip memory, built to avoid the memory-bandwidth bottleneck that limits GPUs.

Two things people mean when they say "Cerebras model":

**Cerebras-GPT** — a family of open source models (from 111 million to 13 billion parameters) trained on Cerebras hardware and released under an Apache 2.0 license. You can download the weights and run them yourself, like other open weight models.

**Cerebras Inference** — a cloud API that runs models such as Llama 3.1 at very high token-per-second speeds, often far faster than GPU-based inference. You send prompts over an API; you never see the weights or run anything locally.

For a forward-deployed engineer, Cerebras matters when inference speed is the deciding factor. A customer needing real-time token generation at scale might choose Cerebras Inference via API, or an on-prem CS-3 system if data cannot leave their network. The trade-off is the same as other closed inference providers: you depend on Cerebras's availability and pricing, and you cannot inspect or modify the model internals unless you use Cerebras-GPT weights yourself.

## OpenRouter

OpenRouter is not a model. It is a model gateway — a single API that routes requests to hundreds of models from many providers (OpenAI, Anthropic, Google, Meta, Mistral, Cohere, and others). Instead of managing separate API keys, SDKs, and billing accounts for each provider, you send every request to OpenRouter and it forwards it to the model you specify.

Key capabilities:

- **One API, many models.** Switch models by changing a single string in the request (`model` parameter) without changing code or managing new credentials.
- **Unified billing.** One account, one bill, with usage broken down by model and provider.
- **Fallback routing.** Configure a chain of models; if the first choice fails or exceeds rate limits, OpenRouter retries on the next model in the list.
- **Load balancing.** Distribute traffic across multiple providers to avoid hitting any single provider's rate limits.
- **Routing by cost or capability.** Pick the cheapest model that meets a quality bar, or route complex prompts to larger models and simple ones to smaller models.

Example models accessible through OpenRouter include GPT-4o (OpenAI), Claude 3.5 Sonnet (Anthropic), Gemini (Google), Llama 3.1 (Meta), and Mistral Large (Mistral AI), among hundreds of others.

For a forward-deployed engineer, OpenRouter matters when you want to offer customers model choice without managing a dozen provider integrations. It is useful for prototyping (try several models quickly), for cost-sensitive deployments (route to cheaper models where possible), and for resilience (fallback to another provider if one is down). The trade-off is that you depend on OpenRouter's availability and add a hop between your application and the model provider, which can add slight latency and means customer data passes through OpenRouter's infrastructure unless you self-host an equivalent gateway.
