# Exercise 03 — Feature Drift as a Serving Incident, with ML-Platform Handoff — Reference Solution

Paired learning file: `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/exercises/exercise-03-feature-drift-as-serving-incident-with-ml-platform-handoff.md`.

> **Scope note.** The paired exercise asks the learner to (a) treat a feature-distribution shift as a first-class serving incident, not a data-science weekly report, (b) wire detection into the on-call alerting path, and (c) define the handoff contract between SRE (who owns serving) and the ML-platform or data team (who owns features and retraining). The reference solution uses the illustrative `chatcat` RAG surface, which is also a first-class user of a classic feature store for user-cohort features used in prompt routing.

## 1. Solution overview

Feature drift is the condition where the statistical distribution of a feature at serving time has shifted away from the distribution the model was trained or last evaluated on. It is a reliability event because model output quality degrades without any code, image, or prompt change — i.e. none of the obvious "what shipped?" signals a classical SRE investigation reaches for. This exercise promotes drift to a first-class incident class with its own detection, paging, triage, and handoff.

Three design invariants:

1. **Drift is paged only when it correlates with a user-facing SLO or eval signal.** Pure distribution shift without user impact is a ticket, not a page. This keeps the signal credible and prevents on-call burnout from statistical noise.
2. **Serving is not where drift is fixed.** SRE's job on a drift incident is to contain impact (route-around, fall back, shed load from the affected cohort) and to hand off to the owners of the feature pipeline and model. The ML-platform team is the fixer.
3. **The handoff contract is a template, not a paragraph in a runbook.** It is a structured form carrying every piece of information the receiving team needs to start work without re-running triage.

This is the control shape named in the NIST AI Risk Management Framework `MANAGE 2.1` ("Measurement activities are planned, documented, reviewed, and communicated across relevant AI actors") — detection, verification, communication, and transfer of responsibility are explicit and auditable.

### 1.1 What "feature" means here

For `chatcat`, the illustrative features are:

- **User-cohort feature** (`user_segment`): a categorical label (`enterprise`, `smb`, `trial`, `internal`) attached to each request by a feature-store lookup; used by the serving process to pick a prompt-template variant.
- **Query-shape features**: derived at serving time (query token count, retrieval k, max retrieved-chunk age in seconds); used both for routing and for post-hoc analysis.
- **Request-time embeddings**: the query embedding, as a vector; used as the retrieval key.

All three can drift. The handoff differs:

- A shift in `user_segment` is usually a change in the upstream user-identity service — the fix lives with the identity team and the ML-platform team only needs to refresh the training data if the segmentation definition changed.
- A shift in query-shape features is usually a product change (a new UI shipped longer queries, or `k` was increased by a release of the serving process) — the fix lives with the product and serving teams.
- A shift in request-time embeddings is usually a change in the embedding model version or a shift in user intent — the fix lives with the ML-platform team.

The handoff template captures which lane and routes accordingly.

## 2. Implementation — worked answer

### 2.1 Detection pipeline

For categorical and bucketable numeric features, the detector computes a *Population Stability Index* (PSI) between a reference window (last stable release's first full week, versioned alongside the model) and a rolling current window (1 h and 24 h). PSI is a standard, simple statistic and is interpretable: < 0.1 is "stable", 0.1 – 0.25 is "moderate", > 0.25 is "significant".

For embeddings, the detector computes the cosine distance between the mean of the reference window and the mean of the current window, and the Maximum Mean Discrepancy (MMD) statistic on a subsample.

Detection runs as a scheduled job every 15 minutes and writes one Prometheus series per feature:

```python
# drift/job.py — scheduled every 15 minutes
from __future__ import annotations

import math
from collections import Counter
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway


def psi(reference: Counter, current: Counter) -> float:
    categories = set(reference) | set(current)
    ref_total = sum(reference.values()) or 1
    cur_total = sum(current.values()) or 1
    eps = 1e-6
    total = 0.0
    for c in categories:
        r = reference[c] / ref_total
        v = current[c] / cur_total
        r = max(r, eps)
        v = max(v, eps)
        total += (v - r) * math.log(v / r)
    return total


def main() -> None:
    reference = load_reference_counts()  # e.g. Counter({'enterprise': 1200, 'smb': 7800, ...})
    current = load_current_counts(window="1h")

    registry = CollectorRegistry()
    psi_g = Gauge(
        "chatcat_feature_psi",
        "Population Stability Index per feature.",
        ["feature", "window", "reference_version"],
        registry=registry,
    )
    for feature, (ref_counts, cur_counts) in reference.items():
        score = psi(ref_counts, current[feature])
        psi_g.labels(feature=feature, window="1h", reference_version=reference.version).set(score)

    push_to_gateway("pushgateway.observability:9091", job="chatcat-drift", registry=registry)
```

The reference window is versioned: when a new model ships, the first full stable week post-ship replaces the reference set, and a row in a `drift_reference` table records `(model_version, reference_version, window_start, window_end)`. Changing the reference without a corresponding model change is forbidden — otherwise drift disappears by rebaselining.

### 2.2 Alerting policy

```yaml
groups:
  - name: chatcat-drift
    interval: 1m
    rules:
      - alert: ChatcatFeaturePSIElevated
        expr: chatcat_feature_psi{window="1h"} > 0.25
        for: 30m
        labels:
          severity: ticket
          incident_class: drift
        annotations:
          summary: "feature PSI elevated: {{ $labels.feature }} (score {{ $value }})"

      - alert: ChatcatFeaturePSICorrelatedWithSLO
        expr: |
          (chatcat_feature_psi{window="1h"} > 0.25)
          and on(route)
          (chatcat:slo_budget_burn_rate:1h > 2)
        for: 10m
        labels:
          severity: page
          incident_class: drift
        annotations:
          summary: "feature drift AND SLO burn ({{ $labels.feature }}, {{ $labels.route }})"
          runbook: https://runbooks.example/rb-drift-incident
```

The alert that pages is not "PSI is high" — it is "PSI is high *and* the SLO budget is burning on the same route". That is the correlation that makes drift an incident. Standalone PSI elevation gets a ticket.

A twin alert also fires when an eval-score signal is regressing in the same route-aligned way, for routes where the eval cadence is high enough to be a real-time signal. This is the control that catches drift which hurts quality before it hurts the serving-path SLO (a RAG answer can be wrong without being slow).

### 2.3 Containment primitives

The SRE on-call has three contain-impact operations before the handoff:

1. **`servingctl set-route-variant --route <r> --variant <v>`** — switches a cohort to an alternate prompt variant known to be more robust to the drifted input (e.g. a more defensive system prompt that asks for clarification more aggressively).
2. **`servingctl shed-cohort --cohort <c> --target-fallback`** — routes traffic from a specific user cohort to a static or baseline-model fallback. Aggressive; used only when the alternate variant is not enough.
3. **`servingctl rebaseline-reference --feature <f> --freeze`** — explicitly *freezes* the reference window so the PSI reading stops changing while triage is underway. Never used as a fix; used only to prevent the signal from decaying during a long incident.

All three are audit-logged and emit a release-ledger row with `strategy=drift-containment` so the DORA ledger reads them.

### 2.4 Handoff template

`templates/drift-handoff.yaml`:

```yaml
incident_id: <required>
detected_at: <ISO-8601 UTC>
detected_by: <alert id or human handle>
service: chatcat
route: <required>
feature:
  name: <required>
  kind: categorical | numeric | embedding
  owner_team: <required>                 # which team owns the upstream definition
drift_evidence:
  statistic: psi | mmd | cosine
  score: <float>
  reference_version: <required>
  current_window: <ISO-8601 range>
  per_bucket_delta:                      # for categorical / numeric
    - bucket: <value>
      ref_share: <float>
      cur_share: <float>
user_impact:
  slo_burn_rate_1h: <float>              # 0.0 if no burn
  eval_delta:                            # relative to baseline
    groundedness: <float>
    refusal_rate: <float>
    structured_output_validity: <float>
  user_reports: <int>
containment_actions:
  - action: <variant-switch | cohort-shed | reference-freeze>
    at: <ISO-8601 UTC>
    operator: <human>
    note: <string>
handoff_lane: identity | serving-and-product | ml-platform
asks_of_receiving_team:
  - <free text; e.g. "verify upstream segment mapping">
  - <free text; e.g. "confirm whether reference-set rebuild is needed">
```

Every field marked `<required>` must be present before the handoff Slack-bot accepts the transfer. The bot writes the completed form into an issue in the receiving team's tracker, attaches the incident id, and posts a link back to the incident channel.

### 2.5 Receiving-team contract

The ML-platform or data team acknowledges within an SLA (typically 1 h during business hours for `handoff_lane=ml-platform`, 30 min for `handoff_lane=identity` because identity shifts unwind quickly). The acknowledgement includes:

- confirmation of lane (sometimes the first-pass lane is wrong),
- a preliminary ETA for a fix or further analysis,
- a pointer to the reference-rebuild plan if rebaselining is on the table.

If the acknowledgement does not arrive within SLA, the incident escalates to the receiving team's manager — this is the `MANAGE 2.1` "communication across relevant AI actors" control with teeth.

### 2.6 Runbook extract — `rb-drift-incident`

```markdown
# rb-drift-incident

## When
- `ChatcatFeaturePSICorrelatedWithSLO` or `ChatcatFeaturePSICorrelatedWithEval` alert pages.

## Triage (first 10 minutes)
1. Confirm the correlation is not coincidence:
   - Open the `Feature drift` dashboard.
   - Compare the drift start timestamp to the SLO burn start timestamp. Correlation within 2 h is the triggering condition.
2. Identify the feature and inspect per-bucket deltas. Save the screenshot into the incident channel.
3. Decide the handoff lane:
   - `user_segment` shifted → identity lane.
   - Query-shape feature shifted *after* a product or serving release → serving-and-product lane.
   - Embeddings or label distribution shifted with no recent product change → ml-platform lane.

## Containment (next 10 minutes)
1. If an alternate prompt variant exists for the affected route, apply it:
   `servingctl set-route-variant --route <r> --variant defensive`.
2. If the drift is concentrated in one cohort, shed that cohort to fallback:
   `servingctl shed-cohort --cohort <c> --target-fallback`.
3. Do NOT rebaseline the reference window; that is a rebaselining decision, not a containment decision.

## Handoff
1. Fill `templates/drift-handoff.yaml` end-to-end. The bot refuses incomplete forms.
2. Run `incidentctl handoff drift --form drift-handoff.yaml`.
3. Expect acknowledgement within the lane's SLA; escalate on breach.

## Post-incident
- The receiving team closes the incident when the drift is either resolved or declared expected.
- The incident record keeps both teams' timelines for the DORA `Time to Restore Service` metric (mod-111 / exercise-06).
```

## 3. Validation steps

1. **PSI arithmetic.** `python -m pytest drift/tests/test_psi.py -q` — fixtures include `[0.5/0.5 vs 0.5/0.5] → 0`, `[0.9/0.1 vs 0.1/0.9] → ~1.76`, and mixed-cardinality edge cases.
2. **Alert correlation.** `promtool test rules prometheus/rules/drift.yaml prometheus/tests/drift-tests.yaml` — drives a timeline where PSI rises alone (ticket), then where PSI and SLO burn rise together (page).
3. **Handoff template validator.** `drift-handoff-validator templates/drift-handoff.yaml` — refuses missing required fields, refuses an unknown `handoff_lane`, refuses an `eval_delta` with keys that are not in the known-good set.
4. **Game-day.** Inject a synthetic `user_segment` shift in a staging environment (flip 30 % of traffic to `enterprise`), confirm the alert fires, walk the on-call through containment and handoff; the identity team acknowledges within SLA.
5. **Audit of reference versions.** `drift-reference-report --days 180` — lists every rebaseline event with the model version it was paired with. A rebaseline not paired with a model change is a fail.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Detection statistic appropriate to feature type *(hard gate)* | Single statistic across all features | Right stat per type but ad hoc thresholds | PSI / MMD / cosine with per-type thresholds and version-anchored reference |
| 2 | Alert policy: pure drift is a ticket *(hard gate)* | Every elevation pages | Pure drift pages | Pure drift tickets; drift-plus-SLO or drift-plus-eval pages |
| 3 | Reference window is versioned *(hard gate)* | Floats over time | Versioned but not model-anchored | Versioned and paired with model version; rebaselining is audited |
| 4 | Containment primitives *(hard gate)* | None | One | Variant switch, cohort shed, reference freeze — all audit-logged |
| 5 | Handoff template present *(hard gate)* | Free-text paragraph | Template with optional fields | Structured template with required fields enforced by bot |
| 6 | Receiving-team SLA | Missing | Named but no escalation | Named with lane-specific SLA and escalation path |
| 7 | Integration with SLO and eval signals | Isolated | Uses one | Uses SLO burn rate and eval-score deltas as co-firing conditions |
| 8 | Release-ledger emission | Missing | Containment logged only | Detection, containment, and handoff all logged |
| 9 | Game-day evidence | None | Dry-run with no injection | Injection-driven game-day with lane switches exercised |
| 10 | Integration with upstream modules | Isolated | Names one | Names `mod-104 / exercise-05` drift exporters, `mod-103` SLO stack, `mod-108` runbook library |

## 5. Common mistakes

- **Paging on every PSI blip.** PSI is noisy at short windows and on low-volume routes. Paging on the standalone signal destroys the credibility of the on-call rotation. Correlate with user-facing signals.
- **Rebaselining to make the alert go away.** A rebaseline without a model change is a lie: it hides the drift the model has not been retrained against. Pair rebaselines with model changes and require the pairing in the reference table.
- **Treating drift as an ML-platform problem only.** Many drifts are upstream-of-ML — identity, product, UI, serving config. The handoff template's `handoff_lane` field exists so the first-pass triage puts the incident on the right team, not the most common team.
- **A free-text handoff.** An incident is a transfer of responsibility between two teams. A paragraph loses detail; a form does not. The structured form is the contract.
- **Forgetting the embedding case.** PSI is not meaningful for embeddings; MMD and cosine-distance comparisons are the right tools, and the detector has to call them separately.
- **No post-incident reference-review.** When the drift is resolved, somebody has to decide if the reference set should move. Rolling that into the incident close is what prevents drift fatigue three months later.
- **Running the detector off cached serving logs with incomplete fields.** If the detector cannot see the same features the serving process routes on, its statistics are a plausible number for a different population than production.

## 6. References

- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `MANAGE 2.1` on measurement planning and communication across AI actors, and `MEASURE 2.6` on performance of AI systems in deployment conditions.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML02 data poisoning` and `ML07 transfer learning attack` both manifest as drift relative to the training distribution; detection is a necessary control.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the `AML.T0019 — Publish Poisoned Datasets` and `AML.T0043 — Craft Adversarial Data` technique classes produce distribution shifts; drift detection is one of the intended controls.
- Google SRE Workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — provides the SLO-burn arithmetic used in the correlated alert.
- Evidently AI, *Data drift concepts* — the OSS reference for drift statistics (PSI, KS, Wasserstein, JS, MMD). The `mod-104 / exercise-05` solution contains the full Evidently-to-Prometheus bridge used by this detector.
