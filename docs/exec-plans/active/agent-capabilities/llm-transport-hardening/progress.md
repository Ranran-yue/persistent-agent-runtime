# Progress — LLM Transport Hardening (minimal)

Tracking [#85](https://github.com/shenjianan97/persistent-agent-runtime/issues/85). Four tasks, one row each. Update inline as work progresses.

| # | Task | Owner | Status | Notes |
|---|---|---|---|---|
| 1 | Transport defaults + resolver (`executor/transport.py`) |  | Not started | Platform defaults: `connect=10s`, `read=300s`, `max_tokens=16_384`. Bounds `[1,60] / [10,900] / [256,200_000]`. Exports constants consumed by Task 3's drift test. |
| 2 | Direct provider construction + `ainvoke` call-site wiring (`providers.py` + `graph.py`) |  | Not started | Rewrite `providers.py` to drop `init_chat_model`. Both `create_llm` sites in `graph.py` (agent_node + summarizer) pass `transport=`. Emit `llm.max_tokens_reached` WARN after `ainvoke` when `stop_reason=max_tokens` (provider-agnostic helper). Ships the regression test (`test_no_legacy_timeout_kwarg_warning_at_startup`) asserting no `"was transferred to model_kwargs"` UserWarning for any of the three providers. |
| 3 | API agent_config.llm_transport sub-object (Java) |  | Not started | Depends on Task 1 (drift test reads `transport.py`). `LlmTransportConfigRequest` record + validation + canonicalization + `LlmTransportBounds` constants + drift test. |
| 4 | System prompt: chunked-artifact guidance |  | Not started | Independent. Unit-test acceptance only. Gated on `create_text_artifact` in `allowed_tools`. |
