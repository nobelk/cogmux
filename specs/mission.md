# cogmux — Mission

## Vision

cogmux is the first production-grade SDK for building AI agents with **three-tier cognitive architecture**. It operationalizes the Hierarchical Language Agent pattern (AAMAS 2024, Liu et al.) — decomposing agent cognition into a deliberative Slow Mind, a tactical Fast Mind, and a reactive Executor — so developers can build agents that reason deeply *and* act instantly.

The "MVC for agentic AI": Slow Mind is the Model (state and reasoning), Fast Mind is the Controller (request routing and coordination), Executor is the View (rendering actions to the environment).

## Audience

1. **AI application developers** building real-time copilots, game AI, conversational agents, or any system that needs sub-second response while retaining strategic reasoning.
2. **Platform teams at AI-native companies** orchestrating multiple agents who need tiered execution to control cost and latency.
3. **Robotics and IoT engineers** integrating LLM reasoning with real-time control loops.
4. **Game studios** building adaptive NPC AI that reasons about player intent but acts in real time.

## Scope

### In scope (v1)

- Three-tier runtime: Slow Mind, Fast Mind, Executor with typed async message passing.
- Tier-aware LLM routing with configurable model mapping per tier.
- Async communication bus with backpressure, dead-letter handling, and goal versioning.
- Fallback chains when Fast Mind exceeds latency thresholds.
- Goal preemption protocol (version-stamped goals with soft/hard preemption).
- Python SDK with decorator-based tier definition.
- Optional Rust executor runtime via `cogmux[rust]` (PyO3).
- Model providers: Anthropic, OpenAI, Ollama/vLLM.
- OpenTelemetry observability: per-tier latency, cost, token tracking, tier transition traces.
- Replay/debug recording and playback.
- Slow Mind health monitoring with configurable staleness detection.
- CLI: `cogmux run`, `cogmux replay`.
- Extension point interface (`CogmuxRunnable` protocol) for community framework adapters.

### Out of scope (v1)

- Framework adapters (LangChain, LangGraph, CrewAI) — deferred to post-v1; extension point ships instead.
- Google Gemini provider — deferred to post-v1.
- Adaptive tier boundary learning (ML-driven routing).
- Multi-agent cogmux coordination (independent three-tier stacks with consensus layer).
- Edge deployment packaging (ONNX Runtime, TensorRT).
- GUI configurator — CLI and code only.
- Fine-tuned Fast Mind models.
- MCP tool integration — deferred to post-v1.
- Shared context store — deferred to post-v1.
- Cost budgeting per-session/per-tier — deferred to post-v1.

## Guiding principles

1. **Latency is a feature.** Every design decision is evaluated against the latency budget: Executor <10ms P99 dispatch overhead (time from receiving an AtomicAction to initiating the tool call, excluding tool execution), Fast Mind <500ms P95, Slow Mind never blocks Executor.

2. **Plays well with others.** cogmux complements existing ecosystems (LangChain, CrewAI, MCP) rather than replacing them. The extension point interface ensures interoperability without coupling release timelines to third-party APIs.

3. **Separation of cognitive concerns.** Strategic reasoning, tactical planning, and reactive execution are fundamentally different problems. Mixing them in a single LLM call creates latency walls, cost explosions, and architectural fragility.

4. **Progressive complexity.** A minimal cogmux agent (three decorated functions + `agent.run()`) works out of the box. Advanced features (Rust executor, replay, custom bus backends) are opt-in.

5. **Observable by default.** Every tier transition, model call, and fallback emits OpenTelemetry spans. Debugging a bad agent outcome means tracing causality from Executor back through Fast Mind to Slow Mind — the SDK must make this possible without additional instrumentation.

6. **Pure Python first, Rust when proven.** The default runtime is pure Python asyncio. Rust acceleration (`cogmux[rust]`) ships as an optional extra, justified by profiling, not assumption.

## Concurrency contract

The three-tier runtime must uphold the following behavioral invariants under concurrent operation:

1. **Goal version authority.** The goal channel is the single source of truth for the current goal version. When Slow Mind publishes a new Goal, its `version` field becomes authoritative the moment it is consumed from the channel.

2. **Soft preemption drains in-flight actions.** On soft preemption, Fast Mind stops producing new MacroActions for the old goal version but does not cancel already-dispatched AtomicActions. Executor completes in-flight actions whose `goal_version` was current at dispatch time, then discards any remaining queued actions with a stale version.

3. **Hard preemption drops immediately.** On hard preemption, Executor discards all queued AtomicActions with a stale `goal_version` and signals cancellation to any in-progress tool calls that support cooperative cancellation.

4. **No stale execution.** Executor must compare `AtomicAction.goal_version` against the current goal version before every dispatch. An action whose version is behind the current version is never executed.

5. **Cancellation surfaces to Fast Mind.** When Executor drops stale actions (soft or hard preemption), it publishes an Observation with a `preempted` flag so Fast Mind can observe the preemption and re-plan against the new goal.

6. **Bus ordering guarantees.** The default `asyncio.Queue` bus preserves FIFO ordering within each channel. Future bus backends (Redis, NATS) must preserve per-channel FIFO ordering; cross-channel ordering is not guaranteed and must not be relied upon.

## Source of truth

The `specs/` directory is the authoritative source for architecture, scope, and roadmap decisions. Other documentation (e.g., `docs/`) may contain supplementary context but defers to `specs/` on any conflict.
