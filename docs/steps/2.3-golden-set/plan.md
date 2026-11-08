# 2.3 Golden set v1 (~1–1.5 days): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


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
