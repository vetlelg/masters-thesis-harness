# 1.6 Daily-check prompt v1 and local-model comparison: MVP (~½–1 day): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** a daily report an operator would recognise, and a default local model chosen from measurements.

**Files:** `prompts/system.md`, `prompts/daily_check.md` (v1), `agent.knowledge` in both dev configs, `docs/notes/bakeoff-2026-11.md`.

**Prompt v1:**

- **`system.md`:**
  - role: a read-only monitoring assistant;
  - rules: every number must come from a tool result, and the report names the query that produced it; say "unknown" instead of guessing; prefer instant queries with aggregation over long range queries;
  - time context: `$now` in UTC and Europe/Oslo, and `$window`;
  - `$knowledge`: datasource UIDs, expected services, service label, metric names and thresholds. These come from config, which keeps the prompt itself independent of the environment.
- **`daily_check.md`:** a procedure and a fixed markdown template.
  - Procedure:
    1. list the services and compare them with the expected list;
    2. per service, get request rate, 5xx ratio and p95 latency;
    3. per service, count error logs and find the most frequent message;
    4. classify each result with the thresholds;
    5. write the report.
  - Template: an overall status line, a per-service table, findings with their queries, and what was not checked.

**Comparison runs:**

- **Candidates:**
  - Claude Haiku 4.5 and Sonnet 5, as the reference;
  - 2–3 local models that list `tools` in `ollama show` and are about 4 GB or smaller, e.g. the 2B/4B variants of a current family such as qwen3.5. Pick them from https://ollama.com/search?c=tools when you get there;
  - llama3.1:8b, already installed, as the "too big for this GPU" reference.
- Do 3 runs per candidate, using `--set model.name=…`.
- **Score each report on:**
  1. all 4 services covered;
  2. dice-java errors flagged;
  3. dice-go latency flagged;
  4. dice-python not flagged;
  5. dice-node's 400s not called a server fault.
- **Also record:** whether the run completed, tool errors, wall time, tokens, and the PROCESSOR and CONTEXT columns of `ollama ps`.
- For the best local model, also compare all tools in the categories (about 15) against a 5–7-tool allowlist, and `num_ctx` 8k against 16k.

**Verify:**

- Claude passes the five checks in at least 2 of 3 runs.
- The table in `docs/notes/` decides the default in `dev-ollama.yaml`. If no local model passes, that is a result too: write it down and keep going, since the bigger machine comes later.
- `git tag v0.3.0`: **MVP complete.**
