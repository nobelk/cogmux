# Validation — Foundation

Phase: 0 · Date: 2026-05-24

## Exit criteria (from roadmap)

The roadmap does not specify explicit exit criteria for Phase 0. The implicit exit is: skeleton project with data model and a minimal end-to-end proof that three tiers can communicate. Since the full demo (task 0.5) and comprehensive test suite (task 0.6) are deferred, exit criteria are scoped to the four in-scope task groups. The integration test in plan task 4.5 serves as the end-to-end proof.

## Merge checklist

- [ ] All task groups in `plan.md` complete
- [ ] Tests green (`uv run pytest`) — basic schema validation and bus round-trip tests pass, including the integration test that wires all three tier stubs
- [ ] CI pipeline passing — GitHub Actions runs ruff check, ruff format --check, mypy, and pytest across Python 3.11–3.13
- [ ] Docs updated — README reflects project structure, setup instructions (`uv sync`, `pytest`), and architecture overview pointing to `specs/`

## How to verify

Run the full CI-equivalent locally:

```bash
uv sync
ruff check .
ruff format --check .
mypy --strict src/
pytest -x
```

The test suite should include an integration test that wires all three tier stubs through the Bus: Slow Mind publishes a Goal → Fast Mind consumes it and emits AtomicActions → Executor consumes AtomicActions and publishes Observations. This test validates the end-to-end message flow without LLM calls.

Confirm the GitHub Actions workflow file exists at `.github/workflows/ci.yml` and includes the Python 3.11, 3.12, 3.13 matrix with all four checks (ruff check, ruff format --check, mypy, pytest).
