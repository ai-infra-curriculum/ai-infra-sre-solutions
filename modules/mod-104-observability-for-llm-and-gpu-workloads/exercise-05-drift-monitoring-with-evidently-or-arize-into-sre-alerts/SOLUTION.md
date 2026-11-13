# Exercise 05 — Drift Monitoring with Evidently or Arize into SRE Alerts (Solution)

## 1. Solution overview

Run model/data drift checks on production traffic with **Evidently AI** (OSS) or **Arize** (managed), export the drift signals as Prometheus-shaped metrics, and wire a small, boring set of Alertmanager rules so drift becomes an on-call-visible signal — not just a weekly dashboard the team forgets to open.

Design invariants:

1. **Drift is not an SLO; it's a *leading indicator* for quality or revenue SLOs.** Alerts should ticket, not page, unless drift is paired with a downstream SLO breach.
2. **Report per-feature and per-prediction**, not a single scalar. One drifted feature is a different incident from "the whole input distribution shifted".
3. **Pair every drift alert with a reference dataset** that is versioned. Drift without a reference is a vibe check.
4. **The runbook is as important as the metric.** A drift alert that arrives without "here's who owns the labels / here's the retrain branch" is noise.

## 2. Worked implementation

### 2.1 Reference window and comparison window

- **Reference**: a frozen window of production traffic known to be "good" (last stable release's first full week, or a curated holdout). Store in object storage; version with the model deploy that produced it.
- **Current**: a rolling window matched in size and shape. 24h is a reasonable default for a steady stream; smaller windows for low-volume services.

Both windows must have matching schemas. Rename/added columns are schema drift, which is a stronger signal than distribution drift and should be alerted separately.

### 2.2 Metric selection

| Signal | Statistic | When to use |
|---|---|---|
| Numeric feature drift | Wasserstein distance, or KS test p-value | Continuous features; KS is simple, Wasserstein is interpretable |
| Categorical feature drift | PSI (Population Stability Index) or JS divergence | Low-cardinality categoricals |
| Prediction drift | PSI on predicted class / regression histogram | Model output changes before features do |
| Embedding drift | Max mean discrepancy (MMD) or cosine-distance distributions | RAG / semantic search contexts |
| Concept drift | Lagged accuracy / error rate against labels when they arrive | Only possible once labels catch up |

Evidently exposes Wasserstein, KS, PSI, JS, chi² and more via its metric catalog. Arize offers similar statistics in its monitors.

> Note: Evidently's default statistic per feature type has changed across versions; check the active version's "data drift" preset docs before relying on a specific default.

### 2.3 Evidently → Prometheus (OSS path)

The Evidently pattern: compute a Report / TestSuite periodically, then export selected metrics to Prometheus via a small bridge. Example sketch (simplified):

```python
# drift_job.py — scheduled every 15 minutes
import pandas as pd
from evidently import Report
from evidently.presets import DataDriftPreset
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

reference = pd.read_parquet("s3://models/my-model/v12/reference.parquet")
current = load_recent_production_rows(window="15min")

report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=reference, current_data=current)
result = report.as_dict()

registry = CollectorRegistry()
share_g = Gauge("model_drift_feature_share", "Share of features with drift",
                ["model", "version"], registry=registry)
count_g = Gauge("model_drift_feature_count", "Count of features marked drifted",
                ["model", "version"], registry=registry)

share_g.labels("my-model", "v12").set(
    result["metrics"][0]["result"]["share_of_drifted_columns"]
)
count_g.labels("my-model", "v12").set(
    result["metrics"][0]["result"]["number_of_drifted_columns"]
)

per_feat_g = Gauge("model_drift_feature_score", "Per-feature drift score",
                   ["model", "version", "feature", "statistic"], registry=registry)
for feat, info in result["metrics"][0]["result"]["drift_by_columns"].items():
    per_feat_g.labels("my-model", "v12", feat, info["stattest_name"]).set(info["drift_score"])

push_to_gateway("pushgateway:9091", job="drift-my-model", registry=registry)
```

> Note: Evidently's Report JSON schema differs across 0.4.x and 0.5.x. Field names here (`drift_by_columns`, `stattest_name`) may need updating against the pinned version; verify before shipping.

The same pattern works for prediction drift (swap the preset) and target drift (needs labels).

### 2.4 Arize → Prometheus (managed path)

Arize monitors fire webhook/PagerDuty events directly, but to keep alerting in one Alertmanager, export via Arize's API on a schedule:

```python
# arize_sync.py — pull monitor state, publish to Prometheus
from arize.monitors import get_monitors  # pseudocode; actual client is the Arize SDK
monitors = get_monitors(model_id="my-model")
for m in monitors:
    monitor_state_g.labels(
        model="my-model",
        monitor=m.name,
        kind=m.kind,
    ).set(1 if m.state == "triggered" else 0)
```

> Note: the exact Arize SDK method signatures and the recommended pull-vs-webhook flow depend on the active product tier; confirm against Arize docs before writing production code.

Webhook-direct is simpler if you already use PagerDuty; Prometheus mirroring is worth it when drift alerts need to compose with Prometheus-side burn-rate and GPU alerts in the same Alertmanager routing tree.

### 2.5 Alert rules

```yaml
# prometheus/rules/drift.yml
groups:
  - name: ai.drift
    interval: 1m
    rules:
      # 1. Fraction of features drifted — leading, ticket severity.
      - alert: ModelFeatureDriftRising
        expr: model_drift_feature_share > 0.3
        for: 30m
        labels:
          severity: ticket
          team: model-platform
        annotations:
          summary: "{{ $labels.model }} v{{ $labels.version }}: >30% of features drifting for 30m"
          runbook: "runbooks/model-drift.md"

      # 2. Prediction drift — model output shape changed; stronger signal.
      - alert: ModelPredictionDrift
        expr: model_drift_prediction_score > 0.2
        for: 15m
        labels:
          severity: ticket
          team: model-platform
        annotations:
          summary: "Prediction distribution drift on {{ $labels.model }} (score {{ $value }})"

      # 3. Composite with an SLO signal: drift AND the quality SLO is burning.
      - alert: DriftWithQualitySLOBurn
        expr: |
          (model_drift_feature_share > 0.2)
          and on(model)
          (model_quality_slo_fast_burn > 2)
        for: 10m
        labels:
          severity: page
          team: model-platform
        annotations:
          summary: "Drift + quality burn on {{ $labels.model }} — investigate release or inputs"

      # 4. Schema drift — stronger than distribution drift, always ticket.
      - alert: ModelSchemaDrift
        expr: increase(model_drift_schema_changes_total[1h]) > 0
        for: 0m
        labels:
          severity: ticket
        annotations:
          summary: "Schema change detected on {{ $labels.model }} — upstream data contract broke"

      # 5. Drift job dead — silent drift is the worst kind.
      - alert: DriftJobStale
        expr: time() - max by (model) (model_drift_last_run_timestamp) > 3600
        for: 10m
        labels:
          severity: ticket
        annotations:
          summary: "Drift job for {{ $labels.model }} has not reported in 1h"
```

Expose `model_drift_last_run_timestamp` from the drift job itself so the "silent stall" case is covered.

### 2.6 Routing

Route drift alerts to a model-platform or ML-eng rotation, not the infra SRE rotation, unless the composite `DriftWithQualitySLOBurn` fires — then page both. Keep this split in the Alertmanager `route` tree, not in per-rule labels, to avoid drift (pun intentional) across alert files.

### 2.7 How drift maps to NIST AI RMF and OWASP ML Top 10

- NIST AI RMF **MEASURE 2.4** calls out ongoing monitoring of trustworthiness characteristics including drift; a drift alert pipeline is one concrete control.
- NIST AI RMF **MANAGE 4.1** calls for post-deployment monitoring processes; routing drift to on-call is how that happens in practice.
- OWASP Machine Learning Security Top 10 lists risks (e.g. input manipulation, data poisoning) that often appear first as distribution drift on input features.
- MITRE ATLAS techniques related to poisoning and evasion can present as drift on the input or embedding distribution; drift alerts are a *detection*, not a classifier of cause.

## 3. Validation steps

1. **Reference parity**: `len(reference) > 1000` (rough floor for KS stability) and schemas match current. Fail the job loudly otherwise.
2. **Known-good day**: run on a known stable day; feature-drift share should sit well below the alert threshold.
3. **Synthetic drift injection**: shift one numeric feature's mean by 2σ for a window; confirm the per-feature metric spikes and the ticket alert fires after the `for:` window.
4. **Schema-drift drill**: drop a column in the synthetic current window; schema-drift counter increments and alert fires.
5. **Silent-stall drill**: scale the drift job to zero replicas for an hour; `DriftJobStale` fires.
6. **Composite drill**: in staging, breach the quality SLO burn AND drift simultaneously; confirm `DriftWithQualitySLOBurn` pages.
7. **Noise audit**: after a week in production, review how many drift alerts fired and how many were acted on. Target <3 drift alerts/week per model; more means thresholds are wrong or the reference window is stale.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Reference/current pair | Versioned reference; current window matches shape; schema parity enforced |
| Statistic choice | Per feature-type statistic justified (continuous vs categorical vs embedding) |
| Prediction drift | Monitored independently of feature drift |
| Metric export | Prometheus-shaped (or equivalent) so alerts compose with existing SRE alerts |
| Alert set | Minimum: feature-share, prediction, schema, composite-with-SLO, job-stale |
| Severity logic | Pure drift is ticket; drift + SLO burn is a page |
| Routing | Drift alerts route to model-platform/ML-eng; composite pages both |
| Framework mapping | Explicit mapping to NIST AI RMF function(s) and relevant OWASP ML risk |
| Noise budget | A target rate for acceptable drift-alert volume per week |

## 5. Common mistakes

- **Alerting on raw p-values from KS tests.** With large N, almost everything is "significantly different". Threshold on effect size (Wasserstein, PSI) instead, or require both p-value < α AND effect size > ε.
- **No frozen reference.** Comparing "last hour vs last day" catches big shifts but misses slow drift; you also can't correlate an alert to a known baseline across weeks.
- **Monolithic drift score.** A single "is the model drifting?" boolean destroys the diagnostic value. Keep per-feature and per-prediction signals.
- **Paging on drift alone.** Drift does not necessarily mean user-visible harm; pairing with the quality SLO is the right trigger for a page.
- **Forgetting embedding drift for RAG systems.** The retrieval corpus shifts independently from queries; both sides need drift monitoring.
- **No silent-stall alert.** A dead drift job means the dashboard reads "all clear" while the model is drifting. `model_drift_last_run_timestamp` is as important as the drift metric itself.
- **Letting the reference age.** Reference must be replaced on model retrain; otherwise every retrain shows up as a fake alert.
- **Confounding concept drift with data drift.** Inputs can stay stable while labels change (concept drift); only label-based evaluation catches this. Build a label-lagged evaluation channel if labels are available.

## 6. References

- Evidently AI documentation: https://docs.evidentlyai.com/
- Evidently data-drift presets and statistics: https://docs.evidentlyai.com/reference/all-metrics
- Arize documentation (monitors, drift): https://docs.arize.com/
- Prometheus Pushgateway (for batch job metrics): https://prometheus.io/docs/instrumenting/pushing/
- NIST AI Risk Management Framework 1.0: https://www.nist.gov/itl/ai-risk-management-framework
- NIST AI RMF Playbook: https://airc.nist.gov/AI_RMF_Knowledge_Base/Playbook
- OWASP Machine Learning Security Top 10: https://owasp.org/www-project-machine-learning-security-top-10/
- MITRE ATLAS: https://atlas.mitre.org/
