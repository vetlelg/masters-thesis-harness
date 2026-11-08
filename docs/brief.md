# Brief: implementation plan for a monitoring agent (master's thesis)

> **Replaced (2026-11-08)** by [architecture.md](architecture.md) and the step docs in [steps/](steps/). Kept only until the first commit; step 1.0.13 removes it.

Create an implementation plan for this project. Don't write implementation code yet: the deliverable is the plan, saved as `docs/implementation-plan.md`. Before writing it, ask me only the questions whose answers would change the plan.

## Context
- Master's thesis in AI Engineering. I'm building an AI agent harness that I will later evaluate scientifically.
- Use case: a monitoring and troubleshooting agent for a rule-based, event-driven microservices system in the Norwegian public sector. The system is instrumented with OpenTelemetry and observed with the Grafana LGTM stack (Loki, Grafana, Tempo, Mimir).
- Today, operators go through logs, metrics and dashboards by hand every day to confirm the system is healthy. The agent should automate and improve that routine check.

## Approach
- MVP first: get a working agent running as fast as possible, then improve the harness in small increments.
- No testbed or lab yet. It comes later, after having a working agent MVP running. The MVP must make adding it cheap (see "Hooks").
- Keep it simple. Start with a simple agent loop and add stuff incrementally as it makes sense to do so. No abstractions I don't need yet.

## Stack
- Python, managed with uv.
- LangGraph via LangChain's `create_agent`, so I can move to a custom LangGraph graph later.
- A local LLM through Ollama (already installed), using `ChatOllama` from `langchain-ollama`. Model name and `num_ctx` come from config.
- Tools via MCP: the official Grafana MCP server (`grafana/mcp-grafana` Docker image, stdio transport, configured with `GRAFANA_URL` and `GRAFANA_SERVICE_ACCOUNT_TOKEN`). Load its tools with `langchain-mcp-adapters` (`MultiServerMCPClient`).
- Telemetry for development: the `grafana/otel-lgtm` all-in-one container plus its example app and traffic generator. Pointing at the real Grafana (read-only) may come later.
- Check the current docs for each library before planning; these APIs change quickly.

## Constraints and known pitfalls
- Read-only: run the Grafana MCP server with `--disable-write` and a Viewer-level service account.
- Enable only the tool categories the agent needs with `--enabled-tools` (e.g. Prometheus, Loki, datasources, search). Small local models choose poorly among many tools.
- Ollama's default context window can be as small as 4,096 tokens, and longer input is cut off silently. Set `num_ctx` explicitly (16k–32k).
- The model must support tool calling (e.g. a qwen3 or gpt-oss model).
- Secrets go in `.env` (git-ignored), with a `.env.example`. Telemetry must not leave my machine.

## Hooks the MVP must include
1. A config file for Grafana URL, model, `num_ctx`, enabled tools and the prompt. Switching environments should be a config change.
2. One entry point, `run_check(config) -> report`. The CLI calls it now; an eval harness will call it later.
3. Reproducibility: every run records its config, the git commit and the exact Ollama model tag.
4. Every run saves its transcript (messages, tool calls, tool results) as JSON.

## Phases
1. MVP: agent + Ollama + Grafana MCP against the local LGTM stack. A CLI command (e.g. `uv run monitor check`) prints a markdown daily health report.
2. Harness basics: a structured report (Pydantic: finding, severity, evidence query, Grafana link, confidence). A golden set of 5–10 questions with known answers, run with pytest. Tracing.
3. Lab: fault injection with ground truth (e.g. OpenTelemetry Demo feature flags or scripted faults) and an evaluation harness in Inspect AI. Re-run old tagged versions. Metrics: detection precision/recall (including false alarms on healthy days), root-cause accuracy, report quality, cost, variance across repeated runs.
4. Improvements: system knowledge (service map, runbooks, normal baselines), an explicit LangGraph workflow, model comparisons, a research idea.

Plan phases 1–2 in detail and phases 3–4 at a high level.

## What the plan should contain
- A proposed repo layout.
- Small steps, each leaving something runnable. For each step: goal, files, commands, and how I verify it works.
- Assumptions, risks and open questions.

## My setup (fill in)
- OS: Windows 11
- GPU / VRAM: NVIDIA Quadro T2000 with Max-Q Design
- Ollama models installed: llama3.1:8b (Can change/update this if necessary)
- Docker available (yes/no): Yes

Checked on 2026-11-06:
- GPU has 4 GB VRAM; CPU i7-10850H (6 cores / 12 threads); 32 GB RAM, of which 16 GB is assigned to Docker Desktop.
- Ollama 0.40.13. Docker Desktop 29.1.3 with Compose v5.0.1. Git 2.53. System Python 3.14. uv is not installed yet.
- A more powerful machine will be available later; the laptop is used at first.
- The Claude API may be used for development on synthetic data only. Real telemetry stays with local models.

Deviations from this brief and the reasons for them are listed in [implementation-plan.md, section 1](implementation-plan.md#1-decisions-and-changes-to-the-brief).