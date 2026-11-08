# Implementation plan: monitoring and troubleshooting agent harness

> **Replaced (2026-11-08)** by [architecture.md](architecture.md), `CLAUDE.md` and the step docs in [steps/](steps/). Nothing has been lost: the cross-cutting content is in architecture.md, and each step's section is in `steps/<n>-<slug>/plan.md`. Kept only until the first commit; step 1.0.13 removes it.

*Draft v1, 2026-11-06. Input: [brief.md](brief.md). Library docs and versions checked on 2026-11-06 ([Appendix A](#appendix-a--versions-and-sources-checked-2026-11-06)).*
*Written from docs only. Nothing has been run on this machine yet. Every assumption that still needs checking is listed in [Risks](#11-risks) with the step that checks it.*

## 0. Summary

- **Running agent at step 1.2 (about day 1):** `uv run monitor check` → LangChain `create_agent` (a LangGraph graph) → Grafana MCP server (Docker, stdio, read-only) → local LGTM stack → markdown report. It works with Claude or Ollama from the first run; the provider is one config value.
- **MVP done at step 1.6 (tag `v0.3.0`, about 3 days of work):** all four hooks from the brief, guard rails for small models, prompt v1, and a local model chosen from measurements.
- **Phase 2 (about 4–5 days):** structured report, a golden set (YAML data + pytest) with ground truth computed live, and optional local tracing.
- **Phases 3–4:** fault-injection lab with Inspect AI, then system knowledge, an explicit LangGraph workflow and model comparisons on the bigger machine.
- Effort figures are rough, for one person.

## 1. Decisions and changes to the brief

Decisions from our Q&A: Claude is used for development on synthetic data only, and a local Ollama model is the one evaluated in the thesis. Work starts on the laptop and moves to a bigger machine later, so the Ollama `base_url` lives in config.

| # | Brief | Plan | Why |
|---|---|---|---|
| 1 | `langchain-mcp-adapters` (`MultiServerMCPClient`) | `langchain.mcp.MCPAdapter` from `langchain[mcp]` ≥ 1.4 (built on FastMCP 4) | LangChain 1.4 moved MCP into core and its docs say this replaces the standalone package. `langchain-mcp-adapters` 0.3.2 pins `mcp<2`, so the two can't share an environment. The new API is beta, so it is pinned in `uv.lock`, with a fallback (R3). |
| 2 | Ollama model, e.g. qwen3 or gpt-oss | Two providers from day one, switched in config: Claude for dev (synthetic data only) and local Ollama (the thesis target). The local model is picked by a measured comparison in step 1.6. | The GPU has 4 GB of VRAM. gpt-oss:20b doesn't fit, and 8B models (including the installed llama3.1:8b) spill onto the CPU. The small-model options have also changed since the brief (e.g. the qwen3.5 family), so measure rather than guess. Claude separates harness bugs from small-model weakness. |
| 3 | `num_ctx` 16k–32k | Start at 16k on the laptop and choose by measurement. 32k+ on the bigger machine. Hard caps on tool output. | A 32k KV cache won't fit in 4 GB next to a 4B model. With tool output capped, 16k is workable. |
| 4 | One example app + traffic generator | All four otel-lgtm example apps as separate services, plus a traffic container | The apps behave differently: Java fails ~30 % of requests with HTTP 500, Go takes ~1 s per request, Node returns 400 when `?rolls` is missing, Python is healthy. That gives the MVP known answers to find. Containers also avoid Windows script problems. |
| 5 | Viewer service account (assumed to exist) | A bootstrap script creates it | otel-lgtm stopped creating service accounts in v0.36. |
| 6 | Read-only via `--disable-write` and a Viewer token | Also: `--usage-stats=disabled` (on by default in mcp-grafana), a client-side tool allowlist, a startup check that no write tool is loaded, Ollama cloud models disabled plus a config guard, Grafana and Phoenix analytics off, LangSmith off | "Telemetry must not leave my machine", plus defence in depth. |
| 7 | `run_check(config) -> report` | `async def run_check(config) -> RunResult` (report, run folder, status). The task or question is part of the config. | MCP tools and Inspect are async. A config snapshot then describes a run completely, and golden/eval items are just config overrides. |
| 8 | Config: URL, model, `num_ctx`, tools, prompt | Also environment knowledge (datasource UIDs, service label, metric names, expected services), check window, limits, and the MCP launcher. Read-only flags are added by code, not config. | Small models need the facts written down. Switching environments stays a config change, and a config typo can't turn off read-only mode. |
| 9 | Golden set run with pytest | Golden set as YAML data with a pytest runner. Ground truth comes from Prometheus/Loki at test time. Fast unit tests run separately. | Live data drifts. The same data file is reused by Inspect in Phase 3. |
| 10 | Tracing (tool not specified) | Local Phoenix in Docker, opt-in. Not exported into the monitored LGTM stack, and never LangSmith cloud. | Data must stay local. Agent traces inside the monitored stack would show up in the agent's own list of services. |
| 11 | Grafana link in the report | Built by code from the evidence query | LLMs garble URLs; links built in code are testable. |
| 12 | MCP image `grafana/mcp-grafana` | Pinned to `:2.2.0`. The native PyPI binary is an alternative launcher. | Tool descriptions are part of the prompt and must not change silently. The native launcher avoids Docker networking problems (e.g. on VPN). |
| 13 | Phase 3: live fault injection | Adds "frozen episodes": snapshot the LGTM data for each fault episode and replay it with a fixed "now" | Every agent version and model then sees identical telemetry, so comparisons are fair and cheap. |

Note: otel-lgtm uses **Prometheus**, not Mimir. Both use PromQL, so this doesn't matter for the agent.

## 2. Working rules (keep the agent running)

- **Every step is done when:**
  1. `uv run monitor check --config configs/dev-claude.yaml` produces a report against the dev stack.
  2. From step 1.4 on, `uv run pytest` passes (fast tests, no Docker or LLM needed).
  3. The work is committed. Milestones get a git tag (`v0.1.0`, …), because Phase 3 re-runs these tags.
- Run the Ollama config at least once in every step that touches the agent loop. It is slow on the laptop, but it must keep working.
- Put risky changes behind a config switch (e.g. `report.mode`, `tracing.enabled`), so falling back is a config change rather than a revert.
- All commands are PowerShell 7, run from the repo root.

## 3. MVP architecture

```text
uv run monitor check --config configs/dev-claude.yaml
        │
        ▼
run_check(config) ───────────────► runs/<run_id>/  config · meta · tools · transcript · report
        │
        ├─ model: init_chat_model(provider, name, params) ──► Claude API (dev only)  |  Ollama :11434
        │
        └─ tools: MCPAdapter(StdioTransport("docker run -i grafana/mcp-grafana:2.2.0 -t stdio --disable-write …"))
                        │  stdio, one container per run
                        ▼
                Grafana :3000 (Viewer token) ──► Prometheus · Loki · Tempo        (grafana/otel-lgtm)
                                                    ▲ OTLP
                        dice-java · dice-go · dice-python · dice-node ◄── traffic
```

API sketch, checked against the current docs. This is not the final code:

```python
# src/monitor_agent/agent.py (step 1.2)
from fastmcp.client import Client
from fastmcp.client.transports import StdioTransport
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.mcp import MCPAdapter  # beta, langchain>=1.4

SAFETY_FLAGS = ["-t", "stdio", "--disable-write", "--usage-stats=disabled"]  # added by code, never read from config

async def run_check(cfg: Config) -> RunResult:
    model = init_chat_model(cfg.model.name, model_provider=cfg.model.provider, **cfg.model.params)
    transport = StdioTransport(
        command=cfg.mcp.command,
        args=[*cfg.mcp.args, *SAFETY_FLAGS, "--enabled-tools", ",".join(cfg.mcp.enabled_categories)],
        env={"GRAFANA_URL": cfg.grafana.url,
             "GRAFANA_SERVICE_ACCOUNT_TOKEN": os.environ["GRAFANA_SERVICE_ACCOUNT_TOKEN"]},
    )
    async with MCPAdapter(Client(transport)) as adapter:  # kept open for the whole run
        tools = await adapter.list_tools()
        agent = create_agent(model, tools, system_prompt=render_system_prompt(cfg))
        state = await agent.ainvoke({"messages": [{"role": "user", "content": render_task(cfg)}]})
    ...
```

Facts from the docs that shaped this design:

- `MCPAdapter` tools open an MCP session on each call unless the adapter context stays open. FastMCP's `StdioTransport` reuses its subprocess (`keep_alive=True`), so keeping the context open means one `docker run` per run instead of one per tool call.
- FastMCP gives stdio servers only a small allowlist of environment variables (on Windows: `PATH`, `USERPROFILE`, `APPDATA`, …). Pass the Grafana URL and token explicitly. The docker args forward them into the container with `-e GRAFANA_URL -e GRAFANA_SERVICE_ACCOUNT_TOKEN`.
- The `grafana/mcp-grafana` image defaults to SSE, so `-t stdio` and `docker run -i` are required.
- Inside a container, `localhost` means the container itself. On Docker Desktop the MCP server reaches the dev Grafana at `http://host.docker.internal:3000`. Links for people use `http://localhost:3000` (`grafana.public_url`).
- An MCP server error reaches the model as `ToolMessage(status="error")`, so the model can react to it. Transport failures raise exceptions.
- `ChatOllama` accepts `num_ctx`, `temperature`, `seed`, `reasoning` (True, False, or "low"/"medium"/"high"), `keep_alive` and `base_url`. Token usage comes from Ollama's `prompt_eval_count`/`eval_count`. `done_reason: "length"` means the output was cut off.
- Ollama 0.40 defaults to a 4k context on GPUs under 24 GiB, and it now offers cloud models. `OLLAMA_NO_CLOUD=1` makes it local-only.
- Use `string.Template` (`$now`) for prompt templates, because the braces in PromQL/LogQL break `str.format`.

## 4. Repo layout

This is the layout at the end of Phase 2. The step that adds each item is in brackets. Start with three modules (`cli`, `config`, `agent`) and split only when a step needs it.

```text
masters-thesis-harness/
├── pyproject.toml                 [1.0] uv project; script: monitor = "monitor_agent.cli:main"
├── uv.lock, .python-version       [1.0]
├── .env.example, .gitignore       [1.0] .env and runs/ are git-ignored
├── README.md                      [1.0] quickstart, kept current
├── configs/
│   ├── dev-claude.yaml            [1.2] dev stack + Claude (synthetic data only)
│   └── dev-ollama.yaml            [1.2] dev stack + local model
├── prompts/
│   ├── system.md                  [1.2] role, rules, $now, $window, $knowledge
│   └── daily_check.md             [1.2] procedure + output template (v1 in 1.6)
├── deploy/dev/compose.yaml        [1.1] otel-lgtm + 4 example apps + traffic (+ phoenix profile in 2.4)
├── scripts/grafana_viewer_token.py [1.1] creates the Viewer service account + token, checks permissions
├── src/monitor_agent/
│   ├── cli.py                     [1.0] monitor check | monitor tools
│   ├── config.py                  [1.2] Pydantic config, YAML loading, guards, --set overrides (1.5)
│   ├── agent.py                   [1.2] run_check(), RunResult
│   ├── runlog.py                  [1.3] run folder, meta, transcript
│   ├── middleware.py              [1.5] tool-output cap, context warnings
│   ├── report.py                  [2.2] HealthReport, links, grounding, markdown rendering
│   └── tracing.py                 [2.4] opt-in Phoenix/OpenInference
├── tests/
│   ├── test_config.py, test_safety.py, test_middleware.py   [1.4–1.5]
│   ├── test_report.py             [2.2]
│   ├── integration/               [2.1] MCP tool snapshot, ground-truth helpers
│   └── golden/                    [2.3] golden runner (pytest)
├── evals/golden/v1.yaml           [2.3] golden items (data; reused by Inspect in Phase 3)
├── docs/                          brief, this plan, notes/ (bake-off results, baselines)
└── runs/                          git-ignored, one folder per run
```

## 5. The config file (hook 1)

```yaml
# configs/dev-claude.yaml: local dev stack + Claude. Synthetic data only.
environment: dev            # "real" ⇒ provider must be ollama, no *cloud* models (enforced in config.py)

grafana:
  url: http://host.docker.internal:3000   # as seen by the MCP server container
  public_url: http://localhost:3000       # used for links in reports

mcp:
  command: docker
  args: [run, --rm, -i, -e, GRAFANA_URL, -e, GRAFANA_SERVICE_ACCOUNT_TOKEN, grafana/mcp-grafana:2.2.0]
  # without Docker: command: uv, args: [run, mcp-grafana]   (after `uv add mcp-grafana==2.2.0`; then url: http://localhost:3000)
  enabled_categories: [prometheus, loki, datasource]   # → --enabled-tools (category names are singular)
  tool_allowlist: []          # step 1.4; empty = every tool in the categories
  max_loki_log_limit: 20      # step 1.5; the server default of 100 lines is too much for 16k

model:
  provider: anthropic
  name: claude-haiku-4-5-20251001     # everyday dev model; claude-sonnet-5 as reference
  params: {temperature: 0, max_tokens: 4096}   # set max_tokens explicitly, reports are long

agent:
  system_prompt_file: prompts/system.md
  task_file: prompts/daily_check.md
  task: null                  # a question here overrides task_file (golden Q&A items, experiments)
  window: 30m                 # the dev stack is young; real system: 24h
  max_model_calls: 15         # step 1.5
  max_tool_result_chars: 6000 # step 1.5
  knowledge:                  # environment facts rendered into the system prompt ($knowledge)
    datasources: {prometheus: prometheus, loki: loki, tempo: tempo}
    services: [dice-java, dice-go, dice-python, dice-node]   # a service that sends nothing is a finding
    service_label: job        # confirm in step 1.1
    metrics: {}               # HTTP metric names per service, written down in step 1.1
    thresholds: {error_ratio_warning: 0.01, error_ratio_critical: 0.05, p95_warning_s: 0.5, p95_critical_s: 2.0}

run:
  output_dir: runs
```

`configs/dev-ollama.yaml` differs only in `model`:

```yaml
model:
  provider: ollama
  name: llama3.1:8b           # replaced by the winner of the step 1.6 comparison
  params: {num_ctx: 16384, temperature: 0, seed: 42, keep_alive: 30m, base_url: http://localhost:11434}
  # add reasoning: false for thinking-capable models (thinking tokens are slow on this GPU)
```

Secrets live only in `.env` (git-ignored). The config names environment variables, never their values. `--config` falls back to `MONITOR_CONFIG` from `.env`, so a plain `uv run monitor check` works on every machine. A later `configs/real-readonly.yaml` changes `environment`, `grafana`, `knowledge`, `window` and the model, and turns on mcp-grafana's Loki and Prometheus query guardrails (`--loki-guardrail-mode enforce`, `--prometheus-guardrail-mode enforce`) to protect the production backends.

## 6. Phase 1: MVP

### 1.0 Prerequisites and project skeleton (~1 h)

**Goal:** tools are installed, an empty package runs, and git history starts now (hook 3 needs a commit to record).

**Files:** `pyproject.toml`, `src/monitor_agent/__init__.py`, `src/monitor_agent/cli.py` (argparse stub with `check`), `.gitignore`, `.env.example`, `README.md`.

**Commands:**

```powershell
winget install --id astral-sh.uv -e           # then open a new terminal
# Ollama: local models only, and an 8-bit KV cache (about half the context memory).
# Quit Ollama from the tray and start it again afterwards.
[Environment]::SetEnvironmentVariable("OLLAMA_NO_CLOUD", "1", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_KV_CACHE_TYPE", "q8_0", "User")

uv init --package --name monitor-agent --python 3.13      # also runs git init
uv add "langchain[mcp,ollama,anthropic]" pydantic pyyaml python-dotenv
uv add --dev pytest
```

Then:

- Set `monitor = "monitor_agent.cli:main"` under `[project.scripts]`.
- Add `.env` and `runs/` to `.gitignore`.
- Write `.env.example`:

```dotenv
# Viewer token, written by scripts/grafana_viewer_token.py (1.1)
GRAFANA_SERVICE_ACCOUNT_TOKEN=
# Only for provider: anthropic, synthetic data only
ANTHROPIC_API_KEY=
# LangSmith is a cloud service: keep it off
LANGSMITH_TRACING=false
# Default for --config
MONITOR_CONFIG=configs/dev-claude.yaml
```

Comments go on their own lines: a comment after a value can end up as part of the value.

Python 3.13 is managed by uv. The system Python 3.14 would probably work too, but 3.13 is the safer choice for Windows wheels.

**Verify:**

```powershell
uv run monitor --help
uv run python -c "import langchain.mcp, langchain_ollama, langchain_anthropic; print('ok')"
Select-String "$env:LOCALAPPDATA\Ollama\server.log" -Pattern "cloud disabled"   # after restarting Ollama
git add -A; git commit -m "Bootstrap project"
```

### 1.1 Dev telemetry stack (~1 h, mostly image builds)

**Goal:** synthetic telemetry with known behaviour, reachable with a Viewer token.

**Files:** `deploy/dev/compose.yaml`, `scripts/grafana_viewer_token.py`.

Compose sketch. Copy the per-app OTLP settings from each example's `docker-compose.oats.yml`:

```yaml
name: monitor-dev
services:
  lgtm:
    image: grafana/otel-lgtm:0.40.0                 # pin it, and record the digest
    ports: ["127.0.0.1:3000:3000", "127.0.0.1:9090:9090", "127.0.0.1:3100:3100"]  # Grafana; Prometheus + Loki for ground truth (2.1)
    environment:
      GF_ANALYTICS_REPORTING_ENABLED: "false"
      GF_ANALYTICS_CHECK_FOR_UPDATES: "false"
      GF_ANALYTICS_CHECK_FOR_PLUGIN_UPDATES: "false"
  dice-java:     # ~30 % HTTP 500 ("simulating an error"); latency set by GRAFANA_ROLL_WAIT (default 200 ms)
    build: https://github.com/grafana/docker-otel-lgtm.git#v0.40.0:examples/java
    environment:
      OTEL_SERVICE_NAME: dice-java
      OTEL_EXPORTER_OTLP_ENDPOINT: http://lgtm:4318
  dice-go:       # ~1 s per request (busy wait)
    build: https://github.com/grafana/docker-otel-lgtm.git#v0.40.0:examples/go
    environment: {OTEL_SERVICE_NAME: dice-go, OTEL_EXPORTER_OTLP_ENDPOINT: "http://lgtm:4318", OTEL_METRIC_EXPORT_INTERVAL: "5000"}
  dice-python:   # healthy baseline
    build: https://github.com/grafana/docker-otel-lgtm.git#v0.40.0:examples/python
    environment: {OTEL_SERVICE_NAME: dice-python, OTEL_EXPORTER_OTLP_ENDPOINT: "http://lgtm:4317", OTEL_METRIC_EXPORT_INTERVAL: "5000"}
  dice-node:     # HTTP 400 when ?rolls is missing: client errors, not a server fault
    build: https://github.com/grafana/docker-otel-lgtm.git#v0.40.0:examples/nodejs
    environment: {OTEL_SERVICE_NAME: dice-node, OTEL_EXPORTER_OTLP_ENDPOINT: "http://lgtm:4318", OTEL_METRIC_EXPORT_INTERVAL: "5000"}
  traffic:
    image: curlimages/curl:<pinned>
    entrypoint: ["/bin/sh", "-c"]
    command: ["while true; do curl -s dice-java:8080/rolldice; curl -s dice-go:8081/rolldice; curl -s dice-python:8082/rolldice; curl -s 'dice-node:8084/rolldice?rolls=2'; curl -s dice-node:8084/rolldice; sleep 2; done"]
```

`scripts/grafana_viewer_token.py` uses only the standard library. It:

1. waits for `/api/health`;
2. finds or creates the service account `monitor-agent` with role Viewer, using admin/admin basic auth;
3. creates a token and writes it to `.env`;
4. tests the new token with `GET /api/datasources` (list), `GET /api/datasources/uid/prometheus`, and a PromQL query through `POST /api/ds/query`, and prints OK or the HTTP status for each.

**Commands:**

```powershell
docker compose -f deploy/dev/compose.yaml up -d --build      # first build takes several minutes
docker compose -f deploy/dev/compose.yaml ps
uv run scripts/grafana_viewer_token.py
```

**Verify:**

- All six services are running. After about 2 minutes, open Grafana at http://localhost:3000 (admin/admin) → Explore:
  - **Prometheus:** find each service's HTTP server metrics. Names differ by language and semantic-conventions version. Write down the metric names, and whether the service name is in the `job` or `service_name` label, into `agent.knowledge`.
  - **Loki:** `{service_name="dice-java"} |= "simulating an error"` returns lines.
  - **Tempo:** there are `dice-java` traces with error status.
- The script's permission table shows whether a Viewer token can list datasources and query. Note the result (risk R1).
- If the build from the Git URL fails, run `git clone --depth 1 --branch v0.40.0 https://github.com/grafana/docker-otel-lgtm deploy/dev/vendor/otel-lgtm` (git-ignored) and point `build:` at the local folders (R10).

### 1.2 Walking skeleton, first agent run: M1 (~3 h)

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

### 1.3 Run records: reproducibility and transcripts, hooks 3 and 4 (~2 h)

**Goal:** every run leaves a folder that fully describes it.

**Files:** `runlog.py`. `agent.py` streams the run instead of waiting for it to finish, and `cli.py` prints the run folder.

```text
runs/20261106T101500Z-3f9a1c/
├── config.json       resolved config (environment variable names only, never secret values)
├── meta.json         see below
├── tools.json        name, description and input schema of every tool given to the model
├── transcript.json   every message with timestamps: system, user, AI (tool calls, reasoning,
│                     usage, response_metadata), tool results (+ MCP artifacts)
├── git.diff          only if the working tree was dirty
└── report.md
```

`meta.json` contains:

- run id, start/end time, duration, and status (`ok`, `error` or `budget_exhausted`);
- git commit, branch and dirty flag;
- model provider and requested name. For Ollama, also the server version, model **digest**, quantization, parameter size and capabilities. For Anthropic, the model id the API returned;
- the MCP image reference and digest (`docker image inspect`) and the sha256 of `tools.json`;
- versions of the key packages, the sha256 of `uv.lock`, and the sha256 of the rendered prompts;
- counts of model calls, tool calls and tool errors, and token totals.

**Implementation notes:**

- Stream the graph and append messages as they arrive. Write the files in `finally`, so a failed run still leaves a transcript.
- Get the Ollama facts from the `ollama` client, which `langchain-ollama` already depends on (`/api/version`, `/api/tags`, `/api/show`).
- Use UTF-8 everywhere (`ensure_ascii=False`), because logs will contain Norwegian text.

**Verify:**

- Two runs give two folders. `meta.json` has the commit hash and `dirty: false`. Edit a file and run again: now `dirty: true` and `git.diff` exists.
- After an Ollama run, the digest in `meta.json` starts with the ID shown by `ollama list`.
- Stop Grafana during a run (`docker compose -f deploy/dev/compose.yaml stop lgtm`). The run should end with `status: error` and a transcript up to the failure. Start Grafana again.
- `uv run python -m json.tool runs\<id>\transcript.json` parses.
- `git tag v0.2.0`

### 1.4 Safety guards and first unit tests (~2 h)

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

### 1.5 Context and loop control for small models (~3 h)

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

### 1.6 Daily-check prompt v1 and local-model comparison: MVP (~½–1 day)

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

## 7. Phase 2: harness basics

### 2.1 Test tiers and integration checks (~2 h)

**Goal:** fast tests run by default; slower tiers are opt-in.

**Files:** pytest markers in `pyproject.toml` (default `-m "not integration and not agent"`), `tests/integration/test_mcp_tools.py`, `tests/integration/truth.py`, `tests/snapshots/tools-dev.json`.

- **`integration` tier** (dev stack up, no LLM): the MCP server starts, and the tool names and schema hash match the snapshot. Tool descriptions are part of the prompt, so an MCP upgrade that changes them should fail a test and be accepted deliberately.
- **`truth.py`:** a tiny client that queries Prometheus (`:9090/api/v1/query`) and Loki (`:3100/loki/api/v1/query`) directly. This gives ground truth by a path independent of Grafana and MCP, so it doesn't share the agent's failure modes.

**Verify:** `uv run pytest` (fast) and `uv run pytest -m integration` (stack up) both pass.

### 2.2 Structured report (~1 day)

**Goal:** the report becomes data: findings with severity, evidence, link and confidence.

**Files:** `report.py`, `agent.py`, `cli.py`, `tests/test_report.py`.

```python
class Evidence(BaseModel):
    datasource: Literal["prometheus", "loki", "tempo"]
    query: str               # the exact PromQL/LogQL/TraceQL that was run
    time_range: str
    observed: str            # what the tool result showed
    # added by the harness: grafana_link, grounded

class Finding(BaseModel):
    title: str
    service: str | None
    severity: Literal["ok", "info", "warning", "critical"]
    description: str
    evidence: list[Evidence]
    confidence: float = Field(ge=0, le=1)

class HealthReport(BaseModel):
    overall_status: Literal["healthy", "degraded", "unhealthy", "unknown"]
    summary: str
    services_checked: list[str]
    findings: list[Finding]
    limitations: list[str]
```

- Use `create_agent(..., response_format=ToolStrategy(HealthReport))`, and use ToolStrategy for every provider, so model comparisons aren't skewed by provider-specific structured output.
- **Added by the harness, not the LLM:**
  - `grafana_link` for each evidence item: an Explore URL built from `grafana.public_url`, the datasource UID, the query and the time range;
  - `grounded` for each evidence item: the query was actually run in this run's transcript;
  - `grounded_ratio` in `meta.json`.
- `report.md` is rendered from `report.json`, so the CLI output keeps its shape.
- **Fallback, `report.mode: extract`:** run the free-text agent as in Phase 1, then make one extraction call with `with_structured_output(HealthReport, method="json_schema")`, which uses Ollama's constrained decoding. Use this if the local model can't produce the ToolStrategy output reliably.

**Verify:**

- `report.json` validates against the schema.
- A link opens Grafana Explore with the query filled in.
- Claude produces valid reports in at least 9 of 10 runs; record the local model's success rate.
- Unit tests cover link building and grounding.

### 2.3 Golden set v1 (~1–1.5 days)

**Goal:** a repeatable score that answers "did this change make the agent better or worse?".

**Files:** `evals/golden/v1.yaml`, `tests/golden/conftest.py` (options `--golden-config`, `--repeats`), `tests/golden/test_golden.py`.

Each item is data with these fields:

- `id`;
- `task`: `daily_check` or a question, which becomes `agent.task`;
- `checks` on `report.json`;
- optional `truth`: PromQL or LogQL evaluated at test time with `truth.py`, plus a tolerance;
- optional `fault`: a fixture that changes the dev stack and restores it afterwards.

| id | Task | Passes when |
|---|---|---|
| services-listed | daily_check | `services_checked` = the 4 services |
| java-5xx-flagged | daily_check | dice-java finding ≥ warning; reported 5xx ratio within ±0.10 of truth |
| go-latency-flagged | daily_check | dice-go latency finding ≥ warning; p95 within ±30 % of truth |
| python-no-false-alarm | daily_check | no dice-python finding ≥ warning |
| node-4xx-not-critical | daily_check | dice-node not `critical` (400s are client errors) |
| evidence-grounded | daily_check | `grounded_ratio` = 1.0 |
| qa-python-rate | "Request rate of dice-python over the last 15 minutes?" | number within ±20 % of truth |
| qa-java-error-log | "Most recent error log line from dice-java?" | answer contains "simulating an error" |
| fault-python-down | `docker compose stop dice-python`, wait 3 min | dice-python flagged (no traffic) ≥ warning |
| fault-java-slow | recreate dice-java with `GRAFANA_ROLL_WAIT=2000` | dice-java latency finding ≥ warning |

The runner marks items with `@pytest.mark.agent`, and the fault items also with `fault`, since they take minutes. It writes `runs/golden/<timestamp>/summary.json` and prints a table of pass rate, tokens and wall time per item.

**Commands:**

```powershell
uv run pytest -m agent tests/golden --golden-config configs/dev-claude.yaml
uv run pytest -m agent tests/golden --golden-config configs/dev-ollama.yaml --repeats 3
```

**Verify:**

- Claude scores at least 8 of 10; record the local model's score in `docs/notes/`.
- From now on, every prompt or harness change states its golden scores before and after, in the commit message.

### 2.4 Local tracing (optional, ~2 h)

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

## 8. Phase 3: lab (high level)

**Goal:** measure detection, root cause, report quality, cost and variance on faults with known ground truth, and re-run old harness versions.

1. **Testbed:**
   - Use the OpenTelemetry Demo 3.1 (pinned), with its collector sending OTLP to the same otel-lgtm container. The agent config then changes only in `agent.knowledge`.
   - The demo has Kafka, which is closer to the real event-driven system, and 20 flagd fault flags, e.g. `paymentFailure`, `cartFailure`, `productCatalogFailure`, `kafkaQueueProblems`, `kafkaConsumerDead`, `paymentUnreachable`, `emailMemoryLeak` and `adHighCpu`.
   - It needs several GB of RAM, so run it on the bigger machine.
   - Add the `tempo` MCP category for root-cause work (traces show which downstream call failed).
   - Keep the four-app dev stack for fast iteration.
2. **Fault catalog** (`evals/faults.yaml`):
   - Each entry has an id, an injection (a flag or a script: `docker stop`/`pause`, CPU stress, latency through toxiproxy, log noise), the target service, the expected symptoms, the root cause and the onset delay.
   - Include healthy episodes with no fault, to measure the false-alarm rate.
3. **Episodes:**
   - The protocol is warm-up → inject (the timestamp is the ground truth) → wait for onset → run the agent → remove the fault → cool down.
   - Recommended: **frozen episodes.** After the fault window, copy otel-lgtm's `/data` volume. To evaluate, start otel-lgtm on that copy and run the agent with a fixed `now` and absolute time ranges. Every model and harness version then sees identical telemetry.
   - Check retention, and check that queries over the stored window still work after a restart.
4. **Inspect AI** (`evals/inspect/`):
   - **Setup:**
     - The dataset is episode × task.
     - A custom solver calls `run_check(config)` as a black box. The model under test is set in our config, not through Inspect's `--model`: Inspect's agent bridge only intercepts OpenAI, Anthropic and Google SDK calls, not ChatOllama.
   - **Scorers:**
     - detection: the right service flagged at warning or above;
     - false alarms: on healthy episodes;
     - root cause: exact component match, plus an LLM judge for the free text;
     - report quality: an LLM judge with a rubric. Claude is acceptable on synthetic data, but validate the judge against about 30 hand-labelled reports first;
     - cost: tokens, wall time and dollars for API models, from `meta.json`.
   - **Running:**
     - Use `--epochs k` for variance and `inspect view` to browse the results.
     - On the laptop, run one sample at a time (`--max-samples 1`), since Ollama serves one request at a time.
5. **Old versions:**
   - For each tag, run `git worktree add ..\harness-<tag> <tag>` and run the eval there with `uv run`. Each tag has its own `uv.lock`.
   - Give every version the same environment section, and add `config_version` to configs so the runner can adapt older schemas.
6. **Statistics:**
   - Run at least 5 epochs per episode and configuration, and report means with confidence intervals.
   - Compare versions and models paired on the same episodes: a paired bootstrap, and McNemar's test for detection.

## 9. Phase 4: improvements (high level)

- **System knowledge** (growing from `agent.knowledge`):
  - A **service map**: static YAML first, later built from Tempo's service graph.
  - **Runbooks**: markdown per service, put in the prompt first; add retrieval only when they no longer fit.
  - **Normal baselines** per service from historical data (e.g. 7-day p95 latency and error ratio). The agent then compares against "normal" instead of guessing. For example, dice-java's 30 % errors become "chronic, unchanged" rather than a new critical.
  - The operators' existing dashboards are a ready source of what to check (`get_dashboard_panel_queries`).
- **Explicit LangGraph workflow:**
  - Nodes: deterministic data collection (fixed RED queries per known service) → LLM triage → a bounded investigation subgraph (the tool loop) → structured report → verification (re-run the evidence queries).
  - `create_agent` already returns a LangGraph graph, so this is a refactor behind the same `run_check`.
- **Model comparisons** on the bigger machine: model sizes, quantizations, `num_ctx`, thinking on/off, with Claude as the ceiling.
- **Research idea** (pick one):
  - (a) workflow vs autonomous agent for routine checks: accuracy, cost, variance;
  - (b) do baselines reduce false alarms?
  - (c) tool-set size and tool selection for small models;
  - (d) verifying evidence to prevent invented findings;
  - (e) context strategies under a small `num_ctx`: truncation, context editing, summarization.

## 10. Assumptions

1. Dev telemetry is synthetic, and sending it to the Anthropic API is acceptable (you confirmed this). Real telemetry only goes to models on machines you control.
2. Reports are in English. The prompts are files, so switching language is cheap.
3. There is one Grafana org with stable datasource UIDs (`prometheus`, `loki`, `tempo` in otel-lgtm).
4. The laptop setup is Windows 11, PowerShell 7, Docker Desktop with 16 GB assigned, 32 GB RAM, a Quadro T2000 with 4 GB, and uv-managed Python 3.13.
5. PromQL that works against Prometheus in dev also works against the real system's Mimir.
6. One person works on this part-time; the effort figures are rough.

## 11. Risks

| # | Risk | Impact | Mitigation | Checked in |
|---|---|---|---|---|
| R1 | A Viewer token can't list datasources in Grafana OSS. The docs disagree on whether Viewer has `datasources:read`. | `list_datasources` fails | Datasource UIDs come from config anyway. If queries fail too, use an Editor token in dev only (still with `--disable-write` and the allowlist), and re-test Viewer on the real Grafana. | 1.1 |
| R2 | Running the MCP server through Docker stdio from Python on Windows misbehaves (env, pipes, leftover containers) | No tools | Native launcher: `uv add mcp-grafana==2.2.0` (it has a win_amd64 wheel), then `command: uv`, `args: [run, mcp-grafana]`, `grafana.url: http://localhost:3000` | 1.2 |
| R3 | `langchain.mcp` is beta and its API may change | Breakage on upgrade | Pinned in `uv.lock`; MCP code in one place; upgrade deliberately. Fallback: `langchain-mcp-adapters==0.3.2` in its own environment (it pins `mcp<2`). | 1.2 |
| R4 | The small local model can't use the tools (bad arguments, loops, invented tool names) | Weak local results | Allowlist, call budget, output caps, environment facts in the prompt, ToolErrorMiddleware. The Claude baseline separates harness bugs from model limits. The bigger machine comes later, and the Phase 4 workflow asks less of the model. | 1.5–1.6 |
| R5 | The context overflows and Ollama truncates silently | Wrong reports | Explicit `num_ctx`, tool output cap, Loki line limit, a warning on every model call | 1.5 |
| R6 | 4 GB of VRAM makes runs slow (CPU offload) | Slow iteration, few eval runs | Claude for dev, small models, q8_0 KV cache; the step 1.6 comparison measures it; Phase 3 evals run on the bigger machine | 1.6 |
| R7 | Data leaves the machine (cloud model names, MCP usage stats, LangSmith, Phoenix or Grafana analytics) | Breaks a hard constraint | Each one disabled explicitly, plus the config guard and tests | 1.0, 1.4 |
| R8 | Golden tests are flaky on live data | Noisy scores | Ground truth computed at test time with tolerances; repeated runs; frozen episodes in Phase 3 | 2.3 |
| R9 | The prompt overfits the toy dev stack | Results don't transfer | Environment facts kept in config, a generic prompt; OTel Demo in Phase 3 | 1.6, Phase 3 |
| R10 | Docker Desktop can't build from a Git URL | No example apps | Shallow clone into a git-ignored folder and build from there | 1.1 |
| R11 | Flash attention isn't active on the T2000, so KV-cache quantization is ignored | Less context fits | Check the Ollama log and `ollama ps`; lower `num_ctx` | 1.6 |
| R12 | Access to the real system takes long (network/VPN from Docker, approvals, personal data in logs) | Phases 3–4 on real data are delayed | Start approvals early; use the native MCP launcher; treat transcripts of real runs as sensitive and never commit them | later |

## 12. Open questions

None of these block Phases 1–2.

1. What exactly do operators check each day: which dashboards, queries and alerts? A copy of that routine becomes the checklist and golden set for the real system.
2. What does the real stack look like: Mimir tenants or orgs, label conventions (`service.name`?), the message broker, rules-engine metrics? This decides how representative the OTel Demo is.
3. Do you have approval to run the agent on real telemetry? If logs contain personal data, a GDPR/DPIA assessment may be needed. And where may the model run: only the laptop, or also the bigger machine?
4. The bigger machine: what GPU and VRAM does it have, and will Ollama there be reached over the network?
5. Should operators get the report in Norwegian or English?
6. What are the thesis milestones? When Phase 3 results are due decides how much of Phase 4 is in scope.

## Appendix A: versions and sources checked 2026-11-06

| Component | Version | Note |
|---|---|---|
| langchain / langchain-core / langgraph | 1.4.9 / 1.7.6 / 1.2.20 | `create_agent`, middleware; `langchain.mcp` (beta) needs ≥ 1.4.0 |
| langchain-ollama | 1.1.0 | `ChatOllama(num_ctx, reasoning, seed, temperature, keep_alive, base_url)` |
| langchain-anthropic | 1.7.10 | via the `langchain[anthropic]` extra |
| fastmcp | 4.1.6 | via the `langchain[mcp]` extra; `StdioTransport(keep_alive=True)` |
| langchain-mcp-adapters | 0.3.2 | replaced; pins `mcp<2`, so not used |
| mcp-grafana | 2.2.0 | `grafana/mcp-grafana:2.2.0`; PyPI wheel includes win_amd64; usage stats on by default |
| grafana/otel-lgtm | 0.40.0 | Prometheus (not Mimir), Loki, Tempo, Pyroscope; no service account since 0.36 |
| Ollama | 0.40.13 (installed) | 4k default context below 24 GiB VRAM; `OLLAMA_NO_CLOUD` |
| inspect-ai | 0.3.287 | agent bridge covers OpenAI/Anthropic/Google SDKs only |
| arize-phoenix / openinference-instrumentation-langchain | 21.0.0 / 0.1.82 | Phase 2.4 |
| OpenTelemetry Demo | 3.1.0 | 20 flagd fault flags, including Kafka faults |
| uv | 0.13.7 | not installed yet |

Sources:

- LangChain [agents](https://docs.langchain.com/oss/python/langchain/agents), [MCP](https://docs.langchain.com/oss/python/langchain/mcp), [migration from langchain-mcp-adapters](https://docs.langchain.com/oss/python/migrate/langchain-mcp-adapters), [built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in), [structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [FastMCP transports](https://gofastmcp.com/clients/transports)
- [mcp-grafana](https://github.com/grafana/mcp-grafana) (README, `docs/sources/configure/command-line-flags.md`)
- [docker-otel-lgtm](https://github.com/grafana/docker-otel-lgtm) (README, `docs/gcx-migration.md`, `examples/`)
- Ollama [context length](https://docs.ollama.com/context-length) and [FAQ](https://docs.ollama.com/faq)
- [Inspect agent bridge](https://inspect.aisi.org.uk/agent-bridge.html)
- [Grafana basic-role permissions](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/access-control/rbac-fixed-basic-role-definitions/)
- [OTel Demo flags](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/flagd/demo.flagd.json)
- [Phoenix configuration](https://arize.com/docs/phoenix/self-hosting/configuration)

## Appendix B: where the brief's hooks live

| Hook | Where | Step |
|---|---|---|
| 1. Config file (Grafana URL, model, `num_ctx`, tools, prompt) | `configs/*.yaml` + `config.py` | 1.2, grows in 1.4–1.5 |
| 2. One entry point | `async def run_check(config) -> RunResult`, called by the CLI, pytest and Inspect | 1.2 |
| 3. Reproducibility (config, git commit, exact model) | `runs/<id>/config.json`, `meta.json` (commit, dirty flag, model digest), `git.diff` | 1.3 |
| 4. Transcript as JSON | `runs/<id>/transcript.json`, plus `tools.json` | 1.3 |
