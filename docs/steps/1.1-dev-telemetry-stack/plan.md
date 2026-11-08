# 1.1 Dev telemetry stack (~1 h, mostly image builds): plan

> **Status: planned, not written up yet.** This is the step's design, moved here unchanged from the original
> implementation plan. When the step is written up, this file becomes `guide.md` (exact steps and code) and
> `explained.md` (why it is built that way), and is deleted. The cross-cutting design (constraints, config shape,
> run flow, decisions) is in [architecture.md](../../architecture.md); risk numbers (R1, …) refer to its
> [risks table](../../architecture.md#9-risks).


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
