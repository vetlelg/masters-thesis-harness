# CLAUDE.md

Master's thesis project (AI Engineering): a read-only AI agent that runs the daily health check of a rule-based, event-driven microservices system in the Norwegian public sector, observed with OpenTelemetry and the Grafana LGTM stack. It uses LangChain `create_agent` (LangGraph), the Grafana MCP server and a local Ollama model. Claude is used only for development, on synthetic data. The agent harness will be evaluated scientifically.

Read `docs/architecture.md` when you need the purpose, constraints, architecture, decisions or roadmap.

## How we work

- **The user writes all the code.** Don't create or edit files in `src/`, `tests/`, `configs/`, `prompts/`, `scripts/`, `deploy/`, `evals/` or `pyproject.toml` unless the user asks for it in the current message. Explain instead: point to `file:line`, and show code in the chat or in the step docs.
- **Claude writes the step docs** when asked (see "Step docs" below).
- **The code is the truth.** The user's code may differ from the guides. Read it before answering questions or writing the next step.
- **Don't answer API questions from memory.** The libraries in this project are newer than your training data: LangChain 1.4 (with `langchain.mcp` and middleware), FastMCP 4, mcp-grafana 2.x, Ollama 0.40. Check the installed source under `.venv\Lib\site-packages\` or the current docs first.
- Never print secrets from `.env`. To check whether a variable is set, don't show its value.
- The user commits. Don't commit unless asked.

## Hard constraints

- **Read-only toward Grafana:** Viewer token, MCP server with `--disable-write`, only the needed tool categories, and write tools refused at startup. Never suggest write access.
- **Telemetry must not leave the machine.** The Claude API is only for the synthetic dev and lab stacks; real telemetry only goes to local models. No LangSmith, no Ollama cloud models; analytics and usage statistics stay off.
- **Every step leaves `uv run monitor check` working.**
- **Keep it simple:** no abstractions before a step needs them.

## Environment

- Windows 11, PowerShell 7. Run commands from the repo root. Python 3.13 is managed by uv; run everything through `uv run`.
- The laptop GPU has 4 GB of VRAM (small local models, about 16k context). A bigger machine comes later.
- Commands, available once the step that adds them is done:
  - `docker compose -f deploy/dev/compose.yaml up -d`: the dev telemetry stack (1.1), with Grafana at http://localhost:3000
  - `uv run monitor check --config configs/dev-claude.yaml`: one agent run (1.2)
  - `uv run pytest`: the fast tests (1.4)

## Step docs

Each roadmap step has a folder `docs/steps/<n>-<slug>/`:

- `plan.md`: the planned design, until the step is written up.
- `guide.md`: exact instructions. Result and time; numbered sub-steps (`1.2.1`, …) with commands and complete file contents marked **create**, **replace** or **edit**; verification with expected output; troubleshooting; a "Done when" checklist.
- `explained.md`: organized by topic, in the order the guide introduces them. What each part is for, why it is built that way, the concepts involved, and how it connects to other steps. Written so it can feed the thesis.

To write up a step:

1. Read the code the user built in the earlier steps.
2. Check the current docs and versions of the libraries the step uses.
3. Write `guide.md` and `explained.md` from `plan.md`, then delete `plan.md`.
4. Update the step's status in the roadmap in `docs/architecture.md`, plus any other section the step changes.

Use `docs/steps/1.0-project-skeleton/` as the model for structure and style.
