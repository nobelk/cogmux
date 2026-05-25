# Requirements — Foundation

Phase: 0
Date: 2026-05-24
Branch: spec/phase-0-foundation

## Context

cogmux's three-tier architecture depends on a correct and performant communication bus and a precise data model — every subsequent phase builds on these primitives. This phase establishes the project skeleton, locks in the Pydantic v2 message schemas that define inter-tier contracts, implements the async bus, and creates tier runtime stubs that prove the wiring works. The mission's "progressive complexity" principle (specs/mission.md §4) requires that the foundation be minimal but extensible; the "latency is a feature" principle (§1) means bus design must account for backpressure from day one.

## Scope

- **Project scaffolding:** `pyproject.toml` (uv), ruff config, mypy strict config, pytest + pytest-asyncio setup, GitHub Actions CI (Python 3.11–3.13 matrix with lint, typecheck, test).
- **Pydantic v2 message schemas:** `Goal`, `MacroAction`, `AtomicAction`, `Observation`, `TierConfig`, `AgentConfig` — all inter-tier message types with full type annotations and validation.
- **Communication bus:** `asyncio.Queue`-based channels (goal, action, observation) with configurable `maxsize` and backpressure semantics.
- **Tier runtime stubs:** Slow Mind, Fast Mind, Executor as async tasks that read/write from bus channels — no LLM calls, just the task lifecycle and channel plumbing.

## Non-goals

- **Hardcoded demo (roadmap task 0.5):** Deferred entirely. The integration test in plan task 4.5 validates bus round-trip; a standalone demo script is not required for this phase.
- **Unit test suite (roadmap task 0.6):** Deferred as a standalone deliverable. Basic tests confirming schema validation and bus FIFO ordering will exist as part of CI, but the comprehensive test suite (backpressure edge cases, ordering guarantees) ships in a follow-up.
- **LLM provider integration:** Phase 1.
- **Decorator API (`@cogmux.slow_mind`, etc.):** Phase 2.
- **Goal preemption protocol:** Phase 3 (schemas will include `version` field to avoid breaking changes, but preemption logic is out of scope).
- **Dead-letter channel:** Deferred entirely to Phase 3. No stub in Phase 0 — adding a channel later is trivial.

## Decisions

- **Stack as-is:** uv, ruff, mypy (strict), pytest, pydantic v2 — no new dependencies beyond what `specs/tech-stack.md` specifies for Phase 0.
- **Ruff line-length:** 88 (black default). Locked in for this phase.
- **MacroAction is bus-internal:** Fast Mind decomposes Goals into MacroActions internally, then emits only `AtomicAction` messages on the action channel. MacroAction never appears on the bus.
- **New dependency to flag:** TBD — if any dependency not listed in tech-stack.md is needed during implementation, it must be explicitly called out and justified before merging.

### Open questions

- [x] **Bus channel typing:** Strongly typed — each channel carries exactly one message type (`Channel[Goal]`, `Channel[AtomicAction]`, `Channel[Observation]`). MacroAction is internal to Fast Mind, not a bus-level message.
- [ ] **Backpressure semantics:** When a channel is full, should `put()` block (await), raise an exception, or drop the message? The choice affects Executor latency guarantees.
- [ ] **TierConfig mutability:** Should `TierConfig` be frozen after construction, or support runtime updates (e.g., changing `max_latency_ms` on the fly)?
- [ ] **Message ID generation:** UUID4 for message IDs, or a more compact scheme (ULID, nanoid)? Affects log readability and storage overhead.

## References

- `specs/mission.md` — Vision, guiding principles (§1 latency, §4 progressive complexity, §6 pure Python first)
- `specs/tech-stack.md` — Core dependencies, architecture constraints (async-native, no LLM in Executor)
- `specs/roadmap.md` — Phase 0
