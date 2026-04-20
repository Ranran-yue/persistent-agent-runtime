<!-- AGENT_TASK_START: task-2-direct-provider-construction.md -->

# Task 2 — Direct Provider-Class Construction + Call-Site Wiring

## Agent Instructions

You are a software engineer implementing one module of a larger system. Scope strictly limited to this task. **No streaming. Keep `ainvoke`.**

**Pre-work:**
1. Read `docs/exec-plans/active/agent-capabilities/llm-transport-hardening/plan.md` — understand why the scope is deliberately narrow (no streaming, no progress events).
2. Read `services/worker-service/executor/providers.py` (entire file, ~45 lines) — the file being rewritten.
3. Read `services/worker-service/executor/transport.py` — Task 1's output, which this task consumes.
4. Read `services/worker-service/executor/graph.py` around the two `providers.create_llm` sites (agent_node near line 850, summarizer near line 1577).

**Post-work:**
1. Restart the worker (`make stop && make start`) and confirm startup logs emit no `"... was transferred to model_kwargs"` warning for any provider.
2. Run `services/worker-service/.venv/bin/python -m pytest services/worker-service/tests/test_providers_transport.py -v`.
3. Update `progress.md` row 2 → Done.

## Context

The original #85 bug was `init_chat_model(timeout=300, max_retries=0)` silently dropping unknown kwargs into `model_kwargs`. Direct per-provider construction with native field names makes the timeout we configure provably the timeout that applies. The whole fix is in this file (plus a small wiring change at two `graph.py` call sites). No streaming rewrite is part of this task — `ainvoke` stays.

## Shared contract

- `create_llm` signature: `create_llm(pool, provider, model_name, temperature, *, transport: LLMTransportConfig) -> BaseChatModel`. Callers must pass `transport=` as a kwarg.
- **Three LLM-construction sites** get wired:
  - `agent_node` in `graph.py` around line 850 — add `resolve_transport(agent_config, provider=provider, model=model_name)` immediately above the `providers.create_llm(...)` call and pass as `transport=`.
  - `_build_summarizer_callable` in `graph.py` around line 1577 — extend the factory's signature to accept `agent_config: dict[str, Any]`, pass that through from its call site (around `graph.py:1233`), then inside the factory call `resolve_transport(agent_config, provider=provider, model=effective_model)` and pass as `transport=`. The summarizer's `ainvoke` at ~`graph.py:1584` stays unchanged.
  - **Tier-3 compaction summarizer** in `executor/compaction/summarizer.py` around line 397 — currently calls `init_chat_model(model=..., timeout=120, max_retries=0)`, the exact pattern that triggers the silent-drop warning. Must be rewritten to go through `providers.create_llm` with a resolved `LLMTransportConfig`. This requires threading a `pool` and either `agent_config` or a pre-resolved transport through `summarize_slice` and its caller in `executor/compaction/pipeline.py`. If the caller doesn't have `agent_config` available, it's acceptable to pass `pool` only and let `summarize_slice` call `resolve_transport(None, provider=..., model=...)` for platform defaults — document this trade-off inline (and add a TODO to plumb agent_config when a future task requires per-agent summarizer tuning).
- Grep after the change: `grep -rn "init_chat_model" services/worker-service/` must be zero matches. `grep -n "providers.create_llm" services/worker-service/executor/graph.py` must show both graph.py calls using `transport=`; the compaction summarizer call in `summarizer.py` must also route through `providers.create_llm`.
- **Per-provider construction details** (native fields, no `timeout=`):
  - **Bedrock:** build `botocore.config.Config(connect_timeout=transport.connect_timeout_s, read_timeout=transport.read_timeout_s, retries={"max_attempts": 0})`; build `boto3.client("bedrock-runtime", region_name=region, config=config)`; construct `ChatBedrockConverse(model=..., temperature=..., region_name=region, client=client, max_tokens=transport.max_output_tokens)`. Preserve the existing `os.environ["AWS_BEARER_TOKEN_BEDROCK"] = api_key` pattern for DB-stored Bedrock API keys.
  - **OpenAI:** `ChatOpenAI(model=..., temperature=..., api_key=api_key, request_timeout=transport.read_timeout_s, max_retries=0, max_tokens=transport.max_output_tokens)`.
  - **Anthropic:** `ChatAnthropic(model=..., temperature=..., api_key=api_key, default_request_timeout=transport.read_timeout_s, max_retries=0, max_tokens=transport.max_output_tokens)`.
- **`connect_timeout_s` is Bedrock-only today.** OpenAI's and Anthropic's default httpx transports don't split connect from read. When OpenAI or Anthropic is constructed with a non-default `connect_timeout_s`, emit a one-shot `logger.info("transport.connect_timeout_unsupported", extra={"provider": ..., "connect_timeout_s": ...})` so operators see evidence their knob didn't apply. Document inline.
- **Prohibited:**
  - No `init_chat_model` import in `providers.py` after the rewrite. `grep -n "init_chat_model" services/worker-service/executor/providers.py` must be zero matches.
  - No hard-coded timeout or max_tokens numbers in `providers.py` — every value comes from `transport`.

## Max-tokens detection (provider-agnostic, added in graph.py)

After each `ainvoke` call at `graph.py:1176` (agent_node), inspect `response.response_metadata` and emit a structured WARN when the model stopped due to max_tokens. Three provider conventions must be recognised:

- Bedrock Converse: `stopReason == "max_tokens"`
- Anthropic: `stop_reason == "max_tokens"`
- OpenAI: `finish_reason == "length"`

Implementation: add a small helper `_is_max_tokens_stop(response_metadata: dict | None) -> bool` (parametrized-tested across the three conventions). When `True`, emit:

```python
logger.warning(
    "llm.max_tokens_reached",
    extra={
        "task_id": task_id,
        "provider": provider,
        "model": model_name,
        "max_tokens": transport.max_output_tokens,
    },
)
```

Do NOT raise. The partial AIMessage flows downstream exactly as today.

## Affected files

- `services/worker-service/executor/providers.py` (rewrite)
- `services/worker-service/executor/graph.py` (modify — both `create_llm` call sites + summarizer-factory signature + `_is_max_tokens_stop` helper + WARN after agent_node's `ainvoke`)
- `services/worker-service/executor/compaction/summarizer.py` (modify — replace the `init_chat_model` call near line 397 with a call through `providers.create_llm`)
- `services/worker-service/executor/compaction/pipeline.py` (modify if needed — thread `pool` / `agent_config` into the summarizer callable)
- `services/worker-service/tests/test_providers_transport.py` (new)
- `services/worker-service/tests/test_long_output_no_timeout.py` (new — regression guard across all THREE construction sites; see "Tests" below)
- `services/worker-service/tests/test_compaction_summarizer.py` / `test_compaction_cost_ledger.py` (modify — existing tests patch `executor.compaction.summarizer.init_chat_model`; migrate to patch `executor.compaction.summarizer.providers.create_llm` or add a `pool=None` fallback path that preserves the legacy patching for tests — whichever is smaller)

## Dependencies

- **Must complete first:** Task 1 (`LLMTransportConfig` resolver).
- **Provides output to:** —

## Tests

### `services/worker-service/tests/test_providers_transport.py`

Parallel structure across three providers:

- **Bedrock** (use real `boto3.Config`, no network):
  - `llm.client.meta.config.read_timeout == transport.read_timeout_s`.
  - `llm.client.meta.config.connect_timeout == transport.connect_timeout_s`.
  - `llm.client.meta.config.retries["total_max_attempts"] == 1` (botocore normalises `max_attempts=0` to `total_max_attempts=1`; stable across versions).
  - `llm.max_tokens == transport.max_output_tokens`.
  - `llm.model_kwargs` / `llm.additional_model_request_fields` contain none of `timeout`, `max_retries`, `max_tokens` (regression guard: the silent-drop bug is gone).
- **OpenAI:**
  - `llm.request_timeout == transport.read_timeout_s`.
  - `llm.max_retries == 0`.
  - `llm.max_tokens == transport.max_output_tokens`.
  - `llm.model_kwargs` contains none of `timeout`, `request_timeout`, `max_retries`, `max_tokens`.
  - Non-default `connect_timeout_s` emits exactly one `transport.connect_timeout_unsupported` INFO log (use `caplog`).
- **Anthropic:**
  - `llm.default_request_timeout == transport.read_timeout_s`.
  - `llm.max_retries == 0`.
  - `llm.max_tokens == transport.max_output_tokens`.
  - `llm.model_kwargs` contains none of `timeout`, `default_request_timeout`, `max_retries`, `max_tokens`.
  - Same one-shot INFO log assertion as OpenAI.
- **Regression guard (all three providers AND all three construction sites):** `warnings.catch_warnings(record=True)` around each `create_llm` call; assert no `UserWarning` matching `r".*was transferred to model_kwargs.*"`. Include the compaction-summarizer path explicitly so a future re-introduction of `init_chat_model` on that code path is caught.

Test the `_is_max_tokens_stop` helper in a separate small parametrized test — cover `{"stopReason": "max_tokens"}`, `{"stop_reason": "max_tokens"}`, `{"finish_reason": "length"}`, `{}`, `None`, and negative cases (`"end_turn"` / `"stop"`).

### `services/worker-service/tests/test_long_output_no_timeout.py` (regression guard, new)

Parametrized across Bedrock / OpenAI / Anthropic:

- `test_no_legacy_timeout_kwarg_warning_at_startup` — builds each provider's model via `providers.create_llm` with a mocked DB pool; asserts no UserWarning matching `r".*was transferred to model_kwargs.*"` is raised. This is the CI-provable regression guard for #85's specific class of bug.

Document the non-coverage explicitly in the module docstring: the real-botocore "whole call doesn't hit a hidden timeout at 300s" assertion belongs to #81's offline real-provider suite, not this file.

## Acceptance Criteria

- [ ] Worker startup emits no `"... was transferred to model_kwargs"` warning for any provider.
- [ ] `init_chat_model` import gone from `providers.py` (verified by grep).
- [ ] For each provider, the constructed model's native timeout field equals `transport.read_timeout_s`.
- [ ] For each provider, `max_tokens == transport.max_output_tokens` and `max_retries == 0`.
- [ ] `_is_max_tokens_stop` handles all three provider conventions; when true after agent_node's `ainvoke`, `llm.max_tokens_reached` WARN fires with `task_id`, `provider`, `model`, `max_tokens`.
- [ ] OpenAI / Anthropic with non-default `connect_timeout_s` emit exactly one `transport.connect_timeout_unsupported` INFO log.
- [ ] Both `create_llm` call sites in `graph.py` pass `transport=`; summarizer factory accepts `agent_config` and resolves its own transport.
- [ ] Both test files pass.
- [ ] No `astream`, no `_astream_or_cancel`, no `llm_stream_progress` / `llm_stream_complete`, no migration introduced.

## Out of Scope

- Streaming anything.
- Progress events, conversation-log changes, Console rendering.
- API-side validation of `agent_config.llm_transport` (Task 3).
- System-prompt chunking guidance (Task 4).

<!-- AGENT_TASK_END -->
