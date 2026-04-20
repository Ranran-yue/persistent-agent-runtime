<!-- AGENT_TASK_START: task-3-api-llm-transport-config.md -->

# Task 3 — API: `agent_config.llm_transport` Sub-Object

## Agent Instructions

You are a software engineer implementing one module of a larger system. Scope strictly limited to this task.

**Pre-work:**
1. Read `docs/exec-plans/active/agent-capabilities/llm-transport-hardening/plan.md`.
2. Read `services/api-service/src/main/java/com/persistentagent/api/model/request/AgentConfigRequest.java`.
3. Read `services/api-service/src/main/java/com/persistentagent/api/model/request/MemoryConfigRequest.java` (pattern for a nested optional sub-object with snake-case JSON keys — mirror this style).
4. Read `services/api-service/src/main/java/com/persistentagent/api/service/ConfigValidationHelper.java`.
5. Read `services/api-service/src/main/java/com/persistentagent/api/service/AgentService.java` (`canonicalizeConfig`).
6. Read `services/worker-service/executor/transport.py` (Task 1's output) — the numeric bounds to mirror.

**Post-work:** Run the Gradle tests in `services/api-service/` covering `AgentConfigValidationTest`, `AgentServiceTest`, and the new drift test. Update `progress.md` row 3 → Done.

## Context

Adds an optional nested `llm_transport` sub-object to `agent_config` so operators can override `connect_timeout_s`, `read_timeout_s`, and `max_output_tokens` per-agent. No worker-side change in this task (Task 2 already reads `agent_config.llm_transport` via the resolver). Matches the exact shape already used for Track 7's `context_management` sub-object.

## Shared contract

- Three optional fields, all nullable:
  - `connect_timeout_s` (Double, JSON key `connect_timeout_s`), range `[1, 60]`. **Honored only by the Bedrock provider today** (OpenAI + Anthropic inherit httpx's default); field-level JavaDoc must say so.
  - `read_timeout_s` (Double, JSON key `read_timeout_s`), range `[10, 900]`.
  - `max_output_tokens` (Integer, JSON key `max_output_tokens`), range `[256, 200_000]`.
- **Bounds mirror Task 1 exactly.** Create a Java constants class `LlmTransportBounds` whose six values match the Python constants (`CONNECT_TIMEOUT_S_MIN/MAX`, `READ_TIMEOUT_S_MIN/MAX`, `MAX_OUTPUT_TOKENS_MIN/MAX`). Ship a build-time test that regex-parses `services/worker-service/executor/transport.py` from test resources and asserts numeric equality. Drift fails the build.
- **Canonicalisation:** `AgentService.canonicalizeConfig` round-trips the sub-object as-is. Absent sub-object stays absent in the persisted JSON; present sub-object preserves field-level absence (an absent `read_timeout_s` is NOT silently filled with the default).
- **Unknown-property rejection:** verify by grep before relying on it. `services/api-service/src/main/java/.../config/JacksonConfig.java` is the `@Primary` `ObjectMapper` bean and doesn't disable Jackson's library-default `FAIL_ON_UNKNOWN_PROPERTIES = true`. Document the reliance (NOT overstate it: it's a library default, not an explicit Spring config) and cover with a regression test that sends `{"llm_transport": {"unknown_field": 1}}` and expects 400.
- No worker behavior changes in this task — the resolver already reads the same shape.

## Affected files

- `services/api-service/src/main/java/com/persistentagent/api/model/request/LlmTransportConfigRequest.java` (new)
- `services/api-service/src/main/java/com/persistentagent/api/config/LlmTransportBounds.java` (new)
- `services/api-service/src/main/java/com/persistentagent/api/model/request/AgentConfigRequest.java` (modify — add typed `llmTransport` field)
- `services/api-service/src/main/java/com/persistentagent/api/service/ConfigValidationHelper.java` (modify — add `validateLlmTransportConfig`, invoke from `validateAgentConfig`)
- `services/api-service/src/main/java/com/persistentagent/api/service/AgentService.java` (modify — `canonicalizeConfig` round-trip)
- `services/api-service/src/test/java/com/persistentagent/api/config/LlmTransportBoundsDriftTest.java` (new)
- Extend `AgentConfigValidationTest` / `AgentServiceTest` (or the established equivalents — follow Track 7's `context_management` precedent).

## Dependencies

- **Must complete first:** Task 1 (the drift test reads `transport.py`).
- **Provides output to:** —

## Implementation Specification

### New record: `LlmTransportConfigRequest`

Mirror `MemoryConfigRequest` / `ContextManagementConfigRequest` style:

```java
public record LlmTransportConfigRequest(
    @JsonProperty("connect_timeout_s") Double connectTimeoutS,
    @JsonProperty("read_timeout_s") Double readTimeoutS,
    @JsonProperty("max_output_tokens") Integer maxOutputTokens
) { }
```

JavaDoc on `connectTimeoutS`: "Honored only by the Bedrock provider today; OpenAI and Anthropic inherit the httpx default."

Class-level JavaDoc: note the relationship with `executor/transport.py` bounds and that `LlmTransportBoundsDriftTest` enforces the cross-language contract.

### Modify `AgentConfigRequest`

Add `LlmTransportConfigRequest llmTransport` — nullable, snake_case `@JsonProperty("llm_transport")`, `@JsonInclude(JsonInclude.Include.NON_NULL)`.

### Modify `ConfigValidationHelper.validateAgentConfig`

Add `validateLlmTransportConfig(LlmTransportConfigRequest t)` invoked when `t != null`:

- `connectTimeoutS` (when non-null): must be in `[1, 60]` and finite. Out-of-range / NaN / Infinity → `BadRequestException("connect_timeout_s must be between 1 and 60")`.
- `readTimeoutS` (when non-null): must be in `[10, 900]` and finite. Message `"read_timeout_s must be between 10 and 900"`.
- `maxOutputTokens` (when non-null): must be in `[256, 200000]`. Message `"max_output_tokens must be between 256 and 200000"`.

Use `Double.isFinite` defensively (Jackson rejects NaN/Infinity tokens at parse time by default, but defense in depth).

### Modify `AgentService.canonicalizeConfig`

When the request carries `llmTransport`, preserve it in the persisted JSON exactly as received (no field-level default fills). When absent, omit the key entirely.

### `LlmTransportBounds` constants class

```java
public final class LlmTransportBounds {
    private LlmTransportBounds() {}
    public static final double CONNECT_TIMEOUT_S_MIN = 1;
    public static final double CONNECT_TIMEOUT_S_MAX = 60;
    public static final double READ_TIMEOUT_S_MIN = 10;
    public static final double READ_TIMEOUT_S_MAX = 900;
    public static final int MAX_OUTPUT_TOKENS_MIN = 256;
    public static final int MAX_OUTPUT_TOKENS_MAX = 200_000;
}
```

Class JavaDoc names Python `executor/transport.py` as the source of truth.

### `LlmTransportBoundsDriftTest`

- Walk up from test cwd to repo root (covers Gradle and IDE run modes) and locate `services/worker-service/executor/transport.py`.
- Parse the six `*_MIN` / `*_MAX` constants with a narrow regex `^(\w+_(MIN|MAX)):\s*(?:float|int)\s*=\s*([0-9_.]+)`.
- Assert all six are present (sanity — a regex miss that silently skips would hide drift).
- Assert each value matches the corresponding `LlmTransportBounds.*` field.

### Tests

Extend `AgentConfigValidationTest` (or the nearest precedent):
- Each field at min, max, just-below-min, just-above-max.
- Absent sub-object accepted; empty sub-object accepted (all fields null).
- Unknown field inside `llm_transport` → 400.

Extend `AgentServiceTest` (or equivalent canonicalization test):
- Round-trip a full sub-object.
- Round-trip a partial sub-object (only `read_timeout_s` set).
- Round-trip an absent sub-object → persisted JSON has no `llm_transport` key.

## Acceptance Criteria

- [ ] `POST /v1/agents` and `PUT /v1/agents/{agent_id}` accept `agent_config.llm_transport` with the three documented fields.
- [ ] Out-of-range values are rejected with clear 400 messages.
- [ ] Unknown properties inside `llm_transport` are rejected with 400 (regression-tested).
- [ ] Canonicalisation preserves field-level absence.
- [ ] Field-level JavaDoc on `connectTimeoutS` documents "honored only by Bedrock today."
- [ ] All extended tests pass, including the drift test that reads `executor/transport.py`.
- [ ] Bounds exactly mirror Task 1's Python constants (enforced by drift test).

## Out of Scope

- Worker-side consumption (Task 1 + Task 2 already handle this).
- Console form for editing these fields (deferred to a follow-up after operator demand).
- Per-model defaults — out of scope for this hardening cycle.

<!-- AGENT_TASK_END -->
