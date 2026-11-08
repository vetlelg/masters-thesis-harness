# 2.2 Structured report (~1 day): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


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
