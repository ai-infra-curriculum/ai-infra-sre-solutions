# Exercise 07 — Grafana Dashboard Authoring for Incident Readiness (Solution)

## 1. Solution overview

Author a Grafana dashboard that an on-call engineer who has never seen the system before can open at 2am and, in under 90 seconds, answer:

1. Is this service currently violating its SLOs?
2. Which direction is the failure — user-facing latency, model quality, GPU hardware, or dependencies?
3. Which deploy, config change, or external event lines up with the breach?

Design principles, borrowed from the SRE Workbook and Grafana's own dashboard best-practices guide:

- **One service per dashboard**; link out to upstream/downstream dashboards. A single mega-board is not navigable.
- **Top row = user-visible SLIs** (TTFT, TPOT, availability, quality signal). Nothing else above the fold.
- **Second row = burn rates and error budgets**, not raw latency. A dashboard without burn rates makes on-call guess at severity.
- **Third row = GPU health** (DCGM) — the most common proximate cause for an LLM latency regression.
- **Fourth row = dependencies and cache**. Fourth, not first, because dependency failures show up as top-row breaches already.
- **Variables** (datasource, env, service, workload, model) at the top, with sane defaults.
- **Annotations** for deploys, feature-flag flips, and known incidents.
- **Every panel has a short description** ("what, how read, what next").

## 2. Worked implementation (dashboard)

The repo ships two artifacts: a provisioned YAML description for teams on Grafana Operator / Helm, and an exported dashboard JSON for teams clicking through the UI.

### 2.1 Variables

```json
{
  "templating": {
    "list": [
      {"name": "datasource", "type": "datasource", "query": "prometheus"},
      {"name": "env", "type": "query", "query": "label_values(up, env)", "current": {"value": "prod"}},
      {"name": "service", "type": "query", "query": "label_values(up{env=\"$env\"}, service)"},
      {"name": "workload", "type": "query", "query": "label_values(vllm_time_to_first_token_seconds_bucket{service=\"$service\"}, workload)", "includeAll": true},
      {"name": "model", "type": "query", "query": "label_values(gen_ai_request_model{service=\"$service\"}, gen_ai_request_model)", "includeAll": true}
    ]
  }
}
```

### 2.2 Row 1 — User-visible SLIs (always above the fold)

| Panel | Query (abbreviated — see Exercise 02 for full) | Panel type |
|---|---|---|
| TTFT p95 | `service:ttft_p95:5m{service="$service"}` | Time series |
| TPOT p95 | `service:tpot_p95:5m{service="$service"}` | Time series |
| Availability | `1 - sum(rate(http_requests_total{service="$service",status=~"5.."}[5m])) / sum(rate(http_requests_total{service="$service"}[5m]))` | Stat, big number |
| Quality SLI | `avg_over_time(model_quality_score{service="$service"}[15m])` | Stat |

Thresholds: set red/yellow/green against the actual SLO target (from mod-103). Example TTFT p95 target 500ms → green <400, yellow <500, red ≥500.

### 2.3 Row 2 — Error budget and burn rates

| Panel | Purpose |
|---|---|
| Error budget remaining | Stat, 0–100%, over the current SLO window |
| Fast burn (1h) | Rate; red if >14 (1h-burn exhausts 30d budget in ~2h) |
| Slow burn (6h) | Rate; red if >6 |
| Burn-rate ratio sparkline | 5m vs 1h quick-look |

This row is why humans open the dashboard; if burn is low, latency variance in Row 1 is not actionable.

### 2.4 Row 3 — GPU and infrastructure (DCGM)

Pulls from Exercise 03's exporter.

| Panel | Query |
|---|---|
| GPU utilization heatmap | `avg by (Hostname, gpu) (DCGM_FI_DEV_GPU_UTIL{service=~"$service"})` as a heatmap |
| Framebuffer free | `min by (Hostname, gpu) (DCGM_FI_DEV_FB_FREE)` |
| GPU temperature | `max by (Hostname) (DCGM_FI_DEV_GPU_TEMP)` |
| XID error timeline | `increase(DCGM_FI_DEV_XID_ERRORS[$__rate_interval]) > 0` as state timeline |
| SM active (if DCP exposed) | `avg by (Hostname, gpu) (DCGM_FI_PROF_SM_ACTIVE)` |

Keep one panel per concern. Nested multi-metric panels are great for familiar users and useless for a cold on-call.

### 2.5 Row 4 — Cache and dependencies

| Panel | Query |
|---|---|
| Prefix cache hit rate | `service:prefix_cache_hit_rate:5m{service="$service"}` |
| Model provider error rate (for gateways) | `sum by (provider) (rate(gen_ai_client_errors_total{service="$service"}[5m]))` |
| Retry rate | `sum(rate(provider_retries_total{service="$service"}[5m]))` |
| Upstream RAG retriever p95 | per retriever |

> Note: `gen_ai_client_errors_total` is not a formal OTel semconv metric name in all SDK versions; confirm what the current semconv emits for error counts before relying on this exact series name.

### 2.6 Annotations

```json
{
  "annotations": {
    "list": [
      {"name": "Deploys", "datasource": "$datasource",
       "expr": "max_over_time(deploy_marker{service=\"$service\",env=\"$env\"}[1m])"},
      {"name": "Feature flags", "datasource": "$datasource",
       "expr": "max_over_time(feature_flag_change_marker{service=\"$service\"}[1m])"},
      {"name": "Incidents", "datasource": "pagerduty",
       "expr": "service:$service"}
    ]
  }
}
```

Deploy markers are the single highest-value annotation; emit one `deploy_marker` metric per release.

### 2.7 Links

- Bottom of the dashboard: **Links** panel pointing to runbook, trace platform (Exercise 04), DCGM alerts rules file (Exercise 03), and the service's postmortems folder.
- Each burn-rate panel data-links to the `drilldown` dashboard for that SLO.
- Each GPU panel data-links to the node dashboard with `Hostname` pre-filled.

### 2.8 Provisioning

```yaml
# grafana/provisioning/dashboards/llm-service.yaml
apiVersion: 1
providers:
  - name: 'llm-service'
    folder: 'LLM Services'
    type: file
    disableDeletion: false
    editable: true
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/llm
```

Then the dashboard JSON lives in `/var/lib/grafana/dashboards/llm/<service>.json`, versioned in git. For Helm, `grafana.sidecar.dashboards.enabled=true` is the equivalent.

### 2.9 Review ritual

Every two weeks, run an "incident readiness" review on the dashboard:

- Walk a new engineer through "an outage is happening right now — tell me what's wrong" using only this dashboard.
- Measure time-to-first-insight; anything >2 minutes is a design bug.
- Cut panels that did not get mentioned in the walkthrough.

## 3. Validation steps

1. **Cold-start test**: give the dashboard to someone unfamiliar with the service during a tabletop; measure time to correctly identify the fault lane. Target <90s.
2. **Variable sanity**: changing `$env` or `$service` reloads every panel without errors. No hardcoded series in any panel.
3. **Mobile/small-screen check**: the top two rows fit on a 1366×768 laptop without scrolling.
4. **Threshold wiring**: thresholds on the SLI panels match the SLO targets from mod-103; not vibes.
5. **Annotation delivery**: push a synthetic `deploy_marker` metric; confirm the deploy vertical line renders.
6. **Link integrity**: every external link resolves; no stale runbook URLs.
7. **No `interval=` hardcode**: every panel uses `$__rate_interval` where appropriate so the query stays sensible at different zooms.
8. **Cardinality check**: the dashboard does not trigger the "too many data points" Grafana warning on a 7d range.
9. **JSON lint**: load the exported JSON into `jsonlint` and into a Grafana staging instance; it imports clean.
10. **Change review**: dashboard JSON is in git with a human diff in each PR; no "export and overwrite".

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Information hierarchy | Row 1 = SLIs, Row 2 = burn rates, Row 3 = GPU, Row 4 = dependencies. Non-negotiable |
| Variables | `datasource`, `env`, `service`, `workload`, `model` all templated |
| Thresholds | Panels have thresholds that match SLO targets, not defaults |
| Annotations | Deploy markers present; incidents and feature flags where feasible |
| Drill-down links | Panels link to the right lower-level dashboard or trace query |
| Cold-start | Unfamiliar engineer reaches correct fault lane in <90s |
| Mobile/laptop fit | Above-the-fold answers the primary question on a laptop |
| Provisioning | Dashboard JSON lives in git; provisioning file loads it |
| Review ritual | A documented bi-weekly review exists |

## 5. Common mistakes

- **Dashboards that start with GPU graphs.** GPU panels are mesmerizing and uninformative to a cold reader. SLIs first, every time.
- **Too many panels.** If the first screen has more than ~12 panels the dashboard is already failing the cold-start test. Prune aggressively; panels not read during incidents belong on a diagnostic-only board.
- **Stacked time series with 30 series each.** Fine for exploration, actively misleading at a glance. Use topK / bottomK or aggregate first.
- **Bar/gauge dashboards.** Rare numeric values at a single point in time don't tell you whether things are getting better or worse. Prefer time series or stat panels with sparklines.
- **Fixed intervals.** `rate(x[5m])` baked into every panel will under-sample at 1h ranges and over-sample at 1m ranges. Use `$__rate_interval`.
- **No annotations.** Without deploy markers, you will spend the first 10 minutes of every incident asking "when did this ship?"
- **Dashboards edited in the UI, never committed.** Within a month, the git version and the live version diverge. Treat Grafana as read-only in prod; edits in staging, provision to prod.
- **Cross-tenant panels.** A single dashboard for all services lets everyone see everyone else's metrics; that is both a privacy and a focus problem. One dashboard per service.
- **Blanket "fix this" thresholds.** Red on TTFT panel at a vendor default like 1s ignores the actual SLO. Thresholds that aren't the SLO are noise.
- **No runbook links.** A dashboard without a link to "what do I do when this is red" is incomplete.

## 6. References

- Grafana — Best practices for creating dashboards: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/
- Grafana — Template variables: https://grafana.com/docs/grafana/latest/dashboards/variables/
- Grafana — Dashboard provisioning: https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards
- Grafana — Annotations: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/annotate-visualizations/
- Prometheus — Query basics / `$__rate_interval` semantics: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Google SRE Workbook — "Alerting on SLOs" (burn rate rationale): https://sre.google/workbook/alerting-on-slos/
- Google SRE Workbook — Monitoring (four golden signals framing): https://sre.google/workbook/monitoring/
- NIST AI Risk Management Framework (MEASURE function — continuous monitoring): https://www.nist.gov/itl/ai-risk-management-framework
