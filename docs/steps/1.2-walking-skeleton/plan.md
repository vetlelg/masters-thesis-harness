# 1.2 Walking skeleton, first agent run: M1 (~3 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


**Goal:** the smallest end-to-end agent: config → model + MCP tools → `create_agent` → markdown report on stdout.

**Files:**

- `config.py`: Pydantic models, `load_config()`, and the guard that `environment: real` requires a local model.
- `agent.py`: `run_check`, `RunResult`.
- `cli.py`: `check` and `tools`.
- `configs/dev-claude.yaml`, `configs/dev-ollama.yaml`.
- `prompts/system.md` and `prompts/daily_check.md` (v0, a few lines each).
- `.env`: add `ANTHROPIC_API_KEY`, created in the Anthropic Console. Until you have a key, start with `dev-ollama.yaml`.

**Commands:**

```powershell
docker pull grafana/mcp-grafana:2.2.0
uv run monitor tools --config configs/dev-claude.yaml   # MCP only, no LLM: the tools the model would get
uv run monitor check --config configs/dev-claude.yaml
uv run monitor check --config configs/dev-ollama.yaml   # llama3.1:8b until step 1.6 picks a model
ollama ps                                                # during or right after the local run
```

**Verify:**

- `monitor tools` lists only Prometheus, Loki and datasource tools: no `update_*`, `create_*` or `grafana_api_request`. It shows an approximate token cost per tool and in total; that cost comes out of `num_ctx` on every model call. Afterwards, `docker ps` shows no leftover `mcp-grafana` container.
- **Claude run:** the report names all four services and mentions errors in `dice-java`. The console shows each tool call as it happens. A run typically takes under 2 minutes.
- **Ollama run:** it finishes. The quality may be poor and the run may take many minutes on this GPU; that's fine for now. `ollama ps` shows CONTEXT 16384, not 4096, and the GPU/CPU split.
- `git tag v0.1.0`

**Notes:**

- Keep the MCP adapter open for the whole run.
- The safety flags are added in code.
- Print every tool call live, so a slow local run isn't a black box.
