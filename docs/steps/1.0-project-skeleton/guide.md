# 1.0 Prerequisites and project skeleton: guide

Exact steps for roadmap step 1.0. Why each part exists is in [explained.md](explained.md); the overall design is in [architecture.md](../../architecture.md).

- Commands are PowerShell 7, run from the repo root (`C:\git\masters-thesis-harness`).
- File blocks say whether you **create**, **replace** or **edit** a file. "Create" and "replace" blocks show the complete file content.
- Library versions are the ones current on 2026-11-06. If `uv` resolves newer versions, that is fine unless a step says otherwise.

**Result:** uv is installed, Ollama runs local-only, and the repo is a Python project with a `monitor` command that prints help. Nothing talks to Grafana or a model yet.

**Time:** about 1 hour, mostly downloads.

## 1.0.1 Install uv

```powershell
winget install --id astral-sh.uv -e
```

Close the terminal and open a new one, so `PATH` includes uv. Then check:

```powershell
uv --version
```

Expected: `uv 0.13.7` or newer.

## 1.0.2 Configure Ollama: local-only and a smaller KV cache

These are user-level environment variables. Ollama reads them only when it starts.

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_NO_CLOUD", "1", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_KV_CACHE_TYPE", "q8_0", "User")
```

Restart Ollama:

1. Right-click the Ollama icon in the system tray and choose **Quit Ollama**.
2. Start Ollama again from the Start menu.

Check that the cloud features are off:

```powershell
Select-String -Path "$env:LOCALAPPDATA\Ollama\server.log" -Pattern "cloud disabled"
```

Expected: a line containing `Ollama cloud disabled: true`. If nothing is printed, Ollama wasn't restarted, or it started before the variables were set. Check the variable with `[Environment]::GetEnvironmentVariable("OLLAMA_NO_CLOUD", "User")`, which should print `1`, and restart Ollama again.

`OLLAMA_KV_CACHE_TYPE` only shows up in the log once a model is loaded. You check it in step 1.6.

## 1.0.3 Create the project

```powershell
uv init --package --name monitor-agent --python 3.13
```

This runs `git init` (the folder isn't a repo yet) and creates:

```text
.git/
.python-version          contains: 3.13
README.md                empty
pyproject.toml
src/monitor_agent/__init__.py
```

uv also writes a `.gitignore`, but only when none exists. The project's `.gitignore` is already in the repo, so uv leaves it alone (see 1.0.8).

If uv says there is already a `pyproject.toml`, you ran the command twice. Nothing is broken; carry on with the next step.

## 1.0.4 Add the dependencies

```powershell
uv add "langchain[mcp,ollama,anthropic]" pydantic pyyaml python-dotenv
uv add --dev pytest
```

The first `uv add` downloads Python 3.13 if it isn't installed yet (uv manages its own Python), creates `.venv\`, and writes `uv.lock`. Expect a minute or two.

## 1.0.5 Edit `pyproject.toml`

**Edit** two things in the file uv generated:

1. Replace `description = "Add your description here"` with a real description.
2. Replace the script entry `monitor-agent = "monitor_agent:main"` with `monitor = "monitor_agent.cli:main"`.

The file should then look like this. Your `authors` line comes from your git config, and dependency order and versions may differ slightly.

```toml
[project]
name = "monitor-agent"
version = "0.1.0"
description = "Read-only monitoring agent harness for a Grafana LGTM stack (master's thesis)"
readme = "README.md"
authors = [
    { name = "Your Name", email = "you@example.com" }
]
requires-python = ">=3.13"
dependencies = [
    "langchain[anthropic,mcp,ollama]>=1.4.9",
    "pydantic>=2.14.3",
    "python-dotenv>=1.2.4",
    "pyyaml>=6.0.3",
]

[project.scripts]
monitor = "monitor_agent.cli:main"

[build-system]
requires = ["uv_build>=0.13.7,<0.14"]
build-backend = "uv_build"

[dependency-groups]
dev = [
    "pytest>=9.2.0",
]
```

Leave the `[build-system]` section exactly as uv wrote it.

## 1.0.6 Replace `src/monitor_agent/__init__.py`

**Replace** the generated `main()` function, which is no longer the entry point, with a module docstring:

```python
"""Monitoring agent harness: a read-only LangGraph agent that checks a Grafana LGTM stack."""
```

## 1.0.7 Create `src/monitor_agent/cli.py`

**Create:**

```python
"""Command-line interface: `uv run monitor <command>`."""

import argparse


def main() -> None:
    parser = argparse.ArgumentParser(
        prog="monitor",
        description="Read-only monitoring agent for a Grafana LGTM stack.",
    )
    commands = parser.add_subparsers(dest="command", required=True)
    commands.add_parser("check", help="run the daily health check and print a markdown report")

    args = parser.parse_args()

    if args.command == "check":
        print("monitor check: not implemented yet (step 1.2)")
```

## 1.0.8 Check `.gitignore`

Nothing to write: `.gitignore` already exists in the repo root, and `uv init` kept it. Open it and read it. Each entry is explained in [explained.md](explained.md#git-from-day-one). The most important entries are:

- `.env` and `.env.*` (except `.env.example`): secrets never get committed;
- `.venv`: the environment is rebuilt from `uv.lock`;
- `runs/`: run records can contain telemetry.

`uv.lock` and `.python-version` are deliberately *not* ignored; they make the project reproducible. Step 1.0.11 checks that the ignore rules work.

## 1.0.9 Create `.env.example` and `.env`

**Create** `.env.example`. Put comments on their own lines, not after a value; a trailing comment can end up as part of the value.

```dotenv
# Copy to .env (git-ignored) and fill in. Never commit .env.

# Grafana service account token with the Viewer role.
# Written by scripts/grafana_viewer_token.py in step 1.1.
GRAFANA_SERVICE_ACCOUNT_TOKEN=

# Anthropic API key. Only for configs with provider: anthropic, and only against synthetic dev data.
# Needed from step 1.2.
ANTHROPIC_API_KEY=

# LangSmith is a cloud tracing service. Keep it off: telemetry must not leave this machine.
LANGSMITH_TRACING=false

# Config file used when `monitor` runs without --config (from step 1.2).
MONITOR_CONFIG=configs/dev-claude.yaml
```

Then make your own copy:

```powershell
Copy-Item .env.example .env
```

The values stay empty for now. Step 1.1 fills in the token, and you add the API key in step 1.2.

## 1.0.10 Write `README.md`

**Replace** the empty file. Note that the README itself contains a code block:

````markdown
# monitor-agent

A read-only AI agent that runs a daily health check of a system observed with the Grafana LGTM stack
(Loki, Grafana, Tempo, Mimir/Prometheus) and writes a markdown report. It uses LangChain/LangGraph,
the Grafana MCP server, and either a local Ollama model or Claude (synthetic dev data only).

Part of a master's thesis in AI Engineering. See [the architecture](docs/architecture.md) and the [step guides](docs/steps/).

## Requirements

- Windows 11 with PowerShell 7
- [uv](https://docs.astral.sh/uv/)
- Ollama, started with `OLLAMA_NO_CLOUD=1`
- Docker Desktop (from step 1.1)

## Quickstart

```powershell
uv sync                       # create .venv and install the locked dependencies
Copy-Item .env.example .env   # then fill in the values
uv run monitor --help
```

## Status

Step 1.0: project skeleton. `monitor check` is not implemented yet.
````

## 1.0.11 Verify

```powershell
uv run monitor --help
```

Expected:

```text
usage: monitor [-h] {check} ...

Read-only monitoring agent for a Grafana LGTM stack.

positional arguments:
  {check}
    check     run the daily health check and print a markdown report

options:
  -h, --help  show this help message and exit
```

```powershell
uv run monitor check
```

Expected: `monitor check: not implemented yet (step 1.2)`

```powershell
uv run python -c "import langchain.mcp, langchain_ollama, langchain_anthropic; print('ok')"
```

Expected: `ok`. A `LangChainBetaWarning` about `langchain.mcp` may appear above it. That is expected, because the MCP integration is still in beta.

```powershell
git status
```

`.env`, `.venv` and `__pycache__` must **not** be listed. `.env.example`, `.python-version`, `uv.lock` and the other new files should be listed.

## 1.0.12 Commit

```powershell
git add -A
git commit -m "Bootstrap project: uv package, monitor CLI stub, env template"
```

This also commits `CLAUDE.md` and `docs/`, including the old `docs/brief.md` and `docs/implementation-plan.md`. Committing them once keeps them in the git history.

## 1.0.13 Remove the old planning documents

`docs/architecture.md`, `CLAUDE.md` and the step folders under `docs/steps/` replace the brief and the implementation plan. Remove the old files in a separate commit:

```powershell
git rm docs/brief.md docs/implementation-plan.md
git commit -m "Remove brief and plan, replaced by CLAUDE.md, architecture.md and step docs"
```

You can still read them later from the history, for example with `git show HEAD~1:docs/implementation-plan.md`.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `uv: The term 'uv' is not recognized` | The terminal was opened before installing. Open a new terminal. |
| `error: Failed to spawn: monitor` | The `[project.scripts]` edit wasn't saved, or the name is misspelled. Fix `pyproject.toml`, then run `uv sync` and try again. |
| `ModuleNotFoundError: No module named 'monitor_agent.cli'` | `cli.py` isn't in `src\monitor_agent\`, or the file name is wrong. |
| `uv add` fails to resolve | Check the quotes around `"langchain[mcp,ollama,anthropic]"`. PowerShell needs them because of the brackets. |

## Done when

- [ ] `uv --version` prints 0.13.7 or newer
- [ ] The Ollama log contains `cloud disabled`
- [ ] `uv run monitor --help` shows the `check` command
- [ ] `uv run monitor check` prints the "not implemented yet" message
- [ ] The import check prints `ok`
- [ ] `git status` doesn't list `.env` before committing
- [ ] The first commit exists (`git log --oneline` shows it)
- [ ] `docs/brief.md` and `docs/implementation-plan.md` are removed in a second commit
