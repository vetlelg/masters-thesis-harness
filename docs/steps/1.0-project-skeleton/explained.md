# 1.0 Prerequisites and project skeleton: explained

What step 1.0 builds and why it is built that way. The exact steps are in [guide.md](guide.md); the overall design is in [architecture.md](../../architecture.md).

## What you have after this step

A Python project that can be installed and run as a command (`monitor`), with locked dependencies, a git history, a place for secrets that never gets committed, and an Ollama installation that can't send prompts off the machine. There is no agent yet. Step 1.0 lays the foundation every later step stands on.

## uv: one tool for Python, environments and dependencies

uv replaces several tools at once: installing Python versions, creating the virtual environment (`.venv`), resolving and installing packages, and running commands inside the environment.

- **`uv run <command>`** first makes sure `.venv` matches `pyproject.toml` and `uv.lock`, then runs the command. You never need to activate the environment by hand. If someone adds a dependency, the next `uv run` installs it.
- **`uv.lock`** records the exact version and file hash of every package, including indirect ones such as `langgraph` and `fastmcp`. `pyproject.toml` says what you need (`langchain>=1.4.9`); `uv.lock` says exactly what you got. The lock file is what makes an old git tag runnable again in Phase 3. That is also why step 1.3 stores a hash of `uv.lock` in every run record.
- **Python 3.13 managed by uv.** Your system has Python 3.14, but uv downloads and uses its own 3.13. 3.13 is the safer choice because some packages publish Windows wheels for a new Python version late. `.python-version` pins it, so everyone running this repo, including you on the bigger machine later, gets the same version.

## The project layout: a package with an entry point

`uv init --package` creates a *src layout*: the code lives in `src/monitor_agent/`, and the project is installed into `.venv` like any other package.

- **Why a package instead of loose scripts?** Imports behave the same everywhere. The CLI, the tests (from step 1.4) and Inspect AI (Phase 3) all `import monitor_agent` the same way, from the installed package. There are no `sys.path` tricks, and no "works from this folder only" bugs.
- **`[project.scripts]`** turns `monitor = "monitor_agent.cli:main"` into a real `monitor.exe` inside `.venv\Scripts`. `uv run monitor check` runs the `main()` function in `cli.py`.
- **`cli.py` is a stub on purpose.** The command name and its subcommands are the user-facing interface. Creating it now means every later step only fills it in. In step 1.2, `check` loads a config and calls `run_check(config)`, the single entry point that tests and the eval harness will also call.
- **argparse** comes with Python. It is enough for a few subcommands and options, and adds no dependency.

## The dependencies and what each one is for

| Package | Role in the harness | First used in |
|---|---|---|
| `langchain` | `create_agent` (builds the agent loop as a LangGraph graph), middleware, `init_chat_model` | 1.2 |
| `langgraph` (pulled in by langchain) | runs the agent graph: model call → tool calls → model call … | 1.2 |
| extra `mcp` → `fastmcp` | starts the Grafana MCP server and turns its tools into LangChain tools (`langchain.mcp.MCPAdapter`) | 1.2 |
| extra `ollama` → `langchain-ollama` | `ChatOllama`: talks to the local Ollama server | 1.2 |
| extra `anthropic` → `langchain-anthropic` | `ChatAnthropic`: the Claude dev model | 1.2 |
| `pydantic` | validates the config file; later the structured report schema | 1.2, 2.2 |
| `pyyaml` | reads the YAML config files | 1.2 |
| `python-dotenv` | loads secrets from `.env` into environment variables | 1.2 |
| `pytest` (dev group) | tests; in the `dev` group because the agent doesn't need it to run | 1.4 |

The `[mcp,ollama,anthropic]` part is called *extras*: optional dependency sets that LangChain defines. Asking for them installs the matching integration packages with versions LangChain is known to work with.

## Secrets: `.env` and `.env.example`

- **`.env`** holds real secret values: the Grafana token and the Anthropic key. It is git-ignored, so it never ends up in the history. Once a secret is in git it is hard to remove, even after deleting the file.
- **`.env.example`** is the committed template. It documents which variables exist and why, without any values.
- **Config files never contain secrets.** From step 1.2 they only *name* the environment variables. That way a config file can be copied into a run record (hook 3) without leaking anything.
- `LANGSMITH_TRACING=false` is there as a safety net. LangChain sends traces to LangSmith, a cloud service, when this variable is true. Setting it explicitly to false protects the "telemetry must not leave my machine" rule against a stray global setting.

## Git from day one

Hook 3 says every run records the git commit it ran from. That only works if there are commits, so the repo starts in step 1.0 (`uv init` runs `git init` for you). Milestone tags such as `v0.1.0` in step 1.2 mark versions you can re-run in Phase 3.

`.gitignore` keeps out what shouldn't be versioned. It is created before `uv init`, and uv only writes its own when none exists, so the project's version is kept:

- **Python build output** (`__pycache__/`, `*.py[oc]`, `build/`, `dist/`, `wheels/`, `*.egg-info`): generated files. These are the same entries uv would have written.
- **`.venv`:** the virtual environment, rebuilt from `uv.lock` with `uv sync`. It differs per machine (laptop, bigger machine), so it must never be shared.
- **`.env` and `.env.*`, except `.env.example`:** secrets (constraint C3). The `.env.*` pattern also covers variants you may create later, such as a `.env.real` with the token for the real Grafana. The `!` line re-includes the template, so `.env.example` stays versioned.
- **`runs/`:** the run records. They can contain telemetry and grow fast; you archive the runs you cite in the thesis separately.
- **`.pytest_cache/`:** pytest's cache (from step 1.4).
- **`deploy/dev/vendor/`:** a local clone of the otel-lgtm example apps, used only if Docker can't build them from the Git URL (step 1.1).
- **`.claude/settings.local.json`:** your personal Claude Code permission settings. The shared `CLAUDE.md` is committed.
- **Editor and OS files** (`.vscode/`, `.idea/`, `Thumbs.db`, `desktop.ini`, `.DS_Store`): machine-specific clutter.

Just as important is what is *not* ignored. `uv.lock` and `.python-version` are committed, because together they let any machine, or an old git tag, rebuild exactly the same environment.

## Ollama settings

- **`OLLAMA_NO_CLOUD=1`:** recent Ollama versions can run "cloud models", which send your prompt to ollama.com. The agent's prompts will contain logs and metrics, so a cloud model would quietly break the rule that telemetry stays local. This setting turns the cloud features off at the server. Step 1.4 adds a second guard in the harness itself.
- **`OLLAMA_KV_CACHE_TYPE=q8_0`:** while generating, a model keeps a key/value cache for every token in its context window. This cache grows with `num_ctx`, and on a 4 GB GPU it competes with the model weights for space. Storing it as 8-bit numbers instead of 16-bit roughly halves its size, with a small loss in precision. That leaves room for a larger context or a larger share of the model on the GPU. It only takes effect when flash attention is active, which you check in step 1.6.
- Both are read when Ollama starts, which is why you restart it.

## How this connects to the next steps

- **1.1** adds the dev telemetry stack (Docker) and writes the Grafana token into `.env`.
- **1.2** turns `monitor check` into a real agent run: it reads a config file, starts the Grafana MCP server, and asks the model for a health report.
- From **1.2** on, every step must leave `monitor check` working (the definition of done in [architecture.md, section 7](../../architecture.md#7-how-we-work)). Step 1.0 is the only step where the command is still a stub.
