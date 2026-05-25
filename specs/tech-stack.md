# cogmux — Tech Stack

## Language & runtime

| Component | Technology | Rationale |
|---|---|---|
| SDK surface | **Python 3.11+** (asyncio) | Matches target user ecosystem. 3.11+ for `TaskGroup`, `ExceptionGroup`, and performance improvements. |
| Executor (optional) | **Rust** via PyO3 | Sub-millisecond action dispatch. Gated behind `cogmux[rust]` extra. Default executor is pure Python asyncio. |
| Package format | PyPI wheel + optional Rust extension | `pip install cogmux` (pure Python), `pip install cogmux[rust]` (with Rust executor). |

## Core dependencies

| Dependency | Purpose | Version constraint |
|---|---|---|
| **pydantic** v2 | Typed message schemas (Goal, MacroAction, AtomicAction, Observation). Structured output parsing from LLMs. | `>=2.0,<3.0` |
| **anthropic** | Anthropic Claude API client (Slow Mind: Opus, Fast Mind: Haiku). Streaming, tool use, batch APIs. | `>=0.40` |
| **openai** | OpenAI API client. GPT-4o for Slow Mind, GPT-4o-mini for Fast Mind. | `>=1.0` |
| **httpx** | HTTP client for Ollama, vLLM, and HuggingFace TGI local model inference. | `>=0.27` |
| **opentelemetry-api** / **opentelemetry-sdk** | Per-tier spans, metrics, and trace export. | `>=1.20` |
| **PyO3** / **maturin** | Rust-Python FFI for optional Rust executor. Build tooling. | Build-time only for `cogmux[rust]`. |

## Model providers (v1)

### Tier capability requirements

The SDK defines provider requirements by capability, not by model name. Specific models are current defaults that will change as providers release new SKUs.

| Capability | Slow Mind | Fast Mind | Notes |
|---|---|---|---|
| Structured output (JSON mode or tool use) | Required | Required | Pydantic schema enforcement for Goal, MacroAction. |
| Streaming | Optional | Required | Fast Mind streams to meet <500ms P95 time-to-first-token. |
| Tool use / function calling | Required | Required | Both tiers invoke tools; Executor dispatches results. |
| Extended thinking / chain-of-thought | Preferred | Not required | Slow Mind benefits from deliberative reasoning. |
| Context window | ≥128K tokens | ≥32K tokens | Slow Mind needs large context for strategic reasoning. |
| Prompt caching | Optional | Preferred | Reduces Fast Mind latency and cost on repeated patterns. |

### Current default models

These are the default model mappings as of the initial release. The model router resolves tier configs to providers; adding a new model means updating the config, not modifying tier code.

| Provider | Models | Tier mapping | Notes |
|---|---|---|---|
| **Anthropic** | Claude Opus 4.7, Claude Sonnet 4.6, Claude Haiku 4.5 | Slow Mind → Opus/Sonnet, Fast Mind → Haiku | Primary provider. Extended thinking for Slow Mind. Prompt caching for Fast Mind. |
| **OpenAI** | GPT-4o, GPT-4o-mini | Slow Mind → GPT-4o, Fast Mind → GPT-4o-mini | Secondary cloud provider. |
| **Ollama / vLLM** | Llama 3, Phi-3, Mistral, Qwen | Fast Mind → any local model | Local inference for cost-sensitive or air-gapped deployments. Accessed via httpx. |

## Development tooling

| Tool | Purpose |
|---|---|
| **uv** | Package management and virtual environments. |
| **pytest** + **pytest-asyncio** | Unit and integration tests. Async test support. |
| **ruff** | Linting and formatting (replaces flake8 + black + isort). |
| **mypy** | Static type checking. Strict mode. |
| **maturin** | Build Rust extensions for `cogmux[rust]`. |
| **GitHub Actions** | CI: lint, type check, test (Python matrix 3.11–3.13), build wheels. |

## Architecture constraints

- **Async-native throughout.** All tier runtimes, the communication bus, and model clients use `asyncio`. No blocking calls in the hot path.
- **Thread-safe.** Executor may run in a thread pool (pure Python) or release the GIL (Rust). Shared state protected by asyncio primitives, not threading locks.
- **Communication bus is pluggable.** Default: `asyncio.Queue` (single-process). Interface designed for future Redis/NATS backends (multi-process, distributed). Not in v1 scope.
- **Model router abstraction.** Tier configs specify model names; the router resolves to provider clients. Adding a new provider means implementing a `ModelProvider` protocol, not modifying tier code.
- **SDK footprint <50MB** installed (pure Python). Executor runtime <10MB standalone (Rust).
- **No LLM in the Executor hot path.** Executor dispatches tool calls and policy actions only. LLM calls are confined to Slow Mind and Fast Mind tiers.

## Deployment targets

- **Development:** Local Python environment via `uv`. Ollama for local Fast Mind inference.
- **Production:** Any Python 3.11+ environment. Cloud model providers via API keys in environment variables.
- **CI:** GitHub Actions with Python 3.11, 3.12, 3.13 matrix. Optional Rust build for `cogmux[rust]` wheels.
- **Distribution:** PyPI. Source distribution + platform wheels (manylinux, macOS ARM/x86, Windows) for `cogmux[rust]`.
