# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

cogmux is a Python SDK implementing a three-tier cognitive architecture for AI agents, based on the Hierarchical Language Agent pattern (AAMAS 2024). It decomposes agent cognition into:

- **Slow Mind** — Frontier LLM (Opus/GPT-4o) for strategic reasoning, called infrequently (seconds-scale)
- **Fast Mind** — Lightweight LLM (Haiku/GPT-4o-mini) for tactical planning, called frequently (<500ms P95)
- **Executor** — Reactive policy engine dispatching atomic actions (<10ms P99 dispatch overhead, no LLM)

Tiers communicate via an async bus (asyncio.Queue channels: goal, action, observation) with backpressure, dead-letter handling, and goal version-based preemption.

## Source of truth

`specs/` is authoritative for architecture, scope, and roadmap. `docs/` contains supplementary context (PRD, dev design) but defers to `specs/` on any conflict.

## Current status

Pre-implementation. Only specs and design docs exist. Implementation follows the phased roadmap in `specs/roadmap.md` (Phase 0 first).

## Build & dev commands

```bash
# Package management
uv sync                           # install dependencies
uv sync --extra rust              # install with optional Rust executor

# Linting & formatting
ruff check .                      # lint
ruff format .                     # format
ruff check --fix .                # auto-fix lint issues

# Type checking
mypy src/                         # strict mode

# Testing
pytest                            # run all tests
pytest tests/test_foo.py          # single file
pytest tests/test_foo.py::test_bar  # single test
pytest -x                         # stop on first failure

# Rust executor (only if cogmux[rust] extra)
maturin develop                   # build Rust extension in-place
```

## Architecture

### Three-tier runtime

Each tier runs as its own asyncio task. Tiers are defined via decorators (`@cogmux.slow_mind`, `@cogmux.fast_mind`, `@cogmux.executor`) and wired together by `cogmux.Agent`.

### Communication bus

Three async channels (goal, action, observation) using `asyncio.Queue` with configurable `maxsize`. Dead-letter channel captures malformed messages after `max_retries`. Bus preserves FIFO ordering per-channel; cross-channel ordering is not guaranteed.

### Goal preemption protocol

Goals carry a `version` field. On preemption:
- **Soft:** Fast Mind stops producing new MacroActions for old version; Executor completes in-flight actions then discards stale queued actions
- **Hard:** Executor immediately discards all stale-version actions and signals cancellation

Executor must compare `AtomicAction.goal_version` against current goal version before every dispatch.

### Model routing

Tier configs specify model names; the router resolves to provider clients (`ModelProvider` protocol). Adding a provider means implementing the protocol, not modifying tier code.

### Data model (Pydantic v2)

Core message types: `Goal`, `MacroAction`, `AtomicAction`, `Observation`, `TierConfig`, `AgentConfig`. All inter-tier messages are Pydantic models with structured output parsing.

## Key constraints

- **Python 3.11+** required (TaskGroup, ExceptionGroup).
- **Async-native throughout** — no blocking calls in the hot path.
- **No LLM in the Executor** — Executor dispatches tool calls and policy actions only.
- **Rust is optional and gated** — only justified if Python executor misses <10ms P99 dispatch target. Default is pure Python asyncio.
- **Latency targets:** Executor <10ms P99 (dispatch overhead only), Fast Mind <500ms P95, Slow Mind never blocks Executor.

## Tech stack

| Tool | Purpose |
|------|---------|
| uv | Package management, virtual environments |
| ruff | Linting + formatting (replaces flake8/black/isort) |
| mypy | Static type checking (strict mode) |
| pytest + pytest-asyncio | Testing |
| pydantic v2 | Message schemas, structured LLM output |
| anthropic / openai / httpx | Model provider clients |
| opentelemetry-api/sdk | Per-tier observability |
| maturin + PyO3 | Rust extension build (optional) |

## Observability conventions

OTel span names: `cogmux.tier.slow_mind`, `cogmux.tier.fast_mind`, `cogmux.tier.executor`. Metrics prefixed with `cogmux_`. Structured JSON logs include `session_id`, `tier`, `goal_id`, `action_id`.
