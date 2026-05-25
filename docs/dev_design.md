# cogmux — Dev Design

Implements [PRD cogmux](../prds/cogmux.md). Three-tier cognitive architecture SDK for building responsive, cost-efficient AI agents.

## Scope & non-goals

**In scope (v1):** Three-tier runtime (Slow Mind / Fast Mind / Executor), async communication bus with dead-letter handling, tier-aware LLM routing, fallback chains, goal versioning and preemption protocol, Python SDK with decorator-based API, OpenTelemetry instrumentation, Anthropic + OpenAI + Google Gemini + Ollama model providers, replay/debug recording, Slow Mind health monitoring.

**Out of scope (v1):** Adaptive tier boundary learning (P2), multi-agent cogmux coordination (P2), edge deployment packaging (P2), ROS2 integration (v1.5), GUI configurator (CLI + code only), fine-tuned Fast Mind models. Framework adapters (LangChain, LangGraph, CrewAI) are P1 — v1 ships an extension point interface (`cogmuxRunnable` protocol) for community adapters.

## High-level architecture

```
┌─────────────────────────────────────────────────────────┐
│                    cogmux SDK (Python)                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Agent Definition Layer              │    │
│  │  @slow_mind / @fast_mind / @executor decorators  │    │
│  └─────────────────┬───────────────────────────────┘    │
│                    │                                     │
│  ┌─────────────────▼───────────────────────────────┐    │
│  │           Communication Bus (async)              │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │    │
│  │  │ Goal     │  │ Action   │  │ Observation  │   │    │
│  │  │ Channel  │  │ Channel  │  │ Channel      │   │    │
│  │  └──────────┘  └──────────┘  └──────────────┘   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │Slow Mind │   │Fast Mind │   │  Executor        │    │
│  │ Runtime  │   │ Runtime  │   │  Runtime (Rust)  │    │
│  │ (async)  │   │ (async)  │   │  via PyO3        │    │
│  └────┬─────┘   └────┬─────┘   └────┬─────────────┘    │
│       │              │              │                   │
│  ┌────▼──────────────▼──────────────▼──────────────┐    │
│  │           Model Router / Tool Bridge             │    │
│  │  Anthropic │ OpenAI │ Ollama │ MCP Tools         │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Observability (OTel spans + tier metrics)       │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Tech stack & rationale

- **Python (3.11+, asyncio)** — SDK surface, tier definitions, LLM client orchestration. Matches target user ecosystem.
- **Rust (PyO3, optional `cogmux[rust]`)** — Optional Executor runtime for sub-millisecond action dispatch. Default executor is pure Python asyncio; Rust acceleration gated behind profiling evidence. Avoids manylinux wheel complexity for early adopters.
- **Pydantic v2** — Typed message schemas for inter-tier communication. Structured output parsing from LLMs.
- **anthropic / openai / httpx** — Model provider clients. httpx for Ollama/vLLM/TGI.
- **OpenTelemetry SDK** — Spans per tier transition, model call, tool invocation.
- **MCP client (mcp-python)** — Tool integration for Fast Mind and Executor tiers.

## Data model

```python
from pydantic import BaseModel
from enum import Enum
from uuid import UUID
from datetime import datetime

class Tier(str, Enum):
    SLOW = "slow_mind"
    FAST = "fast_mind"
    EXECUTOR = "executor"

class Goal(BaseModel):
    id: UUID
    version: int
    intent: str
    constraints: dict[str, Any]
    created_by: Tier
    timestamp: datetime

class MacroAction(BaseModel):
    id: UUID
    goal_id: UUID
    goal_version: int
    steps: list[str]
    fallback: str | None
    ttl_ms: int

class AtomicAction(BaseModel):
    id: UUID
    macro_action_id: UUID
    goal_version: int
    action_type: str
    params: dict[str, Any]
    deadline_ms: int

class Observation(BaseModel):
    id: UUID
    source_action_id: UUID
    tier: Tier
    data: dict[str, Any]
    timestamp: datetime
    metadata: dict[str, Any] | None

class TierConfig(BaseModel):
    tier: Tier
    model: str                      # e.g., "claude-opus-4-7", "claude-haiku-4-5"
    max_latency_ms: int
    fallback_strategy: str          # "cached" | "default" | "escalate"
    budget_per_session_usd: float | None
    tools: list[str]                # MCP tool names

class AgentConfig(BaseModel):
    name: str
    slow_mind: TierConfig
    fast_mind: TierConfig
    executor: TierConfig
    shared_context_size: int        # max tokens in shared context
```

## SDK surface

```python
import cogmux

@cogmux.slow_mind(model="claude-opus-4-7", interval_s=30)
async def strategist(context: cogmux.Context) -> cogmux.Goal:
    """Called periodically or on-demand. Reads observations, sets goals."""
    history = context.observations(last_n=10)
    response = await context.llm.chat(
        messages=[{"role": "user", "content": f"Given {history}, set strategy."}]
    )
    return cogmux.Goal(intent=response.text, constraints={"max_turns": 3})

@cogmux.fast_mind(model="claude-haiku-4-5", max_latency_ms=500)
async def tactician(context: cogmux.Context, goal: cogmux.Goal) -> cogmux.MacroAction:
    """Called on each event. Translates goals to action sequences."""
    response = await context.llm.chat(
        messages=[{"role": "user", "content": f"Goal: {goal.intent}. Plan actions."}]
    )
    return cogmux.MacroAction(steps=response.parsed.steps, fallback="wait")

@cogmux.executor(max_latency_ms=10)
async def actor(context: cogmux.Context, action: cogmux.MacroAction) -> cogmux.Observation:
    """Runs at execution frequency. No LLM calls."""
    result = await context.tools.call(action.steps[0])
    return cogmux.Observation(data=result)

agent = cogmux.Agent(
    name="support_copilot",
    slow_mind=strategist,
    fast_mind=tactician,
    executor=actor,
)

agent.run()
```

## External interfaces

- **PyPI:** `pip install cogmux` — pure Python + Rust extension (manylinux wheels).
- **CLI:** `cogmux run agent.py`, `cogmux replay session.jsonl --fast-mind-model claude-haiku-4-5`.
- **LangChain adapter:** `cogmuxRunnable` wrapping an cogmux agent as a LangChain `Runnable`.
- **LangGraph adapter:** `cogmuxNode` — a LangGraph node containing a full three-tier stack.
- **CrewAI adapter:** `cogmuxCrewAgent` — implements CrewAI `Agent` protocol with internal tiered cognition.
- **MCP:** cogmux agents expose themselves as MCP servers (actions as tools).

## Key flows

### Flow 1 — Standard execution loop

1. Agent starts; Executor enters polling loop at configured frequency.
2. Slow Mind runs initial strategy pass; publishes `Goal` to goal channel.
3. Fast Mind receives goal, generates `MacroAction` sequence; publishes to action channel.
4. Executor pops atomic actions, dispatches via tool bridge, publishes `Observation` to observation channel.
5. Slow Mind periodically reads observations, updates goal if needed (version bump triggers Fast Mind re-plan).

### Flow 2 — Fallback on Fast Mind timeout

1. Fast Mind call exceeds `max_latency_ms`.
2. Bus detects timeout; Executor falls back to `fallback_strategy`:
   - `cached`: replay last successful MacroAction.
   - `default`: execute hardcoded safe action.
   - `escalate`: queue for Slow Mind decision on next cycle.
3. OTel span tagged `fallback=true`; alert emitted if fallback rate exceeds threshold.

### Flow 3 — Goal preemption

1. Slow Mind publishes new Goal with incremented version.
2. Bus broadcasts preemption signal to Fast Mind.
3. Fast Mind abandons in-flight MacroAction, re-plans against new Goal.
4. Executor checks `goal_version` on each `AtomicAction` before dispatch; stale actions (version < current goal version) are dropped. Executor switches to new MacroAction stream.

### Flow 4 — Replay / debug

1. `cogmux replay session.jsonl` loads recorded tier interactions.
2. User overrides model config (e.g., swap Fast Mind model).
3. Runtime replays observations, re-runs Fast Mind and Executor; Slow Mind decisions replayed from recording.
4. Diff view shows divergence between original and replayed execution.

### Flow 5 — Slow Mind health check

1. Bus monitors goal channel for staleness (configurable threshold, default 2× Slow Mind interval).
2. If no new goal published within threshold, health check triggers:
   - OTel event `cogmux.slow_mind.stale` emitted with last goal version and age.
   - If Slow Mind task is alive but blocked on LLM call, log and wait (model provider may be slow).
   - If Slow Mind task has crashed, restart with last known context and emit `cogmux.slow_mind.restarted`.
3. Optional degraded mode: Fast Mind promoted to goal-setting with constrained prompt template until Slow Mind recovers.

## Concurrency, scaling, performance

- Each tier runs in its own asyncio task (Slow Mind and Fast Mind) or thread pool (Executor — pure Python default, optional Rust via `cogmux[rust]`).
- Communication bus uses `asyncio.Queue` with configurable backpressure (drop-oldest for observations, block for goals). Dead-letter channel captures malformed messages after `max_retries` (default 3); dead-letter events emitted as OTel spans.
- Executor hot loop target: <1ms dispatch latency (Rust); PyO3 FFI overhead <0.5ms.
- Fast Mind parallelism: multiple Fast Mind instances can run concurrently for multi-stream agents (e.g., handling N conversations).
- Model calls are the bottleneck; SDK provides connection pooling and request batching for Anthropic/OpenAI clients.

## Observability

- **Spans:** `cogmux.tier.slow_mind`, `cogmux.tier.fast_mind`, `cogmux.tier.executor` — each with latency, model, token count.
- **Metrics:** `cogmux_tier_latency_ms{tier}`, `cogmux_fallback_rate{tier}`, `cogmux_cost_usd{tier,model}`, `cogmux_goal_version`.
- **Logs:** Structured JSON with `session_id`, `tier`, `goal_id`, `action_id`.
- **Replay export:** Full session as JSONL with all tier interactions, model calls, and tool results.

## Security & privacy

- Model API keys stored in environment variables or secret manager; never logged.
- Executor runs tool calls — if tools have side effects, sandbox configuration is the deployer's responsibility (cogmux provides a sandbox interface, not an implementation).
- Shared context store supports redaction policies (regex + field-level masks).

## Testing & evaluation

- **Unit:** Per-tier tests with mock LLM clients and deterministic tool stubs.
- **Integration:** End-to-end agent sessions against recorded scenarios; assert latency SLAs and fallback behavior.
- **Benchmark suite:** Standard tasks (conversation, tool-use, navigation) measured on latency, cost, and task completion rate vs. monolithic baseline.
- **Chaos:** Inject model timeouts, tool failures, and bus congestion; verify graceful degradation.

## Existing implementations

### Official HLA implementation

- **[HosnLS/Hierarchical-Language-Agent](https://github.com/HosnLS/Hierarchical-Language-Agent)** — ★45, Python/PyTorch, last updated Jan 2024. Official AAMAS 2024 paper code. Three-tier architecture (GPT-4 Slow Mind, Llama-2-13B Fast Mind, reactive Executor) but tightly coupled to Overcooked domain. Research prototype — no packaging, no tests, no CI. Effectively unmaintained. 9 forks, none with meaningful divergence.

### Closest research successors (dual-process / fast-slow agents)

- **[sjtu-marl/DPT-Agent](https://github.com/sjtu-marl/DPT-Agent)** — ★59, Python, MIT, ACL 2025 Main. System 1 (FSM + code-as-policy) + System 2 (LLM with Theory of Mind). Same Overcooked domain. Most direct successor to HLA with asynchronous reflection. Research code, not a general framework.
- **[SwiftSage/SwiftSage](https://github.com/SwiftSage/SwiftSage)** — ★326, Python, NeurIPS 2023 Spotlight. Most popular fast/slow agent (Swift/Feedback/Sage tiers, V2 uses Llama-3.1). ScienceWorld-focused. V2 in beta, inactive since Oct 2024.
- **Talker-Reasoner (Google DeepMind)** — Paper only (arXiv:2410.08328). Talker (System 1) + Reasoner (System 2). High-profile conceptual framework, no open-source code.
- **DUMA (Dual-Mind Agent)** — Paper only (arXiv:2310.18075). Fast/Slow Mind for conversational AI. No code released.

### Production-grade frameworks with hierarchical patterns (not dual-process specific)

- **[langroid/langroid](https://github.com/langroid/langroid)** — ★4,024, Python, MIT. Hierarchical task delegation, model-agnostic. Production-ready but no explicit fast/slow cognitive tiers.
- **[ulab-uiuc/LLMRouter](https://github.com/ulab-uiuc/LLMRouter)** — ★1,851, Python, MIT. 16+ routing methods for cost-aware LLM selection. Implements query-level routing, not cognitive architecture.
- **[kyegomez/swarms](https://github.com/kyegomez/swarms)** — ★6,733, Python, Apache 2.0. Hierarchical agent swarms. General multi-agent orchestration, not dual-process.

### Gap assessment

No production-grade, general-purpose framework implements the three-tier cognitive architecture (frontier LLM → lightweight LLM → reactive policy) from the HLA paper. All existing implementations are domain-specific research prototypes. cogmux would be the first production SDK for this pattern.

## Rollout plan

- **M0 (5 wk):** Core runtime — three-tier execution, async bus with dead-letter handling, Pydantic message schemas, goal versioning/preemption, single model provider (Anthropic).
- **M1 (4 wk):** Replay/debug recording and playback, Slow Mind health monitoring, multi-provider support (OpenAI, Google Gemini, Ollama), fallback chains.
- **M2 (3 wk):** Observability (OTel spans + metrics), MCP tool integration, CLI.
- **M3 (4 wk):** Extension point interface (`cogmuxRunnable` protocol), benchmark suite, documentation, PyPI release.
- **M4 (4 wk):** LangChain/LangGraph/CrewAI adapters (P1), 3 early-adopter pilots.
- **M5 (6 wk):** Shared context store, cost budgeting, tier promotion/demotion, optional Rust executor (`cogmux[rust]`).

## Open engineering questions

- **Executor language boundary:** Rust via PyO3 adds build complexity (manylinux wheels). Alternative: pure Python executor with optional Rust acceleration. Lean toward pure Python default + optional `cogmux[rust]` extra.
- **Bus implementation:** asyncio.Queue works for single-process; multi-process or distributed agents need Redis/NATS. Start single-process; design bus interface for pluggable backends.
- **Goal versioning semantics:** Should Fast Mind always re-plan on goal version bump, or only when goal diff exceeds a threshold? Start with always-replan; optimize later.
- **CrewAI adapter depth:** How much of the internal tier state should be visible to the crew orchestrator? Lean toward opaque — the crew sees a single agent, not three tiers.

## Reviewer comments

- The Rust/PyO3 Executor is over-engineered for v1 and the open questions section already concedes this. A pure-Python executor with `asyncio` will meet the <10ms P99 target for tool dispatch (the actual latency is in the tool call, not the dispatch). Ship pure Python in M0; gate the Rust path behind `cogmux[rust]` in M2 only if profiling shows Python dispatch is the bottleneck. This also eliminates manylinux wheel complexity that will block early adopters on macOS/ARM.

- The `asyncio.Queue`-based bus has no dead-letter or poison-pill handling. If a malformed `MacroAction` crashes the Executor repeatedly, the bus silently drops or re-delivers it forever. Add a `max_retries` + dead-letter channel to the bus interface, and specify what happens when all three tiers are healthy but the bus itself is backpressured (the design says "drop-oldest for observations" but does not specify behavior when the goal channel blocks and the Executor is starved of new goals).

- PRD P0 lists Google (Gemini) as a required model provider; the dev design's tech stack and scope section omit it entirely (only Anthropic, OpenAI, Ollama). Either add `google-genai` to the provider list or explicitly defer it to M1 with justification. The PRD reviewer also recommended demoting framework adapters (LangChain/LangGraph/CrewAI) from P0 to P1 — the dev design ignores this and schedules them in M2. Align: either accept the PRD reviewer's recommendation and label adapters as P1 in scope, or rebut it here.

- The data model is missing an `Observation` schema. Flow 1 and the SDK example both return `cogmux.Observation`, but it is not defined in the Data Model section. At minimum it needs `id`, `source_action_id`, `tier`, `data` (structured payload), and `timestamp`. Also, `Goal.constraints` is typed as `dict[str, Any]` — this is too loose for structured output parsing and will break the replay diff view. Define a `GoalConstraints` model or at least a `TypedDict`.

- Goal preemption (Flow 3) has a race condition: between "Bus broadcasts preemption signal" (step 2) and "Fast Mind abandons in-flight MacroAction" (step 3), the Executor may receive and execute new atomic actions from the stale MacroAction. The design says "Executor drains current atomic action, then switches" but does not specify how the Executor knows the current MacroAction is invalidated — it only subscribes to the action channel, not the goal channel. Add an explicit `goal_version` field on `AtomicAction` and have the Executor check it before dispatch.

- The rollout compresses M0–M4 into 18 weeks, then adds M5 at 6 weeks for shared context and cost budgeting. For a 2–3 person team this is aggressive: M0 alone (three-tier runtime + async bus + Pydantic schemas + Anthropic provider) is a substantial 4-week sprint. More critically, the PRD reviewer recommended elevating Replay/Debug to P0, but the dev design schedules it in M3 (week 11–14). If replay is P0, it belongs in M1 at latest — debugging without replay will cripple early-adopter onboarding.

- The design is silent on Slow Mind failure modes. Flow 2 handles Fast Mind timeouts; no flow covers Slow Mind crashes, infinite loops in the strategy LLM call, or model provider outages during Slow Mind deliberation. Since Slow Mind sets goals, a stuck Slow Mind means the entire agent operates on a stale goal indefinitely with no alert. Add a Flow 5: "Slow Mind health check" — heartbeat on the goal channel, configurable staleness threshold, and automatic escalation (e.g., emit a degraded-mode OTel event and optionally promote Fast Mind to goal-setting with a constrained prompt).
