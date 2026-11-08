# Architecture

*Master's thesis in AI Engineering: a monitoring and troubleshooting agent harness. Last updated 2026-11-08.*

This document covers what the project is for, the constraints it must meet, how the agent harness is built, the decisions behind it, and the roadmap. The build instructions live in [`steps/`](steps/), with one folder per roadmap step. Each folder holds a `guide.md` (exact steps and code) and an `explained.md` (why it is built that way). A step that hasn't been written up yet has a `plan.md` with its planned design instead.

## 1. Purpose and scope

**The problem.** A rule-based, event-driven microservices system in the Norwegian public sector is instrumented with OpenTelemetry and observed with the Grafana LGTM stack (Loki, Grafana, Tempo, Mimir). Today, operators go through logs, metrics and dashboards by hand every day to confirm that the system is healthy.

**The goal.** An AI agent that automates and improves that daily check. It queries the telemetry through Grafana, judges what is normal and what isn't, and writes a health report an operator can act on. Later it should also help with troubleshooting: answering questions and pointing to likely root causes.

**The thesis angle.** The project builds an agent *harness*: the agent plus what it takes to run it reproducibly and evaluate it scientifically. That means configuration, run records, golden tests, fault injection and an evaluation harness. The harness is the research instrument. The agent's design (prompt, tools, workflow, model) is what gets varied and measured.

**The approach.** MVP first: a working agent as fast as possible, then small increments, with a working agent after every step. There is no lab or testbed until the MVP runs, but the MVP includes the hooks that make adding one cheap (section 4).

| In scope now | Later | Out of scope |
|---|---|---|
| Daily health check against a local dev stack; Claude for development; small local models | Fault-injection lab and evaluation (Phase 3); the real Grafana, read-only; system knowledge, an explicit workflow, model comparisons (Phase 4) | Any write action (remediation, alert or dashboard changes); running the agent as a hosted service |

## 2. Constraints

| # | Constraint | How it is enforced |
|---|---|---|
| C1 | **Read-only toward Grafana** | Viewer service account. The MCP server runs with `--disable-write` and only the tool categories it needs (`--enabled-tools`). A client-side tool allowlist, and the harness refuses to start if a write tool is loaded (step 1.4). The safety flags are added in code, never read from config. |
| C2 | **Telemetry must not leave the machine** | Real telemetry only goes to local models: with `environment: real`, a config guard requires `provider: ollama` and rejects cloud models. The Claude API is only used with the synthetic dev and lab stacks. Also turned off: Ollama cloud features (`OLLAMA_NO_CLOUD=1`), mcp-grafana usage statistics (`--usage-stats=disabled`; on by default), LangSmith, and Grafana and Phoenix analytics. Tracing stays local. |
| C3 | **Secrets only in `.env`** | `.env` is git-ignored and documented by `.env.example`. Config files name environment variables, never their values, so a config can be copied into a run record safely. |
| C4 | **Small local models** | The laptop GPU has 4 GB of VRAM, which fits models of about 2–4B parameters with ~16k context. A bigger machine comes later; the Ollama `base_url` is in config. `num_ctx` is always set explicitly, because Ollama defaults to 4k on GPUs under 24 GiB and silently truncates longer input. Models must support tool calling, and the tool set stays small because small models choose poorly among many tools. |
| C5 | **A working agent after every step** | The definition of done (section 7). |
| C6 | **Keep it simple** | No abstractions before a step needs them. Start with three modules (`cli`, `config`, `agent`) and split only when needed. |
| C7 | **Reproducible runs** | Every run records its config, its git commit and the exact model (section 4). |

## 3. Architecture

### 3.1 Overview

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

### 3.2 One run, step by step

1. `cli.py` loads the YAML config into Pydantic models, loads secrets from `.env`, and applies any `--set` overrides.
2. `run_check(config)` creates the run folder and the chat model with `init_chat_model(name, model_provider=…, **params)`, which returns a `ChatAnthropic` or a `ChatOllama`.
3. It starts the Grafana MCP server as a subprocess over stdio and lists its tools. The read-only flags are appended in code, the tools are filtered by the allowlist, and a write tool stops the run.
4. It builds the agent with `create_agent(model, tools, system_prompt=…, middleware=[…])`. The system prompt is a template filled in with the current time, the check window and the environment knowledge from the config.
5. The agent loop runs. The model calls tools (PromQL and LogQL queries through Grafana) until it writes the report. Middleware caps tool output, limits the number of model calls, and warns when the context is nearly full.
6. The run record is written: config, metadata (git, model digest, versions), tools, transcript and report. `run_check` returns a `RunResult`.
7. The CLI prints the report. The tests (Phase 2) and Inspect AI (Phase 3) call `run_check` the same way.

### 3.3 Components

| Part | Responsibility | Added in |
|---|---|---|
| `src/monitor_agent/cli.py` | `monitor check` and `monitor tools`; argument parsing | 1.0, 1.2 |
| `src/monitor_agent/config.py` | Config models, YAML loading, `--set` overrides, guards for C1 and C2 | 1.2, 1.4, 1.5 |
| `src/monitor_agent/agent.py` | `run_check()`: model, MCP tools, agent, run | 1.2 |
| `src/monitor_agent/runlog.py` | Run folder, metadata, transcript | 1.3 |
| `src/monitor_agent/middleware.py` | Tool-output cap, context warnings | 1.5 |
| `src/monitor_agent/report.py` | `HealthReport` schema, Grafana links, grounding check, markdown rendering | 2.2 |
| `src/monitor_agent/tracing.py` | Opt-in local tracing with Phoenix | 2.4 |
| `prompts/` | System prompt and task templates | 1.2, 1.6 |
| `configs/` | One YAML file per environment and model | 1.2 |
| `deploy/dev/` | Dev telemetry stack (Docker Compose) | 1.1 |
| `scripts/grafana_viewer_token.py` | Creates the Viewer service account and token, and checks its permissions | 1.1 |

### 3.4 Key mechanics, from the current library docs

This sketch was checked against the docs on 2026-11-06. It is not the final code:

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

- `MCPAdapter` tools open an MCP session on every call unless the adapter context stays open. FastMCP's `StdioTransport` reuses its subprocess (`keep_alive=True`), so keeping the context open means one `docker run` per run instead of one per tool call.
- FastMCP gives stdio servers only a small allowlist of environment variables (on Windows: `PATH`, `USERPROFILE`, `APPDATA`, …). The Grafana URL and token must be passed explicitly. The docker args forward them into the container with `-e GRAFANA_URL -e GRAFANA_SERVICE_ACCOUNT_TOKEN`.
- The `grafana/mcp-grafana` image defaults to SSE, so `-t stdio` and `docker run -i` are required.
- Inside a container, `localhost` means the container itself. On Docker Desktop, the MCP server reaches the dev Grafana at `http://host.docker.internal:3000`. Links meant for people use `http://localhost:3000` (`grafana.public_url`).
- An MCP server error reaches the model as `ToolMessage(status="error")`, so the model can react to it. Transport failures raise exceptions.
- `ChatOllama` accepts `num_ctx`, `temperature`, `seed`, `reasoning` (True, False, or "low"/"medium"/"high"), `keep_alive` and `base_url`. Token usage comes from Ollama's `prompt_eval_count` and `eval_count`, and `done_reason: "length"` means the output was cut off.
- Ollama 0.40 defaults to a 4k context on GPUs under 24 GiB, and it now offers cloud models. `OLLAMA_NO_CLOUD=1` makes it local-only.
- Prompt templates use `string.Template` (`$now`), because the braces in PromQL and LogQL break `str.format`.

### 3.5 Environments

| | Dev (now, laptop) | Lab (Phase 3, bigger machine) | Real (later) |
|---|---|---|---|
| Telemetry | otel-lgtm and four example apps; synthetic | OpenTelemetry Demo with fault injection; synthetic | The production Grafana, read-only |
| Models allowed | Claude, local Ollama | Claude (development, LLM judge), local Ollama | Local Ollama only |
| Config | `configs/dev-*.yaml` | `configs/lab-*.yaml` | `configs/real-readonly.yaml` |

The dev stack is the otel-lgtm all-in-one container, its four example apps each running as its own service, and a traffic generator. The apps behave differently, which gives the agent known answers to find:

| Service | Behaviour | Expected verdict |
|---|---|---|
| dice-java | ~30 % of requests fail with HTTP 500 ("simulating an error"); ~200 ms latency | errors, critical |
| dice-go | ~1 s per request (busy wait) | latency, warning |
| dice-python | healthy | no finding |
| dice-node | HTTP 400 when `?rolls` is missing | client errors, not a server fault |

otel-lgtm stores metrics in Prometheus rather than Mimir. Both use PromQL, so this doesn't matter for the agent. Its datasource UIDs are `prometheus`, `loki`, `tempo` and `pyroscope`.

To use the real environment you write `configs/real-readonly.yaml` with:

- `environment: real` and the real Grafana URLs;
- the real system's knowledge and a 24 h window;
- a local model;
- mcp-grafana's Loki and Prometheus query guardrails (`--loki-guardrail-mode enforce`, `--prometheus-guardrail-mode enforce`), which protect the production backends from expensive queries.

### 3.6 Configuration (hook 1)

This is the target shape at the end of Phase 1. Steps 1.4–1.6 add the fields marked with a step number.

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

Principles:

- A config file plus `.env` describes a run completely. The task is part of the config, so a golden item or eval sample is just a config override.
- Environment facts (datasource UIDs, services, label and metric names, thresholds) live in the config, and the prompts stay generic.
- Secrets are referenced by environment-variable name only.
- `--config` falls back to `MONITOR_CONFIG` from `.env`, so a plain `uv run monitor check` works on every machine.

### 3.7 Run records (hooks 3 and 4)

```text
runs/20261106T101500Z-3f9a1c/
├── config.json       resolved config (environment variable names only, never secret values)
├── meta.json         run id, timing, status, git, model, MCP, versions, counts, tokens
├── tools.json        name, description and input schema of every tool given to the model
├── transcript.json   every message with timestamps: system, user, AI (tool calls, reasoning,
│                     usage, response metadata), tool results
├── git.diff          only if the working tree was dirty
└── report.md         report.json too, from step 2.2
```

`meta.json` records:

- the status: `ok`, `error` or `budget_exhausted`;
- the git commit, branch and dirty flag;
- the model provider and requested name. For Ollama also the server version, model **digest**, quantization and parameter size; for Anthropic, the model id the API returned;
- the MCP image digest and a hash of `tools.json` (tool descriptions are part of the effective prompt);
- the versions of the key packages, and hashes of `uv.lock` and of the rendered prompts;
- counts of model calls, tool calls and tool errors, and token totals.

The transcript is written even when a run fails, so failures can be analysed too.

### 3.8 Repo layout

The target layout at the end of Phase 2. The step that adds each item is in brackets.

```text
masters-thesis-harness/
├── CLAUDE.md                       instructions for Claude Code (loaded automatically)
├── pyproject.toml                  [1.0] uv project; script: monitor = "monitor_agent.cli:main"
├── uv.lock, .python-version        [1.0]
├── .env.example, .gitignore        [1.0] .env and runs/ are git-ignored
├── README.md                       [1.0]
├── configs/                        [1.2] dev-claude.yaml, dev-ollama.yaml (later lab-*, real-readonly)
├── prompts/                        [1.2] system.md, daily_check.md
├── deploy/dev/compose.yaml         [1.1] otel-lgtm + 4 example apps + traffic (+ phoenix profile in 2.4)
├── scripts/grafana_viewer_token.py [1.1]
├── src/monitor_agent/              cli, config, agent [1.0–1.2]; runlog [1.3]; middleware [1.5];
│                                   report [2.2]; tracing [2.4]
├── tests/                          unit tests [1.4–1.5]; integration/ [2.1]; golden/ [2.3]
├── evals/golden/v1.yaml            [2.3] golden items (data; reused by Inspect in Phase 3)
├── docs/
│   ├── architecture.md             this document
│   ├── steps/<n>-<slug>/           guide.md + explained.md per step (plan.md until written up)
│   └── notes/                      results: model comparison, baselines
└── runs/                           git-ignored, one folder per run
```

## 4. Hooks for the evaluation harness

These four hooks are required in the MVP, so that the lab and the evaluation harness can be added cheaply later.

| Hook | Where | Step |
|---|---|---|
| 1. A config file for the Grafana URL, model, `num_ctx`, enabled tools and the prompt; switching environments is a config change | `configs/*.yaml` + `config.py` | 1.2, grows in 1.4–1.5 |
| 2. One entry point: the CLI calls it now, an eval harness later | `async def run_check(config) -> RunResult` in `agent.py` | 1.2 |
| 3. Reproducibility: every run records its config, git commit and exact model | `runs/<id>/config.json`, `meta.json` (commit, dirty flag, model digest), `git.diff` | 1.3 |
| 4. Every run saves its transcript (messages, tool calls, tool results) as JSON | `runs/<id>/transcript.json`, plus `tools.json` | 1.3 |

## 5. Key decisions

| # | Decision | Why |
|---|---|---|
| D1 | The agent loop is LangChain's `create_agent`, which builds a LangGraph graph | A standard loop with middleware now. Moving to a custom LangGraph workflow (Phase 4) is a refactor behind the same `run_check`. |
| D2 | MCP integration through `langchain.mcp.MCPAdapter` from `langchain[mcp]` ≥ 1.4 (FastMCP 4), not `langchain-mcp-adapters` | LangChain 1.4 moved MCP into core and documents it as the replacement. `langchain-mcp-adapters` 0.3.2 pins `mcp<2` and can't share an environment with it. The API is beta, so it is pinned in `uv.lock`, with a fallback (R3). |
| D3 | The official `grafana/mcp-grafana:2.2.0` image over stdio, one container per run; the native PyPI binary as an alternative launcher | Tool descriptions are part of the prompt, so the version is pinned. Stdio ties the server's lifetime to the run. The native launcher avoids Docker networking problems (e.g. on VPN). |
| D4 | Two model providers from day one, chosen in config: Claude for development (synthetic data only) and a local Ollama model as the thesis target | Claude separates harness bugs from small-model weakness and makes iteration fast. The local model is what the thesis evaluates (C2). |
| D5 | The local model is chosen by a measured comparison (step 1.6), not fixed in advance | With 4 GB of VRAM, gpt-oss:20b doesn't fit and 8B models (like the installed llama3.1:8b) spill onto the CPU. The small-model landscape also changes fast (e.g. the qwen3.5 family). |
| D6 | `num_ctx` starts at 16k on the laptop and is set by measurement; tool output and log lines have hard caps | A 32k KV cache doesn't fit in 4 GB next to a 4B model. With the caps, 16k is workable. 32k+ on the bigger machine. |
| D7 | The dev stack runs all four otel-lgtm example apps as separate services, plus a traffic container | Their different behaviour gives the MVP known answers to find. Containers avoid Windows script problems. |
| D8 | A bootstrap script creates the Viewer service account and token | otel-lgtm stopped creating service accounts in v0.36. |
| D9 | `async def run_check(config) -> RunResult`; the task or question is part of the config | MCP tools and Inspect are async. A config snapshot then describes a run completely, and golden and eval items are config overrides. |
| D10 | Environment knowledge lives in config; the prompts stay generic | Small models need the facts written down. Switching environments stays a config change, and the prompt doesn't overfit the dev stack. |
| D11 | Safety flags are added in code, never read from config | A config typo can't turn off read-only mode. |
| D12 | Grafana links in reports are built by code from the evidence query | LLMs garble URLs; links built in code are testable. |
| D13 | Structured output uses `ToolStrategy` for every provider, with a constrained-decoding fallback for small models | Model comparisons aren't skewed by provider-specific structured output. |
| D14 | The golden set is YAML data with a pytest runner; ground truth is queried from Prometheus and Loki at test time | Live data drifts. The same data file is reused by Inspect in Phase 3. |
| D15 | Tracing uses local Phoenix, opt-in; it is never exported into the monitored stack, and never to LangSmith | C2. Agent traces inside the monitored stack would show up in the agent's own list of services. |
| D16 | Phase 3 uses frozen episodes: the LGTM data is snapshotted per fault episode and replayed with a fixed "now" | Every model and harness version sees identical telemetry, so comparisons are fair and cheap. |

## 6. Roadmap

Effort figures are rough, for one person part-time.

### Phase 1: MVP (about 3 days)

| Step | Result | Docs | Status |
|---|---|---|---|
| 1.0 Prerequisites and project skeleton | uv project, `monitor` CLI stub, Ollama local-only, first commit | [1.0](steps/1.0-project-skeleton/) | written |
| 1.1 Dev telemetry stack | otel-lgtm + four example apps + traffic; Viewer token | [1.1](steps/1.1-dev-telemetry-stack/) | planned |
| 1.2 Walking skeleton (M1) | first agent run: config → model + MCP tools → report; tag `v0.1.0` | [1.2](steps/1.2-walking-skeleton/) | planned |
| 1.3 Run records | hooks 3 and 4: run folder, metadata, transcript; tag `v0.2.0` | [1.3](steps/1.3-run-records/) | planned |
| 1.4 Safety guards and first unit tests | read-only and data-locality checks in code, pytest | [1.4](steps/1.4-safety-guards/) | planned |
| 1.5 Context and loop control | output caps, call budget, context warnings, `--set` | [1.5](steps/1.5-context-and-loop-control/) | planned |
| 1.6 Prompt v1 and model comparison | daily-check prompt v1; local model chosen on numbers; tag `v0.3.0` = **MVP** | [1.6](steps/1.6-prompt-and-model-comparison/) | planned |

### Phase 2: harness basics (about 4–5 days)

| Step | Result | Docs | Status |
|---|---|---|---|
| 2.1 Test tiers and integration checks | unit / integration / agent test tiers; MCP tool snapshot; ground-truth helpers | [2.1](steps/2.1-test-tiers/) | planned |
| 2.2 Structured report | `HealthReport` (finding, severity, evidence, link, confidence); tag `v0.4.0` | [2.2](steps/2.2-structured-report/) | planned |
| 2.3 Golden set v1 | 10 items with live ground truth; scores per model; tag `v0.5.0` | [2.3](steps/2.3-golden-set/) | planned |
| 2.4 Local tracing (optional) | Phoenix, opt-in; tag `v0.6.0` | [2.4](steps/2.4-local-tracing/) | planned |

### Phase 3: lab (high level)

**Goal:** measure detection precision and recall (including false alarms on healthy days), root-cause accuracy, report quality, cost, and variance across repeated runs, on faults with known ground truth. Re-run old harness versions on the same faults.

1. **Testbed:**
   - The OpenTelemetry Demo 3.1 (pinned), with its collector sending OTLP to the same otel-lgtm container. The agent config then changes only in `agent.knowledge`.
   - The demo has Kafka, which is closer to the real event-driven system, and 20 flagd fault flags, e.g. `paymentFailure`, `cartFailure`, `productCatalogFailure`, `kafkaQueueProblems`, `kafkaConsumerDead`, `paymentUnreachable`, `emailMemoryLeak` and `adHighCpu`.
   - It needs several GB of RAM, so it runs on the bigger machine.
   - The `tempo` MCP category is added for root-cause work, since traces show which downstream call failed. The four-app dev stack stays for fast iteration.
2. **Fault catalog** (`evals/faults.yaml`):
   - Each entry has an id, an injection (a flag or a script: `docker stop`/`pause`, CPU stress, latency through toxiproxy, log noise), the target service, the expected symptoms, the root cause and the onset delay.
   - It includes healthy episodes with no fault, to measure the false-alarm rate.
3. **Episodes:**
   - The protocol is warm-up → inject (the timestamp is the ground truth) → wait for onset → run the agent → remove the fault → cool down.
   - Recommended: **frozen episodes** (D16). After the fault window, copy otel-lgtm's `/data` volume. To evaluate, start otel-lgtm on that copy and run the agent with a fixed `now` and absolute time ranges.
   - Check the retention settings, and check that queries over the stored window still work after a restart.
4. **Inspect AI** (`evals/inspect/`):
   - **Setup:** the dataset is episode × task. A custom solver calls `run_check(config)` as a black box. The model under test is set in our config, not through Inspect's `--model`, because Inspect's agent bridge only intercepts OpenAI, Anthropic and Google SDK calls, not ChatOllama.
   - **Scorers:**
     - detection: the right service flagged at warning or above;
     - false alarms: findings on healthy episodes;
     - root cause: exact component match, plus an LLM judge for the free text;
     - report quality: an LLM judge with a rubric. Claude is acceptable on synthetic data, but validate the judge against about 30 hand-labelled reports first;
     - cost: tokens, wall time and dollars for API models, from `meta.json`.
   - **Running:** `--epochs k` for variance, `inspect view` to browse. On the laptop run one sample at a time (`--max-samples 1`), since Ollama serves one request at a time.
5. **Old versions:**
   - For each tag, run `git worktree add ..\harness-<tag> <tag>` and run the eval there with `uv run`. Each tag has its own `uv.lock`.
   - Give every version the same environment section, and add `config_version` to the configs so the runner can adapt older schemas.
6. **Statistics:**
   - At least 5 epochs per episode and configuration; means with confidence intervals.
   - Versions and models compared paired on the same episodes: a paired bootstrap, and McNemar's test for detection.

### Phase 4: improvements (high level)

- **System knowledge**, growing from `agent.knowledge`:
  - a **service map**: static YAML first, later built from Tempo's service graph;
  - **runbooks**: markdown per service, put in the prompt first; retrieval only once they no longer fit;
  - **normal baselines** per service from historical data (e.g. 7-day p95 latency and error ratio). The agent then compares against "normal" instead of guessing; dice-java's 30 % errors, for example, become "chronic, unchanged" rather than a new critical finding;
  - the operators' existing dashboards, a ready source of what to check (`get_dashboard_panel_queries`).
- **An explicit LangGraph workflow:**
  - the nodes: deterministic data collection (fixed RED queries per known service) → LLM triage → a bounded investigation subgraph (the tool loop) → structured report → verification (re-running the evidence queries);
  - `create_agent` already returns a LangGraph graph, so this is a refactor behind the same `run_check`.
- **Model comparisons** on the bigger machine: model sizes, quantizations, `num_ctx`, thinking on or off, with Claude as the ceiling.
- **A research idea**, pick one:
  - (a) a workflow versus an autonomous agent for routine checks: accuracy, cost, variance;
  - (b) do baselines reduce false alarms?
  - (c) tool-set size and tool selection for small models;
  - (d) verifying evidence to prevent invented findings;
  - (e) context strategies under a small `num_ctx`: truncation, context editing, summarization.

## 7. How we work

- **The user writes all the code.** Claude writes the step docs: `guide.md` with exact instructions and code, and `explained.md` with the reasoning. Claude also answers questions along the way. Each step's guide builds on the code the user actually wrote, which may differ from the earlier guides.
- **A step is done when:**
  1. `uv run monitor check --config configs/dev-claude.yaml` produces a report against the dev stack (from step 1.2 on);
  2. `uv run pytest` passes (from step 1.4 on);
  3. the work is committed. Milestones get a git tag (`v0.1.0`, …), because Phase 3 re-runs these tags.
- Run the Ollama config at least once in every step that touches the agent loop. It is slow on the laptop, but it must keep working.
- Put risky changes behind a config switch (e.g. `report.mode`, `tracing.enabled`), so falling back is a config change rather than a revert.
- From step 2.3 on, every prompt or harness change reports its golden scores before and after, in the commit message.
- All commands are PowerShell 7, run from the repo root.

## 8. Assumptions

1. Dev and lab telemetry is synthetic, so sending it to the Anthropic API is acceptable (confirmed). Real telemetry only goes to models on machines you control.
2. Reports are in English. The prompts are files, so switching language is cheap.
3. There is one Grafana org with stable datasource UIDs.
4. PromQL that works against Prometheus in dev also works against the real system's Mimir.
5. One person works on this part-time; the effort figures are rough.

## 9. Risks

| # | Risk | Impact | Mitigation | Checked in |
|---|---|---|---|---|
| R1 | A Viewer token can't list datasources in Grafana OSS. The docs disagree on whether Viewer has `datasources:read`. | `list_datasources` fails | Datasource UIDs come from config anyway. If queries fail too, use an Editor token in dev only (still with `--disable-write` and the allowlist), and re-test Viewer on the real Grafana. | 1.1 |
| R2 | Running the MCP server through Docker stdio from Python on Windows misbehaves (env, pipes, leftover containers) | No tools | Native launcher: `uv add mcp-grafana==2.2.0` (it has a win_amd64 wheel), then `command: uv`, `args: [run, mcp-grafana]`, `grafana.url: http://localhost:3000` | 1.2 |
| R3 | `langchain.mcp` is beta and its API may change | Breakage on upgrade | Pinned in `uv.lock`; MCP code in one place; upgrade deliberately. Fallback: `langchain-mcp-adapters==0.3.2` in its own environment (it pins `mcp<2`). | 1.2 |
| R4 | The small local model can't use the tools (bad arguments, loops, invented tool names) | Weak local results | Allowlist, call budget, output caps, environment facts in the prompt, ToolErrorMiddleware. The Claude baseline separates harness bugs from model limits. The bigger machine comes later, and the Phase 4 workflow asks less of the model. | 1.5–1.6 |
| R5 | The context overflows and Ollama truncates silently | Wrong reports | Explicit `num_ctx`, tool output cap, Loki line limit, a warning on every model call | 1.5 |
| R6 | 4 GB of VRAM makes runs slow (CPU offload) | Slow iteration, few eval runs | Claude for dev, small models, q8_0 KV cache; the step 1.6 comparison measures it; Phase 3 evals run on the bigger machine | 1.6 |
| R7 | Data leaves the machine (cloud model names, MCP usage stats, LangSmith, Phoenix or Grafana analytics) | Breaks constraint C2 | Each one disabled explicitly, plus the config guard and tests | 1.0, 1.4 |
| R8 | Golden tests are flaky on live data | Noisy scores | Ground truth computed at test time with tolerances; repeated runs; frozen episodes in Phase 3 | 2.3 |
| R9 | The prompt overfits the toy dev stack | Results don't transfer | Environment facts kept in config, a generic prompt; OTel Demo in Phase 3 | 1.6, Phase 3 |
| R10 | Docker Desktop can't build from a Git URL | No example apps | Shallow clone into a git-ignored folder and build from there | 1.1 |
| R11 | Flash attention isn't active on the T2000, so KV-cache quantization is ignored | Less context fits | Check the Ollama log and `ollama ps`; lower `num_ctx` | 1.6 |
| R12 | Access to the real system takes long (network/VPN from Docker, approvals, personal data in logs) | Phases 3–4 on real data are delayed | Start approvals early; use the native MCP launcher; treat transcripts of real runs as sensitive and never commit them | later |

## 10. Open questions

None of these block Phases 1–2.

1. What exactly do operators check each day: which dashboards, queries and alerts? A copy of that routine becomes the checklist and golden set for the real system.
2. What does the real stack look like: Mimir tenants or orgs, label conventions (`service.name`?), the message broker, rules-engine metrics? This decides how representative the OTel Demo is.
3. Do you have approval to run the agent on real telemetry? If logs contain personal data, a GDPR/DPIA assessment may be needed. And where may the model run: only the laptop, or also the bigger machine?
4. The bigger machine: what GPU and VRAM does it have, and will Ollama there be reached over the network?
5. Should operators get the report in Norwegian or English?
6. What are the thesis milestones? When Phase 3 results are due decides how much of Phase 4 is in scope.

## Appendix A: setup

Checked on 2026-11-06:

- Windows 11 Pro, PowerShell 7.
- Laptop: NVIDIA Quadro T2000 Max-Q with 4 GB VRAM; Intel i7-10850H (6 cores / 12 threads); 32 GB RAM, of which 16 GB is assigned to Docker Desktop.
- Ollama 0.40.13, with llama3.1:8b installed. Docker Desktop 29.1.3 with Compose v5.0.1. Git 2.53. System Python 3.14; the project uses Python 3.13 managed by uv.
- A more powerful machine will be available later.

## Appendix B: versions and sources (checked 2026-11-06)

| Component | Version | Note |
|---|---|---|
| langchain / langchain-core / langgraph | 1.4.9 / 1.7.6 / 1.2.20 | `create_agent`, middleware; `langchain.mcp` (beta) needs ≥ 1.4.0 |
| langchain-ollama | 1.1.0 | `ChatOllama(num_ctx, reasoning, seed, temperature, keep_alive, base_url)` |
| langchain-anthropic | 1.7.10 | via the `langchain[anthropic]` extra |
| fastmcp | 4.1.6 | via the `langchain[mcp]` extra; `StdioTransport(keep_alive=True)` |
| langchain-mcp-adapters | 0.3.2 | replaced; pins `mcp<2`, so not used |
| mcp-grafana | 2.2.0 | `grafana/mcp-grafana:2.2.0`; the PyPI wheel includes win_amd64; usage stats on by default |
| grafana/otel-lgtm | 0.40.0 | Prometheus (not Mimir), Loki, Tempo, Pyroscope; no service account since 0.36 |
| Ollama | 0.40.13 (installed) | 4k default context below 24 GiB VRAM; `OLLAMA_NO_CLOUD` |
| inspect-ai | 0.3.287 | the agent bridge covers OpenAI/Anthropic/Google SDKs only |
| arize-phoenix / openinference-instrumentation-langchain | 21.0.0 / 0.1.82 | step 2.4 |
| OpenTelemetry Demo | 3.1.0 | 20 flagd fault flags, including Kafka faults |
| uv | 0.13.7 | |

Sources:

- LangChain [agents](https://docs.langchain.com/oss/python/langchain/agents), [MCP](https://docs.langchain.com/oss/python/langchain/mcp), [migration from langchain-mcp-adapters](https://docs.langchain.com/oss/python/migrate/langchain-mcp-adapters), [built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in), [structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [FastMCP transports](https://gofastmcp.com/clients/transports)
- [mcp-grafana](https://github.com/grafana/mcp-grafana) (README, `docs/sources/configure/command-line-flags.md`)
- [docker-otel-lgtm](https://github.com/grafana/docker-otel-lgtm) (README, `docs/gcx-migration.md`, `examples/`)
- Ollama [context length](https://docs.ollama.com/context-length) and [FAQ](https://docs.ollama.com/faq)
- [uv: creating projects](https://docs.astral.sh/uv/concepts/projects/init/)
- [Inspect agent bridge](https://inspect.aisi.org.uk/agent-bridge.html)
- [Grafana basic-role permissions](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/access-control/rbac-fixed-basic-role-definitions/)
- [OTel Demo flags](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/flagd/demo.flagd.json)
- [Phoenix configuration](https://arize.com/docs/phoenix/self-hosting/configuration)
