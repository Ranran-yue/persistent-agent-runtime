# LLM Transport Hardening — Minimal Plan

> **For agentic workers:** Use `superpowers:subagent-driven-development` to implement this plan task-by-task.

**Tracking issue:** [#85](https://github.com/shenjianan97/persistent-agent-runtime/issues/85) — Worker LLM calls silently drop transport timeouts; slow models occasionally die mysteriously before the task-level timeout budget is up.

**Goal:** Fix the silent-kwarg-drop bug so that the single visible customer-facing timeout (the submit-form "Task timeout (s)" field) actually governs task lifetime, rather than being pre-empted by a hidden HTTP-layer default. Keep the change small: direct per-provider construction with native timeout fields + a `max_tokens` cap + a prompt nudge for long artifacts + a regression test. No streaming, no progress UI, no new DB kinds, no new migration.

---

## Context

The original failure mode: `init_chat_model(timeout=300, max_retries=0)` silently routes unknown kwargs into `model_kwargs`. LangChain prints a one-line warning at startup; the `timeout=300` we thought we set never reaches the HTTP client. What actually runs is whatever the underlying SDK's default is (botocore / httpx), and for some provider combinations that default is much smaller than 300s. Result: a slow-model LLM call dies at the hidden default before the customer's task timeout has a chance to matter.

Customer-facing symptom: submit a task with `task_timeout_seconds=600`, task dead-letters at ~60–300s with a vague `ReadTimeoutError` for no visible reason.

## What we are NOT doing (deliberate scope cuts)

After investigation, these turned out to cost more than they're worth:

- **LLM-call streaming (`astream`).** Provides per-chunk read-timeout reset, but no real-world failure in our deployment came from a single LLM call exceeding a properly-configured whole-response `read_timeout`. With a 300 s read timeout set correctly, `ainvoke` handles every real scenario.
- **Progress events (`llm_stream_progress` / `llm_stream_complete`).** Only meaningful for plain-text streams. Tool_use responses buffer server-side on Anthropic, and OpenAI has no opt-out at all. The UI ends up showing misleading "N chars · M chunks" counters that don't reflect actual progress. Customers already see task status, cost, and elapsed-time ticking up on the task page.
- **Console rolling-progress render.** Follows from the progress events above; no events → no render branch.
- **Fine-grained tool streaming (`eager_input_streaming`), invalid-JSON salvage, O(N) chunk merge, cancel-race helper.** All plumbing for the streaming path we're not taking.
- **Migration 0018 + new `ConversationLogKind` values.** No streaming → no new conversation-log kinds to store.

All tracked as potential follow-ups, not blockers for closing #85.

## Architecture of the minimal fix

1. **`executor/providers.py`** stops using `init_chat_model(...)` for all three providers. Each provider's native chat-model class is constructed directly with its native timeout + retry + max-tokens fields:
   - Bedrock: `boto3.client("bedrock-runtime", config=botocore.Config(connect_timeout, read_timeout, retries))` + `ChatBedrockConverse(client=, max_tokens=)`.
   - OpenAI: `ChatOpenAI(request_timeout=, max_retries=0, max_tokens=)`.
   - Anthropic: `ChatAnthropic(default_request_timeout=, max_retries=0, max_tokens=)`.
   Values come from a single `LLMTransportConfig` resolver that consults agent config first, then platform defaults.
2. **`executor/graph.py`** keeps `ainvoke` (no streaming rewrite). The only change at the call sites is threading `transport=` through to `create_llm(...)`. After the call, check `response.response_metadata` for a max-tokens stop reason and emit a structured `llm.max_tokens_reached` warning (provider-agnostic normaliser handles Bedrock / Anthropic / OpenAI field naming).
3. **`services/api-service/...`** gains a nested `agent_config.llm_transport` sub-object so operators can override `read_timeout_s` / `max_output_tokens` per-agent if a specific slow model needs it. Bounds mirror the worker-side constants via a file-parse drift test.
4. **Default system prompt** (in the platform system-message builder) adds a short directive when `create_text_artifact` is in `allowed_tools`: split long deliverables into multiple smaller artifacts. This is the single highest-leverage behavioural change for preventing pathologically-long single LLM calls in the first place.

---

## Impacted Components

| Component | Path | Change Type |
|---|---|---|
| Transport defaults + resolver | `services/worker-service/executor/transport.py` (new) | new module |
| LLM construction | `services/worker-service/executor/providers.py` (rewrite) | `init_chat_model` → direct per-provider construction |
| Graph call sites | `services/worker-service/executor/graph.py` (modify — both `create_llm` sites pass `transport=`; add max-tokens warning after `ainvoke`) | small diff |
| API request model | `services/api-service/src/main/java/.../request/LlmTransportConfigRequest.java` (new) + `AgentConfigRequest.java` + `ConfigValidationHelper.java` + `AgentService.java` | Java |
| API drift test | `services/api-service/src/test/java/.../LlmTransportBoundsDriftTest.java` (new) | parses `executor/transport.py` |
| Default system prompt | `services/worker-service/executor/graph.py::_build_platform_system_message` | chunked-artifact guidance gated on the artifact tool |
| Tests | `services/worker-service/tests/test_transport_resolver.py`, `test_providers_transport.py`, `test_system_prompts.py`, `test_long_output_no_timeout.py` (regression guard) | new |

No DB migration. No Console changes. No new conversation-log kinds.

---

## Dependency Graph

```
Task 1 (transport.py resolver)
   │
   ▼
Task 2 (providers.py + graph.py call-site wiring)  ──► regression test

Task 3 (API agent_config.llm_transport) ──► must land AFTER Task 1
                                           (bounds drift test reads transport.py)

Task 4 (System prompt chunking guidance) ──► independent
```

Four tasks, two of which can run in parallel. Roughly 300 lines of production code + ~200 lines of tests total.

---

## Acceptance Criteria (mirrors #85 after scope narrowing)

- [ ] Worker startup logs no longer emit any langchain `"... was transferred to model_kwargs"` warning for any of the three providers.
- [ ] Unit tests assert the configured `read_timeout` is actually applied on each provider's native field (Bedrock `client.meta.config.read_timeout`, OpenAI `ChatOpenAI.request_timeout`, Anthropic `ChatAnthropic.default_request_timeout`).
- [ ] `max_tokens` is set on every LLM call; when the model stops with `stop_reason=max_tokens` (normalised across provider field names), a `llm.max_tokens_reached` WARN is logged.
- [ ] Running the original failing prompt from #85 either completes or fails fast with `llm.max_tokens_reached` — never silently dies at a hidden HTTP-layer timeout.
- [ ] `POST /v1/agents` / `PUT /v1/agents/:id` accept `agent_config.llm_transport` with `connect_timeout_s` / `read_timeout_s` / `max_output_tokens` overrides; out-of-range values are rejected 400; bounds match `executor/transport.py` (enforced by drift test).
- [ ] System prompt for agents with `create_text_artifact` in allowed tools includes chunked-output guidance.
- [ ] No `astream`, no `llm_stream_progress` / `llm_stream_complete` kinds, no migration 0018, no Console changes introduced by this branch.

---

## Out of Scope (explicit non-goals)

- **LLM-call streaming and progress UI.** Deferred indefinitely pending a clearer customer need (see §"What we are NOT doing").
- **Per-tool fine-grained Anthropic streaming** (`eager_input_streaming`). Provider-specific, brittle, and unneeded without streaming in the graph.
- **Bedrock-specific beta headers** for tool streaming.
- **Per-chunk cancellation primitive.** The existing `_await_or_cancel` already wraps `ainvoke` against the task cancel event; no separate helper needed.
- **Console UX changes.** Existing duration / cost counters already tick up honestly.

---

## Risks

| Risk | Mitigation |
|---|---|
| A very slow model + long response exceeds the default `read_timeout=300s` and fails | Per-agent override via `agent_config.llm_transport.read_timeout_s` (Task 3). Bounded to [10, 900]. |
| `max_tokens=16_384` default cuts off a legitimate long response | Surface `llm.max_tokens_reached` warning clearly; operator can raise via `agent_config.llm_transport.max_output_tokens` (up to 200_000). Task 4's prompt guidance reduces the chance by encouraging chunked artifacts. |
| Bounds in the API drift from the worker-side constants | `LlmTransportBoundsDriftTest` parses `executor/transport.py` at test time and asserts numeric equality. |
