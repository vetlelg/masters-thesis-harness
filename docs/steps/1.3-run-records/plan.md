# 1.3 Run records: reproducibility and transcripts, hooks 3 and 4 (~2 h): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


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
