# 2.4 Local tracing (optional, ~2 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** browse the LLM calls in a UI while debugging prompts: the full prompt including tool schemas, latency and tokens.

**Files:** a `phoenix` service under the compose profile `tracing`, `tracing.py`, and `tracing.enabled` in config.

- Run `arizephoenix/phoenix:<pinned version>` with `PHOENIX_TELEMETRY_ENABLED=false` and `PHOENIX_ALLOW_EXTERNAL_RESOURCES=false` on `127.0.0.1:6006`.
- Run `uv add arize-phoenix-otel openinference-instrumentation-langchain`. Register the LangChain instrumentor only when tracing is enabled, and tag the spans with the run id.
- Don't export these traces to the monitored LGTM stack. The agent would then see its own traces in the system it checks.
- The JSON transcript stays the source of truth, and tracing must never make a run fail.

**Verify:**

- With tracing on, a run appears at http://localhost:6006 with one span per LLM call and per tool call.
- With Phoenix stopped, `monitor check` still works.

Tags: `v0.4.0` after 2.2, `v0.5.0` after 2.3, `v0.6.0` after 2.4.
