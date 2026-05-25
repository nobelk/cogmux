# Plan — Foundation

Phase: 0 · Date: 2026-05-24

## 1. Project scaffolding

Goal: Establish the package structure, tooling configs, and CI pipeline so every subsequent commit is linted, type-checked, and tested automatically.

- [ ] Create `pyproject.toml` with uv-compatible metadata, Python 3.11+ constraint, and dependency groups (core, dev, rust optional)
- [ ] Configure ruff: linting rules (select = ["E", "F", "I", "UP", "B", "SIM"]) and formatting (line-length 88)
- [ ] Configure mypy: strict mode, per-module overrides for tests
- [ ] Set up pytest with pytest-asyncio (auto mode), configure in `pyproject.toml`
- [ ] Create `src/cogmux/__init__.py` and `src/cogmux/py.typed` marker
- [ ] Create GitHub Actions CI workflow: Python 3.11, 3.12, 3.13 matrix → ruff check, ruff format --check, mypy, pytest

## 2. Message schemas

Goal: Define the Pydantic v2 models that form the inter-tier communication contract.

- [ ] `Goal` model: `id`, `version`, `description`, `context` (arbitrary dict), `timestamp`
- [ ] `MacroAction` model: `id`, `goal_id`, `goal_version`, `steps` (list of action descriptors), `timestamp`
- [ ] `AtomicAction` model: `id`, `macro_action_id`, `goal_version`, `action_type`, `params`, `timestamp`
- [ ] `Observation` model: `id`, `action_id`, `goal_version`, `result`, `error`, `preempted` flag, `timestamp`
- [ ] `TierConfig` model: `tier_name`, `model_name`, `max_latency_ms`, `interval_s` (Slow Mind scheduling), provider config
- [ ] `AgentConfig` model: `slow_mind`, `fast_mind`, `executor` tier configs, bus config (channel maxsize)

## 3. Communication bus

Goal: Implement the async channel layer that connects tiers, with configurable capacity and clear backpressure behavior.

- [ ] `Channel` class wrapping `asyncio.Queue` with typed generic interface
- [ ] `Bus` class composing three channels: `goal_channel`, `action_channel`, `observation_channel`
- [ ] Configurable `maxsize` per channel (from `AgentConfig.bus_config`)
- [ ] Backpressure policy: decide and implement block/raise/drop behavior (resolve open question)
- [ ] Bus lifecycle methods: `start()`, `stop()`, `drain()`

## 4. Tier runtime stubs

Goal: Create async task shells for each tier that prove the bus wiring works without any LLM logic.

- [ ] `SlowMindRuntime`: async task that periodically publishes a `Goal` to the goal channel
- [ ] `FastMindRuntime`: async task that consumes `Goal`, decomposes it internally into MacroActions, then emits `AtomicAction` messages on the action channel
- [ ] `ExecutorRuntime`: async task that consumes `AtomicAction`, dispatches (no-op/logging), publishes `Observation`
- [ ] Tier base class or protocol defining the `start()` / `stop()` / `run()` contract
- [ ] Wire all three stubs through `Bus` and verify messages flow in a basic integration test

## Sequencing notes

- Group 1 (scaffolding) must complete first — everything else depends on the package structure and CI.
- Group 2 (schemas) before Group 3 (bus) — the bus is generic over message types defined in Group 2.
- Group 4 (stubs) depends on both schemas and bus.
- The open questions in `requirements.md` (bus typing, backpressure, config mutability) should be resolved during or before Group 3 implementation.
