# cogmux — Hierarchical Language Agent Framework

## Problem

LLM-powered agents face a fundamental tradeoff: deep reasoning requires large, slow models, while real-time human-AI collaboration demands sub-second responses. Current agentic frameworks (LangChain, CrewAI, AutoGen, LangGraph) treat each agent as a monolithic LLM call — a single model handles both strategic planning and moment-to-moment execution. This creates three pain points:

1. **Latency walls** — A single GPT-4/Claude Opus call takes 2–10s; real-time applications (gaming copilots, surgical assistants, live customer support, warehouse cobots) need ≤200ms reaction loops.
2. **Cost explosions** — Routing every micro-decision through a frontier model is prohibitively expensive at scale. A customer-service copilot handling 10k concurrent sessions at $15/M output tokens burns >$50k/day.
3. **Architectural fragility** — Without separation of concerns, a planning hallucination instantly becomes an execution error. There is no cognitive "firewall" between strategic reasoning and tactical action.

The HLA paper (AAMAS 2024, Liu et al.) demonstrated that decomposing agent cognition into three tiers — a deliberative Slow Mind, a tactical Fast Mind, and a reactive Executor — dramatically improves both cooperation quality and response speed in human-AI teaming. **No production-grade SDK exists to operationalize this pattern.**

## Target users

- **AI application developers** building real-time copilots, game AI, robotics control, or conversational agents who need sub-second response while retaining strategic reasoning.
- **Platform teams at AI-native companies** who orchestrate multiple agents and need a tiered execution model to control cost and latency.
- **Robotics and IoT engineers** integrating LLM reasoning with real-time control loops (ROS2, industrial PLCs).
- **Game studios** building adaptive NPC AI that reasons about player intent but acts in real time.

## Landscape

| Product / Framework | Approach | Gap |
|---|---|---|
| LangChain / LangGraph | Single-model agent loops, graph-based orchestration | No tiered cognition; every node pays full LLM latency |
| CrewAI | Role-based multi-agent | Agents are peers, not hierarchical; no fast/slow split |
| AutoGen | Conversation-based multi-agent | No latency-aware scheduling or reactive fallback |
| Semantic Kernel | Plugin-based orchestration | Single planner; no executor tier |
| ROS2 + LLM bridges | Robotics-specific, ad hoc | No standardized cognitive architecture |
| HuggingFace smolagents | Lightweight agents | Single-tier, no hierarchical decomposition |

**Gap:** No framework provides an opinionated, production-ready three-tier cognitive architecture with built-in latency routing, cost optimization, and tier-specific observability.

## Core idea

cogmux is an open-source SDK (Python-first, with Rust core for the executor runtime) that implements a **three-tier cognitive architecture** for building responsive, cost-efficient AI agents:

- **Slow Mind** — A frontier LLM (Claude Opus, GPT-4, Gemini Ultra) handles strategic reasoning, goal decomposition, and natural-language interaction. Called infrequently (seconds-scale).
- **Fast Mind** — A lightweight LLM (Claude Haiku, GPT-4o-mini, Phi-3, local models) translates strategic goals into macro-action sequences. Called frequently (sub-second).
- **Executor** — A reactive policy engine (rule-based, small RL policy, or scripted) executes atomic actions at millisecond scale. No LLM in the loop.

The SDK provides the communication bus between tiers, tier-aware routing, fallback chains, and unified observability — so developers define *what* each tier does, not *how* they coordinate.

## Functional requirements

### P0 — Launch (MVP)

- **Three-tier runtime:** Declarative definition of Slow Mind, Fast Mind, and Executor with typed message passing between tiers.
- **Tier-aware LLM routing:** Automatic model selection per tier; configurable model mapping (e.g., Slow Mind → Claude Opus, Fast Mind → Claude Haiku).
- **Async communication bus:** Slow Mind decisions propagate to Fast Mind without blocking Executor. Backpressure handling when Slow Mind is deliberating.
- **Fallback chains:** If Fast Mind latency exceeds threshold, Executor falls back to cached macro-action or safe default.
- **Framework integrations:** First-class adapters for LangChain, LangGraph, and CrewAI — cogmux agents can participate as nodes in existing graphs.
- **Python SDK:** `pip install cogmux`. Decorator-based tier definition with sensible defaults.
- **Observability:** Per-tier latency, cost, and token tracking via OpenTelemetry. Tier transition traces.
- **Model provider support:** Anthropic, OpenAI, Google, Ollama/vLLM (local models for Fast Mind and Executor RL policies).

### P1 — Adoption accelerators

- **Macro-action library:** Pre-built action vocabularies for common domains (conversational, tool-use, navigation).
- **Tier promotion/demotion:** Runtime escalation of decisions from Fast Mind to Slow Mind when uncertainty exceeds threshold; demotion of routine decisions from Slow Mind to Fast Mind after learning.
- **MCP tool integration:** Executor and Fast Mind can invoke MCP-compatible tools; Slow Mind can plan tool sequences.
- **Shared context store:** A structured memory layer (Meridian-compatible) accessible by all tiers with tier-appropriate read/write permissions.
- **Cost budgeting:** Per-session and per-tier budget caps with graceful degradation (drop to cheaper model, not crash).
- **Replay and debugging:** Record full tier interaction traces; replay with different model configurations.

### P2 — Differentiation

- **Adaptive tier boundaries:** ML-driven routing that learns which decisions can safely be delegated to cheaper tiers over time.
- **Multi-agent cogmux:** Multiple cogmux agents coordinating, each with independent three-tier stacks, sharing a consensus layer for joint planning.
- **Real-time streaming:** WebSocket/gRPC streaming for Executor outputs; optimized for game loops and robotics control frequencies (30–1000 Hz).
- **Edge deployment:** Fast Mind + Executor packaged for edge devices (ONNX Runtime, TensorRT); Slow Mind remains cloud-hosted.
- **Benchmark suite:** Standardized benchmarks measuring cooperation quality, response latency, and cost across tiered vs. monolithic architectures.

## Non-functional requirements

- Executor loop P99 latency <10ms (no LLM in path).
- Fast Mind response P95 <500ms.
- Slow Mind does not block Executor; maximum tolerable Slow Mind latency before fallback: configurable (default 5s).
- SDK footprint <50MB installed; Executor runtime <10MB standalone.
- Thread-safe and async-native (asyncio/tokio).

## Leveraging existing frameworks and SDKs

cogmux is designed as a **complement, not a competitor** to existing agentic ecosystems:

- **Anthropic SDK / Claude API:** Native integration for Slow Mind (Opus with extended thinking) and Fast Mind (Haiku with prompt caching). Leverages streaming, tool use, and batch APIs.
- **LangChain/LangGraph:** cogmux agents expose a `Runnable` interface, plugging directly into LangChain chains and LangGraph state machines. A LangGraph node can contain an entire cogmux three-tier stack.
- **CrewAI:** cogmux agents implement the CrewAI `Agent` protocol, allowing them to serve as crew members with internal tiered cognition invisible to the crew orchestrator.
- **MCP (Model Context Protocol):** Executor and Fast Mind tiers consume MCP tools natively. cogmux itself can be exposed as an MCP server for other agents.
- **OpenTelemetry:** All tier transitions, model calls, and tool invocations emit OTel spans, compatible with existing observability stacks (Datadog, Grafana, Jaeger).
- **Ollama / vLLM / HuggingFace TGI:** Fast Mind tier supports local model inference for cost-sensitive or air-gapped deployments.
- **Pydantic / instructor:** Structured output parsing at each tier boundary ensures type safety across the cognitive hierarchy.

This "plays well with others" strategy lowers adoption barriers — teams don't need to rewrite existing agent infrastructure; they add cogmux inside individual agents where latency and cost matter.

## Architecture sketch

```
                    ┌─────────────────────────────────┐
                    │         cogmux Agent               │
                    │                                  │
 User / Env ──────▶│  ┌───────────┐   Strategic       │
                    │  │ Slow Mind │   goals, plans    │
                    │  │ (Opus)    │──────────┐        │
                    │  └───────────┘          │        │
                    │                         ▼        │
                    │  ┌───────────┐   Macro-actions   │
                    │  │ Fast Mind │──────────┐        │
                    │  │ (Haiku)   │          │        │
                    │  └───────────┘          ▼        │
                    │  ┌───────────┐   Atomic actions  │
                    │  │ Executor  │──────────────────▶│──▶ Tools / Env
                    │  │ (Policy)  │                   │
                    │  └───────────┘                   │
                    │                                  │
                    │  ┌──────────────────────────┐    │
                    │  │ Communication Bus (async) │    │
                    │  │ + Shared Context Store    │    │
                    │  └──────────────────────────┘    │
                    └─────────────────────────────────┘
```

## Key risks

- **Tier boundary design is hard** — Developers must decide what constitutes a "strategic" vs. "tactical" vs. "reactive" decision. Poor decomposition negates the latency benefit. Mitigation: provide domain-specific templates and an adaptive routing layer (P2).
- **Consistency across tiers** — Slow Mind may update strategy while Fast Mind is mid-execution. Stale macro-actions could conflict with new goals. Mitigation: version-stamped goals with graceful preemption.
- **Model heterogeneity** — Tiers using different model families may have subtly different capabilities (tool calling formats, context lengths). Mitigation: unified abstraction layer with per-model adapters.
- **Adoption friction** — Developers comfortable with single-agent loops may not see the value until latency/cost hits production. Mitigation: benchmark suite and clear migration guide.

## Success metrics

- **Adoption:** 2,000 GitHub stars and 500 weekly PyPI downloads within 6 months of launch.
- **Latency:** Executor loop <10ms P99, Fast Mind <500ms P95 in reference benchmarks.
- **Cost reduction:** Demonstrate ≥60% cost reduction vs. monolithic Opus agent on equivalent task quality.
- **Framework integration:** Working examples with LangChain, LangGraph, CrewAI, and at least one robotics framework (ROS2 or PyBullet).
- **Developer satisfaction:** >4.0/5.0 in post-pilot surveys from 3 early adopter teams.

## Usage scenarios

1. **Real-time customer support copilot:** Slow Mind analyzes conversation history and customer profile to set strategy ("apologize, offer 20% credit, escalate if unresolved in 2 turns"). Fast Mind generates response templates and selects macro-actions ("pull order history", "draft apology"). Executor sends the actual API calls and renders the response in <200ms.

2. **Adaptive game NPC:** Slow Mind reasons about player skill level and narrative arc every 30 seconds. Fast Mind selects combat macro-actions (flank, defend, retreat) every second. Executor runs pathfinding and animation triggers at 60 Hz.

3. **Warehouse cobot coordinator:** Slow Mind plans optimal pick routes for the shift. Fast Mind adjusts routes when a new order arrives. Executor controls motor commands and collision avoidance at 100 Hz.

4. **AI pair programmer:** Slow Mind understands the feature request and plans implementation across files. Fast Mind generates code completions as the developer types. Executor handles syntax highlighting and linting in real time.

## Impact

- **Developers** get a principled way to build responsive AI agents without ad hoc latency hacks.
- **End users** experience AI that feels instantaneous while making intelligent decisions.
- **The ecosystem** gains a reusable architectural pattern — the "Model-View-Controller" equivalent for agentic AI.
- **Research community** gets an open-source implementation of the HLA pattern, enabling reproducible experiments and extensions.

---

## Reviewer comments

- The competitive landscape omits several direct competitors. Google DeepMind's SIMA and NVIDIA's Voyager both implement hierarchical agent cognition for game and robotics domains. Microsoft's Autogen v0.4 introduced "nested chat" patterns that approximate tiered decomposition. DSPy's modular pipeline approach also competes for the "structured agent orchestration" niche. Adding these would strengthen the gap claim and force sharper differentiation on what cogmux uniquely provides — the async communication bus and tier-aware routing.

- The "MVC for agentic AI" analogy in the Impact section is evocative but underexploited. The PRD should lead with it in the Core Idea section and carry the metaphor through: Slow Mind as the Model (state and reasoning), Fast Mind as the Controller (request routing and coordination), Executor as the View (rendering actions to the environment). This would give developers an immediate mental model and make the three-tier architecture feel inevitable rather than novel.

- Framework integrations (LangChain, LangGraph, CrewAI adapters) are listed as P0 but should be P1. The MVP needs to prove the core three-tier runtime delivers on latency and cost promises before investing in compatibility shims. Shipping adapters at launch splits engineering focus and couples the release timeline to third-party API stability. Move them to P1 and replace with a "bring your own Runnable" extension point at P0 that lets motivated early adopters write adapters themselves.

- The success metrics conflate adoption vanity metrics with product-quality metrics. GitHub stars and PyPI downloads tell you about marketing, not product-market fit. Add retention-oriented metrics: percentage of installers who deploy to production within 90 days, number of community-contributed tier templates or executor policies, and repeat usage rate among early adopter teams. The cost reduction target (60%) also needs a defined benchmark workload — "equivalent task quality" is not measurable without specifying the task and the quality rubric.

- The risk section is silent on the hardest operational risk: debugging across tiers. When an agent produces a bad outcome, the developer must trace causality from Executor action back through Fast Mind macro-action selection to Slow Mind strategy — across different models, different latency windows, and potentially different context windows. The P0 observability requirement (OpenTelemetry spans) is necessary but not sufficient; add a risk entry for cross-tier debugging complexity and consider elevating the Replay and Debugging feature from P1 to P0.

- The usage scenarios are imaginative but unevenly grounded. The customer support copilot (scenario 1) is compelling and immediately buildable. The warehouse cobot scenario (scenario 3) requires motor control integration, real-time safety certification, and hardware-in-the-loop testing that is far beyond an SDK's scope — it reads as aspirational rather than actionable. Replace it with a scenario closer to the SDK's actual capabilities at launch, such as a multi-step research assistant or a monitoring/alerting triage agent, and move the robotics scenario to a "Future Vision" callout.

- The PRD does not address state management between Slow Mind invocations. If Slow Mind is called infrequently (every 30 seconds in the game NPC scenario), the Fast Mind and Executor must operate on a stale strategic context during that window. The Consistency risk acknowledges this but the functional requirements do not specify how strategic state is versioned, how Fast Mind detects goal staleness, or what happens when Slow Mind's new plan invalidates in-flight macro-actions. Add a P0 requirement for a goal versioning and preemption protocol with clear semantics (e.g., soft preemption vs. hard abort).
