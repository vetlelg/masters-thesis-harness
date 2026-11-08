# 2.1 Test tiers and integration checks (~2 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** fast tests run by default; slower tiers are opt-in.

**Files:** pytest markers in `pyproject.toml` (default `-m "not integration and not agent"`), `tests/integration/test_mcp_tools.py`, `tests/integration/truth.py`, `tests/snapshots/tools-dev.json`.

- **`integration` tier** (dev stack up, no LLM): the MCP server starts, and the tool names and schema hash match the snapshot. Tool descriptions are part of the prompt, so an MCP upgrade that changes them should fail a test and be accepted deliberately.
- **`truth.py`:** a tiny client that queries Prometheus (`:9090/api/v1/query`) and Loki (`:3100/loki/api/v1/query`) directly. This gives ground truth by a path independent of Grafana and MCP, so it doesn't share the agent's failure modes.

**Verify:** `uv run pytest` (fast) and `uv run pytest -m integration` (stack up) both pass.
