# 1.4 Safety guards and first unit tests (~2 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** read-only access and "telemetry stays local" are enforced by code and covered by tests.

**Files:** `config.py`, `agent.py`, `tests/test_config.py`, `tests/test_safety.py`.

- **Startup check on the loaded tools:** abort if a tool name looks like a write (`create_`, `update_`, `delete_`, `patch_`, `install_`, `grafana_api_request`), or if the server's MCP annotations mark a tool as not read-only. `monitor tools` prints the annotations, so you can see whether the server provides them.
- **`mcp.tool_allowlist`:** exact tool names. An unknown name fails at startup, which catches typos and tools renamed by an MCP upgrade.
- **Config guard:** `environment: real` requires `provider: ollama` and a model name that doesn't contain `cloud`.
- **Tests:**
  - the guard rejects real + anthropic and rejects `*-cloud` models;
  - the built MCP command always contains `-t stdio`, `--disable-write` and `--usage-stats=disabled`;
  - allowlist filtering works.

**Verify:**

```powershell
uv run pytest            # under 10 s, no Docker or LLM needed
uv run monitor tools     # with an allowlist set, only those tools appear
```

Also: put a write tool name in the allowlist; startup must refuse to run.
