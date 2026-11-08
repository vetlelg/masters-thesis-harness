# 1.5 Context and loop control for small models (~3 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** a small model with a 16k context can't overflow silently or loop forever, and a run stopped by its budget still produces a report.

**Files:** `middleware.py`, `agent.py`, and `config.py`, which gains `--set key=value` overrides (values parsed as YAML) for experiments.

- **Tool-output cap:** `@wrap_tool_call` middleware cuts each tool result to `agent.max_tool_result_chars` and appends `[truncated: showing N of M chars. Narrow the query: shorter range, more label filters, lower limit]`. The transcript keeps the original size.
- **Loki line limit:** pass `--max-loki-log-limit` from `mcp.max_loki_log_limit`.
- **Call budget:** `ModelCallLimitMiddleware(run_limit=agent.max_model_calls, exit_behavior="end")`. If the run ends without a final answer, make one call without tools ("Budget used up. Write the report from what you have.") and set `status: budget_exhausted`.
- **Tool errors:** `ToolErrorMiddleware` turns tool exceptions into messages the model can react to.
- **Context warning:** after each model call, warn if `input_tokens > 0.85 × num_ctx` or `done_reason == "length"`. Print it to the console and store it in `meta.context_warnings`.

**Verify:**

```powershell
uv run monitor check -c configs/dev-ollama.yaml --set model.params.num_ctx=4096        # → context warnings
uv run monitor check -c configs/dev-claude.yaml --set "agent.task=Show the last 500 log lines of dice-java"   # → truncation marker in the transcript
uv run monitor check -c configs/dev-claude.yaml --set agent.max_model_calls=2          # → budget_exhausted, report still written
uv run pytest
```
