# Exercise 05 — Unified Pipeline Observability in One Pane — Reference Solution

Paired learning file: `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/exercises/exercise-05-unified-pipeline-observability-in-one-pane.md`.

> **Scope note.** The paired exercise asks the learner to design and author the single on-call dashboard that joins the four data-and-config pipeline surfaces this module covers — prompt versions (exercise 01), retrieval freshness (exercise 02), feature drift (exercise 03), and eval-set rot (exercise 04) — together with the serving SLO so an on-call can answer "which pipeline is breaking production right now?" without context-switching. The reference solution produces (a) the dashboard structure and panels, (b) the join keys that make the panels coherent, (c) the recording rules that keep queries cheap, and (d) the incident-mode variant that an on-call switches to when the pager goes off. The illustrative product is the fabricated `chatcat` RAG surface used across this module family.

## 1. Solution overview

The reliability of a model-backed service is a function of four pipelines plus the serving path. In most organisations, each pipeline has its own dashboard, its own metric names, and its own on-call rotation. The cost of that fragmentation lands on the generalist SRE at 3 a.m., when the pager fires and the question "which pipeline is the problem?" takes 20 minutes of tab-switching before triage can start. This exercise collapses the four pipelines into one coherent view, with shared join keys, consistent time windows, and an explicit incident-mode layout.

Three design invariants:

1. **One time range, one service variable, one route variable.** Every panel reads from the same `$service`, `$route`, and `$time_range`; no panel hard-codes a window. If this invariant is not held, cross-correlation is accidental at best.
2. **Panels are ordered top-to-bottom by proximity to the user.** Serving SLO first (user pain), then the pipelines that feed it (prompt, retrieval, feature, eval). An on-call who finds no serving pain at the top can leave; everything below is leading-indicator context.
3. **Incident mode is a one-click variant.** When an alert is live, the on-call switches a dashboard variable from `overview` to `incident` and the layout collapses to the four comparison plots that triage uses, with the alert's window pre-selected.

This is the "single pane of glass" discipline referenced in the Google SRE book's *Monitoring* chapter: an on-call-facing dashboard is a product with usability requirements, not a dump of every available metric.

### 1.1 Join keys

The following attributes are on every span emitted by the serving process (per the OpenTelemetry GenAI semantic conventions used in `mod-104 / exercise-01`):

- `service.name` — the service (e.g. `chatcat-serving`)
- `route` — the route (e.g. `chatcat/faq`)
- `genai.prompt.version_id` — the prompt alias resolution at request time (exercise 01)
- `genai.retrieval.index` — which index the retrieval query hit (exercise 02)
- `genai.retrieval.all_fresh` — was the answer served from fresh chunks (exercise 02)
- `chatcat.user_segment` — the request's cohort (exercise 03)
- `chatcat.eval_set_id` — which eval set last scored this route (exercise 04)

These seven attributes are the join keys that make the dashboard panels talk to each other. Any panel that cannot be sliced by `service`, `route`, and at least one of the pipeline attributes is a candidate for removal.

## 2. Implementation — worked answer

### 2.1 Recording rules

Cross-pipeline joins are too expensive to re-run per dashboard load; the recording rules below produce pre-aggregated series per `service, route` with the pipeline attributes carried along as labels. `prometheus/rules/unified-pipeline-overview.yaml`:

```yaml
groups:
  - name: chatcat-unified-pipeline-overview
    interval: 30s
    rules:
      # 1. Serving-path signals
      - record: chatcat:slo_error_rate:5m
        expr: |
          sum by (service, route) (
            rate(chatcat_requests_total{status=~"5..|429"}[5m])
          )
          /
          sum by (service, route) (
            rate(chatcat_requests_total[5m])
          )

      - record: chatcat:slo_ttft_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum by (service, route, le) (
              rate(chatcat_ttft_seconds_bucket[5m])
            )
          )

      # 2. Prompt-version churn
      - record: chatcat:prompt_version_count:1h
        expr: |
          count by (service, route) (
            count_over_time(chatcat_requests_total[1h])
          )
          # paired with: chatcat_prompt_alias_set_total, below

      - record: chatcat:prompt_alias_flips:1h
        expr: |
          sum by (service, route) (
            increase(chatcat_prompt_alias_set_total[1h])
          )

      # 3. Retrieval freshness
      - record: chatcat:retrieval_fresh_ratio:5m
        expr: |
          sum by (service, route) (
            rate(chatcat_retrieval_answers_total{fresh="true"}[5m])
          )
          /
          sum by (service, route) (
            rate(chatcat_retrieval_answers_total[5m])
          )

      - record: chatcat:retrieval_ingest_lag_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum by (service, route, le) (
              rate(chatcat_index_ingest_lag_seconds_bucket[5m])
            )
          )

      # 4. Feature drift
      - record: chatcat:feature_psi_max:1h
        expr: |
          max by (service, route, feature) (
            chatcat_feature_psi{window="1h"}
          )

      # 5. Eval-set rot
      - record: chatcat:eval_representativeness:current
        expr: |
          max by (service, route, eval_set) (
            chatcat_eval_representativeness_shortfall
          )

      - record: chatcat:eval_overdue_fraction:current
        expr: |
          max by (service, route, eval_set) (
            chatcat_eval_overdue_items / chatcat_eval_total_items
          )
```

Three design points:

- Every recorded series is labelled with `service` and `route`. A dashboard variable can therefore filter the whole pane in one place.
- Rates use a 5-minute window on the top-row serving signals (fast enough to notice a page-worthy regression) and 1 h on the leading-indicator pipelines (slow enough that the signal is not noise).
- The rules are grouped so Prometheus evaluates them in a single pass; adding a new pipeline is one block, not a schema change.

### 2.2 Dashboard JSON — overview mode

`grafana/dashboards/unified-pipeline.json`:

```json
{
  "title": "Pipelines → Serving overview",
  "uid": "chatcat-pipeline-overview",
  "templating": {
    "list": [
      { "name": "service", "type": "query", "query": "label_values(chatcat:slo_error_rate:5m, service)" },
      { "name": "route",   "type": "query", "query": "label_values(chatcat:slo_error_rate:5m{service=~\"$service\"}, route)" },
      { "name": "mode",    "type": "custom", "options": [
        { "text": "overview", "value": "overview", "selected": true },
        { "text": "incident", "value": "incident" }
      ]}
    ]
  },
  "panels": [
    {
      "title": "Serving — SLO health",
      "type": "row",
      "panels": [
        { "title": "Error rate",   "targets": [{ "expr": "chatcat:slo_error_rate:5m{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "TTFT p95",     "targets": [{ "expr": "chatcat:slo_ttft_p95:5m{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Burn rate 1h", "targets": [{ "expr": "chatcat:slo_budget_burn_rate:1h{service=~\"$service\",route=~\"$route\"}" }] }
      ]
    },
    {
      "title": "Prompt pipeline (exercise 01)",
      "type": "row",
      "panels": [
        { "title": "Active version",     "type": "stat",     "targets": [{ "expr": "chatcat_prompt_alias_current{service=~\"$service\",route=~\"$route\",environment=\"prod\"}" }] },
        { "title": "Alias flips (1h)",   "type": "timeseries","targets": [{ "expr": "chatcat:prompt_alias_flips:1h{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Rollback count (24h)","type": "stat",    "targets": [{ "expr": "sum by (service, route) (increase(chatcat_prompt_alias_set_total{reason=\"rollback\",service=~\"$service\",route=~\"$route\"}[24h]))" }] }
      ]
    },
    {
      "title": "Retrieval pipeline (exercise 02)",
      "type": "row",
      "panels": [
        { "title": "Fresh ratio 5m",     "targets": [{ "expr": "chatcat:retrieval_fresh_ratio:5m{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Ingest lag p95",     "targets": [{ "expr": "chatcat:retrieval_ingest_lag_p95:5m{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Missing-timestamp rate","targets": [{ "expr": "sum by (service, route) (rate(chatcat_index_missing_timestamp_total{service=~\"$service\",route=~\"$route\"}[5m]))" }] }
      ]
    },
    {
      "title": "Feature pipeline (exercise 03)",
      "type": "row",
      "panels": [
        { "title": "Feature PSI (max)", "targets": [{ "expr": "chatcat:feature_psi_max:1h{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Drift incidents (14d)", "type": "stat",
          "targets": [{ "expr": "sum by (service, route) (increase(chatcat_drift_incidents_total{service=~\"$service\",route=~\"$route\"}[14d]))" }] }
      ]
    },
    {
      "title": "Eval pipeline (exercise 04)",
      "type": "row",
      "panels": [
        { "title": "Representativeness shortfall", "targets": [{ "expr": "chatcat:eval_representativeness:current{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Overdue fraction", "targets": [{ "expr": "chatcat:eval_overdue_fraction:current{service=~\"$service\",route=~\"$route\"}" }] },
        { "title": "Gate verdicts (7d)", "type": "timeseries",
          "targets": [{ "expr": "sum by (service, route, verdict) (increase(chatcat_release_gate_verdicts_total{service=~\"$service\",route=~\"$route\"}[7d]))" }] }
      ]
    }
  ]
}
```

Rows read top-to-bottom by proximity to the user. A serving-side page lands on the first row; the on-call then scans downwards to find the pipeline whose numbers moved on the same timeline.

### 2.3 Dashboard JSON — incident mode

The `mode=incident` layout is a smaller set of panels tuned for triage: all plots share the alert's time window, and each panel overlays the four pipelines on the same y-axis-normalised range so a visual correlation is the first thing the eye catches.

```json
{
  "panels": [
    {
      "title": "Incident timeline",
      "type": "timeseries",
      "overrides": [
        { "matcher": { "id": "byRegexp", "options": "pipelines/.*" },
          "properties": [{ "id": "custom.axisPlacement", "value": "right" }] }
      ],
      "targets": [
        { "legendFormat": "serving/error",  "expr": "chatcat:slo_error_rate:5m{service=~\"$service\",route=~\"$route\"}" },
        { "legendFormat": "pipelines/prompt-flips","expr": "chatcat:prompt_alias_flips:1h{service=~\"$service\",route=~\"$route\"}" },
        { "legendFormat": "pipelines/fresh-ratio", "expr": "1 - chatcat:retrieval_fresh_ratio:5m{service=~\"$service\",route=~\"$route\"}" },
        { "legendFormat": "pipelines/psi-max", "expr": "chatcat:feature_psi_max:1h{service=~\"$service\",route=~\"$route\"}" },
        { "legendFormat": "pipelines/eval-rot","expr": "max by (service, route) (chatcat:eval_representativeness:current{service=~\"$service\",route=~\"$route\"})" }
      ]
    },
    {
      "title": "Change log",
      "type": "annotations-list",
      "targets": [
        { "expr": "chatcat_prompt_alias_set_total{service=~\"$service\",route=~\"$route\"}" },
        { "expr": "chatcat_eval_set_promotions_total{service=~\"$service\",route=~\"$route\"}" },
        { "expr": "chatcat_release_ledger_events{service=~\"$service\",route=~\"$route\"}" }
      ]
    }
  ]
}
```

Two important choices:

- **Normalise everything to a `0-1`-ish range.** The incident-mode plot overlays an error rate (fraction), a flip count (small integer), a fresh-ratio complement (fraction), a PSI (float), and a shortfall (fraction) on the same chart. Each gets a per-series axis scaling so a human eye can spot the one that moved on the alert timeline; absolute numbers go into the drilldown row underneath.
- **A change-log annotation stream.** The second panel pulls annotations from the release ledger (every forward promote, every rollback, every eval promotion) so the incident-mode plot has vertical markers for every change. That is often the whole triage: find the alert timestamp, find the nearest change marker.

### 2.4 Serving-path attribute hygiene

For the whole dashboard to work, every serving-path span must carry the join keys. The reference serving-side middleware adds them at the top of the handler:

```python
# chatcat/serving/middleware.py
from opentelemetry import trace


def enrich_span(span, request, resolved_prompt, retrieval_result, user_segment, eval_set_id):
    span.set_attribute("service.name", "chatcat-serving")
    span.set_attribute("route", request.route)
    span.set_attribute("genai.prompt.version_id", resolved_prompt.version_id)
    if retrieval_result is not None:
        span.set_attribute("genai.retrieval.index", retrieval_result.index)
        span.set_attribute("genai.retrieval.all_fresh", retrieval_result.all_fresh)
    if user_segment is not None:
        span.set_attribute("chatcat.user_segment", user_segment)
    if eval_set_id is not None:
        span.set_attribute("chatcat.eval_set_id", eval_set_id)
```

This is the minimum attribute set. Teams that want finer joins add `genai.model` and `genai.model.version` (both from the OpenTelemetry GenAI semantic conventions); the dashboard templating already allows filtering by them when the attributes are present.

### 2.5 Runbook-adjacent linkouts

Every pipeline row has a hidden "runbook" panel with a plain-text link: on hover it exposes the paired runbook URL. The reference set:

- Prompt row → `rb-prompt-rollback` (exercise 01)
- Retrieval row → `rb-retrieval-freshness-burn` (exercise 02)
- Feature row → `rb-drift-incident` (exercise 03)
- Eval row → `rb-eval-refresh` (exercise 04)

The point is that the dashboard is the top of the triage funnel; the runbooks are where the triage ends. Linking them from the pane is how the on-call avoids "which runbook does this belong to?" friction.

## 3. Validation steps

1. **Recording rules compile.** `promtool check rules prometheus/rules/unified-pipeline-overview.yaml`.
2. **Dashboard loads with variables.** Import the JSON into a staging Grafana, confirm the `service` and `route` variables resolve against the recording rules, and confirm every panel renders without `N/A`.
3. **Cross-join sanity.** Inject a prompt alias flip in staging and confirm the "Alias flips" panel increments and the "Serving — SLO health" panel's time axis aligns with the increment.
4. **Incident-mode variant.** Flip `mode=incident`; the layout should collapse to the normalised overlay and the annotations list. The overlay's five legend lines must all be visible and labelled.
5. **Attribute coverage audit.** Run `scripts/audit-attrs.sh --service chatcat-serving --days 1` — the script samples 1 % of recent spans and asserts that each of the seven join-key attributes is present on more than 99 % of serving-path spans. Attribute drift is a failure.
6. **Load test.** Open the dashboard with `$service=.*` and `$route=.*` and a 7-day range; the slowest panel must render in under 5 s. If not, add a recording rule for the offending query; it is the single most common performance fix.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Single shared time range and variables *(hard gate)* | Panels hard-code ranges | Some share | All panels read `$service`, `$route`, `$time_range` |
| 2 | Top-to-bottom ordering by user proximity *(hard gate)* | Arbitrary order | Some grouping | Serving first, then prompt, retrieval, feature, eval |
| 3 | Join keys on every serving-path span *(hard gate)* | Missing | Partial | All seven attributes present on >99 % of spans with an audit script |
| 4 | Recording rules used *(hard gate)* | Raw queries only | A few recording rules | Pre-aggregated series for every panel that joins pipelines |
| 5 | Incident-mode variant | Missing | Single extra panel | Mode toggle collapses to overlay + change-log annotations |
| 6 | Change-log annotation stream | Missing | Present but sparse | Covers prompt flips, eval promotions, release-ledger events |
| 7 | Attribute coverage audited | Missing | Named but not run | Scheduled audit with thresholds and failure signal |
| 8 | Runbook linkouts | Missing | Named but not clickable | Every pipeline row links to its paired runbook |
| 9 | Performance sanity | > 15 s to load | Some panels slow | All panels under 5 s with wide filters on 7 d range |
| 10 | Integration with upstream modules | Isolated | Names one | Pulls signals from exercises 01–04 and the SLO stack from mod-103 |

## 5. Common mistakes

- **One dashboard per pipeline, no cross-view.** This is the default most organisations drift into. It forces context-switching at the exact moment an on-call needs the opposite — correlation across pipelines on a shared timeline.
- **Different time windows on different panels.** A 1-minute window on serving next to a 24-hour window on feature drift makes visual correlation impossible. One `$time_range` variable binds every panel.
- **No join key on serving spans.** If the SLO panel cannot be sliced by prompt version, retrieval index, or eval set, the dashboard is five dashboards drawn on one page.
- **Overlaying absolute values.** An error rate (0 – 1) and a PSI (0 – 3) on the same y-axis makes one of them invisible. Normalise per-series or use a secondary axis.
- **Missing the change-log annotations.** Half of all incidents are explained by "something changed". Without annotations for every change the pipelines produce, the on-call is guessing.
- **Treating the dashboard as read-only.** Every pipeline row should link to its runbook. The dashboard is the top of the triage funnel; losing the handoff undoes the benefit of the join.
- **Letting query performance rot.** A dashboard that is slow to open is a dashboard the on-call tabs away from. Recording rules are the fix; add them proactively.
- **Dropping a panel because "nobody opens it".** Leading-indicator pipelines look quiet until they do not. Keep them in view; that is the point.

## 6. References

- Google SRE Book, *Monitoring Distributed Systems* — [sre.google/sre-book/monitoring-distributed-systems/](https://sre.google/sre-book/monitoring-distributed-systems/) — on the "four golden signals" and on dashboards as on-call products.
- Google SRE Workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — for the burn-rate panels shown in the serving row.
- OpenTelemetry Semantic Conventions for Generative AI — [opentelemetry.io/docs/specs/semconv/gen-ai/](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — source of the GenAI span attributes used as join keys.
- Prometheus recording-rule guidance — [prometheus.io/docs/practices/rules/](https://prometheus.io/docs/practices/rules/) — informs the split between recording and alerting rules.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `MEASURE` function names the need for unified visibility across the AI system's components.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06 AI supply chain` and `ML09 output integrity attack` both require the correlated visibility this dashboard provides.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — adversarial techniques against ML systems frequently manifest across multiple pipelines; a unified pane is a precondition for noticing that pattern.
- Prior-module references: `mod-104 / exercise-01` (OTel GenAI attributes), `mod-104 / exercise-07` (Grafana dashboard authoring for incident readiness), `mod-103` (SLO stack), and exercises 01–04 of this module.
