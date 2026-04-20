<!-- AGENT_TASK_START: task-1-transport-resolver.md -->

# Task 1 — Transport Defaults + Resolver

## Agent Instructions

You are a software engineer implementing one module of a larger system. Scope strictly limited to this task.

**Pre-work:**
1. Read `docs/exec-plans/active/agent-capabilities/llm-transport-hardening/plan.md`.
2. Read `services/worker-service/executor/providers.py` — this task creates the resolver the rewritten `providers.py` (Task 2) will consume.

**Post-work:** Run `services/worker-service/.venv/bin/python -m pytest services/worker-service/tests/test_transport_resolver.py -v`. Update `progress.md` row 1 → Done.

## Context

Introduce a small pure stdlib module that returns `(connect_timeout_s, read_timeout_s, max_output_tokens)` given an agent's config. Task 2 consumes it in the rewritten `create_llm`. Task 3 (Java API) imports the same numeric bounds via a file-parse drift test.

## Shared contract

- **Platform defaults (not customer-tunable globally):**
  - `connect_timeout_s = 10`
  - `read_timeout_s = 300` (whole-response budget; `ainvoke` is one HTTP read, so this covers the full call)
  - `max_output_tokens = 16_384`
- **Per-agent overrides (optional, from `agent_config.llm_transport`):**
  - `connect_timeout_s ∈ [1, 60]`
  - `read_timeout_s ∈ [10, 900]`
  - `max_output_tokens ∈ [256, 200_000]`
- The upper bound on `read_timeout_s` (900s = 15min) is intentionally larger than the submit-form default `task_timeout_seconds` (60s dev / typically much higher in production). The task-level timeout is authoritative via `_await_or_cancel`: it cancels the in-flight `ainvoke` operation regardless of how large `read_timeout_s` is. So an operator who legitimately needs a long-running LLM call sets BOTH `task_timeout_seconds` and `read_timeout_s` appropriately; they do not conflict. Add a comment in the module docstring explaining this.
- **Out-of-bounds overrides** are clamped to the nearest bound AND emit a `WARN` log (`transport.override_clamped`).
- **Non-numeric overrides** (string, NaN, Infinity, bool) fall back to defaults with a `WARN` log (`transport.override_invalid_type` / `transport.override_non_finite`).
- **Returned `LLMTransportConfig` is a frozen dataclass.**
- **Signature accepts the resolved provider and model** (for future per-provider tuning) but v1 ignores both arguments; document in docstring.

## Affected files

- `services/worker-service/executor/transport.py` (new)
- `services/worker-service/tests/test_transport_resolver.py` (new)

## Dependencies

- **Must complete first:** none.
- **Provides output to:** Task 2 (consumes `LLMTransportConfig`), Task 3 (mirrors the six `*_MIN` / `*_MAX` constants in Java, enforced by a file-parse drift test).

## Implementation Specification

### Module `executor/transport.py`

Exports (all at module top level — Task 3's Java drift test regex-matches bare `NAME: type = literal` lines):

- `@dataclass(frozen=True) class LLMTransportConfig` with `connect_timeout_s: float`, `read_timeout_s: float`, `max_output_tokens: int`.
- Default constants:
  - `DEFAULT_CONNECT_TIMEOUT_S = 10`
  - `DEFAULT_READ_TIMEOUT_S = 300`
  - `DEFAULT_MAX_OUTPUT_TOKENS = 16_384`
- Bounds constants (shared with Task 3):
  - `CONNECT_TIMEOUT_S_MIN = 1`, `CONNECT_TIMEOUT_S_MAX = 60`
  - `READ_TIMEOUT_S_MIN = 10`, `READ_TIMEOUT_S_MAX = 900`
  - `MAX_OUTPUT_TOKENS_MIN = 256`, `MAX_OUTPUT_TOKENS_MAX = 200_000`
- A comment immediately above the bounds block naming `LlmTransportBoundsDriftTest` (Task 3, Java side) so future refactors know about the cross-language contract.

Function:

```python
def resolve_transport(
    agent_config: Mapping[str, Any] | None,
    *,
    provider: str,
    model: str,
) -> LLMTransportConfig
```

Behaviour:
- `agent_config is None`, or `agent_config.get("llm_transport") is None` → return defaults verbatim, no warning.
- For each of the three override fields:
  - Non-numeric / NaN / Infinity / bool → WARN + fallback to default.
  - In-range → use override.
  - Out-of-range → clamp + WARN.
- Unknown keys inside `llm_transport` silently ignored (forward-compat with future Task 3 fields).
- `provider` / `model` accepted but unused in v1; documented in docstring.

### Tests `tests/test_transport_resolver.py`

- Defaults returned when `agent_config` is `None`, `{}`, or `{"llm_transport": None}`.
- Each override applied in-range; both low and high bounds inclusive.
- Each override clamped + WARN when out of range (parametrize high and low).
- Non-numeric / NaN / Infinity / bool overrides fall back to default + WARN.
- Frozen dataclass: assignment raises `dataclasses.FrozenInstanceError`.
- Unknown keys inside `llm_transport` silently ignored (no warning).

## Acceptance Criteria

- [ ] Resolver returns documented defaults when `agent_config` lacks `llm_transport`.
- [ ] Each override clamped / rejected per bounds with structured WARN log.
- [ ] `LLMTransportConfig` is frozen.
- [ ] All tests pass via `services/worker-service/.venv/bin/python -m pytest services/worker-service/tests/test_transport_resolver.py -v`.
- [ ] Module has no deps outside stdlib (no langchain, no asyncpg).
- [ ] Bounds and default constants at module top level (bare `NAME: type = literal`) so Task 3's drift-test regex can parse them.

## Out of Scope

- Wiring the resolver into `providers.py` (Task 2).
- API-side validation (Task 3).

<!-- AGENT_TASK_END -->
