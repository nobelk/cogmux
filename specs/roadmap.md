# cogmux — Roadmap

Solo developer, ~6 months. Each phase is 2–4 weeks, designed to produce a working (if incomplete) agent at the end of each phase. Early phases (0–2) are internal milestones validated by tests and benchmarks; external adoption checkpoints begin at Phase 7 when the CLI and extension points ship.

---

## Phase 0 — Foundation (3 weeks)

Skeleton project, data model, and a minimal end-to-end proof that three tiers can communicate.

- [ ] Project scaffolding: `pyproject.toml` (uv), ruff config, mypy config, pytest setup, GitHub Actions CI.
- [ ] Pydantic v2 message schemas: `Goal`, `MacroAction`, `AtomicAction`, `Observation`, `TierConfig`, `AgentConfig`.
- [ ] Communication bus: `asyncio.Queue`-based channels (goal, action, observation) with configurable `maxsize`.
- [ ] Tier runtime stubs: Slow Mind, Fast Mind, Executor as async tasks that read/write from bus channels.
- [ ] Hardcoded demo: three tiers passing messages in a loop with no LLM calls — validates bus, message schemas, and task lifecycle.
- [ ] Unit tests for message schemas and bus behavior (backpressure, ordering).

## Phase 1 — Model routing & first LLM calls (3 weeks)

Plug real LLM providers into the tier runtimes.

- [ ] `ModelProvider` protocol: abstract interface for LLM calls (chat, streaming).
- [ ] Anthropic provider: Claude Opus (Slow Mind), Claude Haiku (Fast Mind). Streaming support.
- [ ] OpenAI provider: GPT-4o (Slow Mind), GPT-4o-mini (Fast Mind).
- [ ] Ollama provider: local model inference via httpx.
- [ ] Model router: resolve tier config → provider + model. Connection pooling.
- [ ] End-to-end demo: Slow Mind sets a goal via Opus, Fast Mind plans actions via Haiku, Executor dispatches (no tools yet, just logging).
- [ ] Integration tests with mock LLM responses.
- [ ] Latency benchmark harness: measure Executor dispatch overhead (target <10ms P99) and Fast Mind round-trip (target <500ms P95) with real and mock providers. Run in CI on every commit. This is the acceptance harness for the core latency contract — architecture choices in subsequent phases are validated against it.

## Phase 2 — Decorator API & agent lifecycle (2 weeks)

The developer-facing SDK surface.

- [ ] `@cogmux.slow_mind`, `@cogmux.fast_mind`, `@cogmux.executor` decorators.
- [ ] `cogmux.Context` object: access to LLM client, observations, ephemeral session state (in-process dict scoped to the agent run — not the post-v1 shared context store), and bus.
- [ ] `cogmux.Agent` class: wires decorated functions to tier runtimes.
- [ ] `agent.run()` lifecycle: start tiers, manage asyncio tasks, graceful shutdown.
- [ ] Slow Mind scheduling: periodic invocation (`interval_s`) and on-demand trigger.
- [ ] Minimal working example matching the SDK surface from the dev design doc.
- [ ] Session recording (basic): all tier interactions, model calls, and observations captured to JSONL during `agent.run()`. Recording starts here so that concurrency debugging in Phases 3–4 has an artifact to inspect. The full replay CLI and diff view remain in Phase 5.

## Phase 3 — Fallback chains & goal preemption (3 weeks)

Resilience: what happens when things are slow or plans change.

- [ ] Fast Mind timeout detection: monitor latency against `max_latency_ms`.
- [ ] Fallback strategies: `cached` (replay last MacroAction), `default` (safe action), `escalate` (queue for Slow Mind).
- [ ] Goal versioning: `version` field on Goal, `goal_version` on MacroAction and AtomicAction.
- [ ] Goal preemption protocol: bus broadcasts preemption signal, Fast Mind re-plans, Executor drops stale actions.
- [ ] Executor version check: compare `AtomicAction.goal_version` against current goal before dispatch.
- [ ] Dead-letter channel: malformed messages moved to dead-letter after `max_retries` (default 3).
- [ ] Tests: timeout scenarios, preemption race conditions, dead-letter behavior.

## Phase 4 — Observability (2 weeks)

OpenTelemetry instrumentation so developers can debug across tiers.

- [ ] OTel span per tier transition: `cogmux.tier.slow_mind`, `cogmux.tier.fast_mind`, `cogmux.tier.executor`.
- [ ] Span attributes: latency, model name, token count (input/output), goal version.
- [ ] Fallback spans: tagged `fallback=true` with strategy used.
- [ ] Metrics: `cogmux_tier_latency_ms`, `cogmux_fallback_rate`, `cogmux_cost_usd`, `cogmux_goal_version`.
- [ ] Structured JSON logging: `session_id`, `tier`, `goal_id`, `action_id`.
- [ ] Dead-letter events as OTel spans.
- [ ] Slow Mind health monitoring: staleness detection on goal channel, `cogmux.slow_mind.stale` event.

## Phase 5 — Replay & debug (3 weeks)

Replay CLI and advanced debugging on top of the session recording introduced in Phase 2.

- [ ] Extend Phase 2 session recording with tool results, fallback events, and preemption signals.
- [ ] `cogmux replay session.jsonl`: load and replay recorded sessions.
- [ ] Model override on replay: swap Fast Mind model, re-run, compare outputs.
- [ ] Diff view: show divergence between original and replayed execution.
- [ ] Slow Mind replay: replay from recording (don't re-call LLM), re-run Fast Mind and Executor live.
- [ ] CLI: `cogmux replay session.jsonl --fast-mind-model claude-haiku-4-5`.

## Phase 6 — Rust executor (3 weeks) — GATED

Optional high-performance executor for latency-critical deployments.

**Gate:** Only proceed if benchmark data from Phases 1–5 shows the pure Python executor misses the <10ms P99 dispatch overhead target. If the Python executor meets the target, skip this phase and move Rust to the post-v1 backlog. This aligns with the mission principle: "Pure Python first, Rust when proven."

- [ ] Rust executor crate: action dispatch loop, PyO3 bindings.
- [ ] `cogmux[rust]` extra: maturin-based build, platform wheels (manylinux, macOS ARM/x86).
- [ ] Python ↔ Rust bridge: Executor receives AtomicAction from Python bus, dispatches in Rust, returns Observation.
- [ ] Benchmark: Python executor vs. Rust executor dispatch latency.
- [ ] CI: Rust build + test in GitHub Actions matrix.
- [ ] Fallback: if Rust executor fails to load, fall back to Python executor with warning.

## Phase 7 — CLI & extension points (2 weeks)

Developer tooling and the integration story.

- [ ] `cogmux run agent.py`: discover and run agent from a Python file.
- [ ] `cogmux run` with `--slow-mind-model`, `--fast-mind-model` overrides.
- [ ] `CogmuxRunnable` protocol: interface for framework adapters (LangChain, LangGraph, CrewAI).
- [ ] Example adapter: minimal LangChain `Runnable` wrapping a cogmux agent (in `examples/`, not core).
- [ ] `cogmux init`: scaffold a new agent project with boilerplate.

## Phase 8 — Polish & release (3 weeks)

Documentation, benchmarks, and PyPI release.

- [ ] API reference documentation (auto-generated from docstrings).
- [ ] Tutorial: build a customer support copilot step by step.
- [ ] Tutorial: build an adaptive game NPC.
- [ ] Full benchmark suite: end-to-end latency, cost, and task completion rate vs. monolithic agent baseline. Extends the Phase 1 latency harness with comparative analysis and publishable results.
- [ ] PyPI release: `pip install cogmux` and `pip install cogmux[rust]`.
- [ ] README with quick-start, architecture diagram, and benchmark results.
- [ ] CHANGELOG and contribution guide.

---

## Post-v1 backlog

Deferred features, ordered by expected impact:

1. **Framework adapters** — First-party LangChain, LangGraph, CrewAI adapters.
2. **Google Gemini provider** — Gemini Ultra for Slow Mind, Gemini Flash for Fast Mind.
3. **MCP tool integration** — Executor and Fast Mind consume MCP tools natively.
4. **Shared context store** — Structured memory accessible by all tiers with tier-appropriate permissions.
5. **Cost budgeting** — Per-session and per-tier budget caps with graceful degradation.
6. **Tier promotion/demotion** — Runtime escalation/delegation based on uncertainty thresholds.
7. **Macro-action library** — Pre-built action vocabularies for common domains.
8. **Adaptive tier boundaries** — ML-driven routing that learns safe delegation patterns.
9. **Multi-agent cogmux** — Multiple agents with independent three-tier stacks + consensus layer.
10. **Edge deployment** — Fast Mind + Executor packaged for edge devices.
11. **Real-time streaming** — WebSocket/gRPC for game loops and robotics (30–1000 Hz).
