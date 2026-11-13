# Exercise 04 — Eval Set Drift and Rot Detection — Reference Solution

Paired learning file: `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/exercises/exercise-04-eval-set-drift-and-rot-detection.md`.

> **Scope note.** The paired exercise asks the learner to design the controls that detect *eval-set rot*: the condition where the fixed evaluation suite the release-gate relies on has become unrepresentative of production, and therefore its "pass" verdict stops predicting user experience. The reference solution produces (a) the rot-detection pipeline, (b) the eval-set versioning and refresh workflow, (c) the release-gate interaction, and (d) a review cadence and rubric. The illustrative product is the fabricated `chatcat` RAG surface used across this module family.

## 1. Solution overview

An eval set is a frozen artifact that the release-gate uses to decide whether a new model version, prompt version, or retrieval change is safe to ship. The gate is as trustworthy as the eval set's representativeness of production. Two failure modes degrade that trust over time:

1. **Eval-set drift (representativeness rot).** Production queries change — new topics, new user cohorts, new intents — but the eval set does not. The gate keeps passing against a distribution that has stopped existing.
2. **Eval-set contamination (ground-truth rot).** Expected answers become wrong. The world moved (product changed, docs updated, prices changed), but the gold answers in the eval set did not. A correct new system now fails the gate; a stale system passes it.

The control shape has two parts, matched to the two failure modes:

- A *distributional* check that compares the eval set to a rolling sample of production queries and raises a representativeness verdict.
- A *ground-truth* check that compares the eval set's gold answers to a periodically-rebuilt gold set and raises a correctness verdict.

Both verdicts are per eval-set version. When either verdict goes `fail`, the release-gate degrades the eval set's weight (or refuses to use it) and triggers an eval-refresh workflow owned by a joint SRE + ML-platform team.

This is the control the NIST AI Risk Management Framework names in `MEASURE 2.11` ("Fairness and bias — as identified in the MAP function — are evaluated and results are documented") and `MEASURE 3.2` ("Risk tracking approaches are considered for settings where AI risks are difficult to assess"): an evaluation whose representativeness is not itself audited is not a measurement; it is a ritual.

### 1.1 What counts as "the eval set"

An eval set is a directory tree under version control (or a rows-with-schema blob in an eval store) containing:

- **items** — each with a query, context (if applicable), gold answer, and a set of gold-answer criteria (exact, substring, regex, rubric id, judge prompt);
- **a schema** — declares the required fields and the per-item scoring strategy;
- **metadata** — authored-by, authored-at, last-reviewed-at, source (synthetic, sampled-from-prod, redacted-prod), domain tags, version id.

The *version id* is a content hash over the items (sorted and canonicalised) plus the schema. Any change — one new item, one updated gold answer, a renamed field — yields a new eval version.

## 2. Implementation — worked answer

### 2.1 Representativeness check

The representativeness check compares the eval set's query distribution to a rolling sample of production queries along two axes: **intent / topic** (via query embeddings and clustering) and **surface shape** (query length, token distribution, presence of structured fields). Both axes are versioned against the same reference model used to compute the production embeddings.

```python
# evalrot/representativeness.py
from __future__ import annotations

import math
from dataclasses import dataclass
from statistics import median


@dataclass
class ClusterCoverage:
    cluster_id: int
    prod_share: float
    eval_share: float

    @property
    def shortfall(self) -> float:
        return max(0.0, self.prod_share - self.eval_share)


def coverage(prod_clusters: list[int], eval_clusters: list[int], n_clusters: int) -> list[ClusterCoverage]:
    def share(labels: list[int]) -> list[float]:
        totals = [0] * n_clusters
        for c in labels:
            totals[c] += 1
        s = sum(totals) or 1
        return [t / s for t in totals]

    ps = share(prod_clusters)
    es = share(eval_clusters)
    return [
        ClusterCoverage(cluster_id=i, prod_share=ps[i], eval_share=es[i])
        for i in range(n_clusters)
    ]


def representativeness_verdict(cov: list[ClusterCoverage]) -> tuple[str, float]:
    """Verdict: pass | watch | fail, and the total mass of shortfall."""
    total_shortfall = sum(c.shortfall for c in cov)
    # "mass of production intents that the eval set under-covers"
    if total_shortfall >= 0.25:
        return "fail", total_shortfall
    if total_shortfall >= 0.10:
        return "watch", total_shortfall
    return "pass", total_shortfall
```

The two bands (0.10 / 0.25) are the reference defaults for `chatcat`. They are deliberately generous for `watch` and strict for `fail`: the goal is early notification without false alarms, and a strict stop before the gate's signal is worthless.

Clustering uses a KMeans or HDBSCAN pass over query embeddings; the number of clusters is a configurable hyperparameter (`chatcat` uses 48). The reference centroids are recomputed when the embedding model version changes, and the eval-set and prod-sample must be embedded by the *same* version — otherwise the "distance" is between coordinate systems.

### 2.2 Ground-truth-rot check

Ground-truth rot is subtler than representativeness: it requires a trustworthy comparison for each gold answer. The reference pipeline uses three complementary sources, in increasing order of cost:

1. **Deterministic source checks.** If the gold answer is a fact that lives in a system of record (a price, a status, a boolean), the eval runner can re-fetch it and compare. Fast, no human.
2. **Rubric-based judges.** For prose answers, a separate judge (a strong model with a locked rubric, verified against a human-labelled calibration set) re-scores the gold answer against current retrieval context. Divergences flag the item.
3. **Human spot-check.** A weekly rotation of eval items — stratified so every cluster is touched monthly — is reviewed by the owning team.

Every item carries an `last_verified_at` timestamp; items whose last verification is older than 90 days are automatically queued for human review. Items that fail a deterministic or rubric check are marked `stale` and excluded from the gate until refreshed.

```python
# evalrot/ground_truth.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass
class EvalItem:
    id: str
    query: str
    gold_answer: str
    verification_kind: str   # 'deterministic' | 'rubric' | 'human'
    last_verified_at: datetime
    verification_version: str   # version of the verifier (rubric id, judge model, etc.)


def is_overdue(item: EvalItem, now: datetime, max_age: timedelta) -> bool:
    return (now - item.last_verified_at) > max_age


def rot_sweep(items: list[EvalItem], now: datetime, max_age: timedelta = timedelta(days=90)) -> dict[str, list[str]]:
    overdue, by_kind = [], {"deterministic": [], "rubric": [], "human": []}
    for item in items:
        if is_overdue(item, now, max_age):
            overdue.append(item.id)
            by_kind[item.verification_kind].append(item.id)
    return {"overdue": overdue, **by_kind}


def ground_truth_verdict(
    items: list[EvalItem],
    now: datetime,
    max_age: timedelta = timedelta(days=90),
    fail_fraction: float = 0.2,
    watch_fraction: float = 0.08,
) -> tuple[str, float]:
    if not items:
        return "fail", 1.0
    overdue = sum(1 for it in items if is_overdue(it, now, max_age))
    frac = overdue / len(items)
    if frac >= fail_fraction:
        return "fail", frac
    if frac >= watch_fraction:
        return "watch", frac
    return "pass", frac
```

Two defaults that matter:

- The 90-day max age is calibrated against how often `chatcat`'s source content refreshes; organisations on a slower cadence (e.g. compliance eval sets) may use 180 days, and fast-moving product surfaces may use 30 days.
- The 20 % `fail_fraction` is a joint-ownership tripwire: it is deliberately set so that an eval set which has quietly aged out of relevance stops being used before it certifies a regression.

### 2.3 Release-gate interaction

The release-gate (defined in `mod-111 / exercise-05`) requires every eval set it consults to carry current representativeness and ground-truth verdicts. The gate's behaviour is:

- **Both verdicts `pass`.** Eval set counts at full weight.
- **Either `watch`.** Eval set counts at reduced weight (reference default: 0.5); gate still issues a verdict but the ticket for `watch` is open.
- **Either `fail`.** Eval set is excluded from the gate and the gate refuses to issue a `pass` verdict on any release that would depend on it. A separate "unblock" workflow exists (next section) and is deliberately expensive.

Each release-gate verdict carries the eval-set versions it used and their verdicts, so a later audit can tell whether a past "pass" was issued against a fully-trusted eval or a degraded one.

### 2.4 Eval-refresh workflow

A `fail` verdict triggers a scheduled refresh workflow with explicit human steps:

1. **Sampling.** An automated step samples `N` new candidate items from production, applying PII redaction and stripping any user-identifying fields. The sample is stratified by the under-covered clusters identified in §2.1.
2. **Gold answering.** For deterministic items, the pipeline fills in current values. For rubric items, a human reviewer or a judge-plus-review process fills in gold answers.
3. **Spot-check.** A paired reviewer (different from the gold-answer authors) runs a small blind re-scoring on a 10 % sample to verify the new items are not themselves broken.
4. **Promotion.** The new eval set version is content-hashed, stored, and attached to the gate config. The prior version is archived (never deleted — it is a historical artifact for audits of past release verdicts).
5. **Re-evaluation of pending releases.** Any release currently blocked on the failed eval is re-evaluated against the refreshed set.

The workflow runs on a Kubernetes CronJob with manual approval gates between steps (2 → 3 and 3 → 4). It is deliberately not fully automated: a gold-answer edit has the same blast radius as a prompt change and belongs in the audited change path.

### 2.5 Dashboards and recording rules

`prometheus/rules/eval-rot.yaml`:

```yaml
groups:
  - name: chatcat-eval-rot
    interval: 10m
    rules:
      - record: chatcat:eval_representativeness:current
        expr: chatcat_eval_representativeness_shortfall{eval_set=~".+"}

      - record: chatcat:eval_overdue_fraction:current
        expr: |
          chatcat_eval_overdue_items{eval_set=~".+"}
          /
          chatcat_eval_total_items{eval_set=~".+"}

      - alert: ChatcatEvalRepresentativenessFail
        expr: chatcat:eval_representativeness:current >= 0.25
        for: 1h
        labels:
          severity: ticket
          incident_class: eval-rot
        annotations:
          summary: "eval set {{ $labels.eval_set }} under-covers production ({{ $value }})"
          runbook: https://runbooks.example/rb-eval-refresh

      - alert: ChatcatEvalOverdueFail
        expr: chatcat:eval_overdue_fraction:current >= 0.2
        for: 1h
        labels:
          severity: ticket
          incident_class: eval-rot
        annotations:
          summary: "eval set {{ $labels.eval_set }} is {{ $value }} overdue"
          runbook: https://runbooks.example/rb-eval-refresh
```

Alerts are tickets, not pages: eval rot is a slow-moving signal whose fix cadence is days, not minutes. The gate's `fail` behaviour is the thing that stops bad releases — the alerts are what trigger the refresh workflow.

### 2.6 Runbook extract — `rb-eval-refresh`

```markdown
# rb-eval-refresh

## When
- `ChatcatEvalRepresentativenessFail` or `ChatcatEvalOverdueFail` alert fires.
- Or: the release-gate is refusing a release because an eval set is in `fail`.

## Preconditions
- You know the eval set name.
- You have the joint SRE + ML-platform channel open.

## Steps
1. `evalctl status --set <name>` — confirm the current verdicts and the shortfall / overdue numbers.
2. If `representativeness=fail`: start the sampling workflow. `evalctl refresh --set <name> --mode representativeness --n 200`.
3. If `overdue=fail`: start the verification sweep. `evalctl refresh --set <name> --mode ground-truth`.
4. Review the generated items with the owning team (spot-check sample required).
5. Promote: `evalctl promote --set <name> --version <new-id>`.
6. Confirm the gate re-evaluates pending releases automatically; if not, poke `release-gate reset --eval-set <name>`.

## Post-incident
- Open a follow-up issue on the owning team if the same eval set has rotted twice in a quarter — the refresh cadence is too slow.
```

## 3. Validation steps

1. **Representativeness arithmetic.** `python -m pytest evalrot/tests/test_representativeness.py -q` — fixtures include matching shares (verdict pass), 15 % shortfall (watch), 35 % shortfall (fail), and degenerate inputs (empty prod sample — treated as unknown and fails closed).
2. **Ground-truth sweep.** Fixtures where every item is current (pass), 10 % overdue (watch), 30 % overdue (fail), and an empty set (fail with full shortfall).
3. **Release-gate interaction.** A fixture gate config with two eval sets: flip one to `watch` and assert the combined weight halves; flip it to `fail` and assert the gate refuses to issue a pass verdict.
4. **Refresh workflow dry-run.** Run the CronJob with `--dry-run` against a staging eval set; assert that the two manual approval gates are honoured (the job waits) and that the output eval version is a fresh content hash.
5. **Audit replay.** Replay a past release's gate verdict through a current audit tool. The audit must show which eval-set versions were used, each with its verdict at the time, so a past `pass` can be re-interpreted.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Both rot modes handled *(hard gate)* | Neither | One | Representativeness and ground-truth checks both implemented |
| 2 | Eval set versioning *(hard gate)* | Floating | Hashed but not required | Content-hash id required; changes refuse unless hash is recomputed |
| 3 | Release-gate interaction *(hard gate)* | Gate ignores rot | Gate warns but still passes | Gate degrades weight on `watch`; refuses on `fail` |
| 4 | Ground-truth verification kinds | Only human | Human + rubric | Deterministic + rubric + human, chosen per item |
| 5 | Refresh workflow present *(hard gate)* | Missing | Fully automated | Manual approval gates on gold-answer edits |
| 6 | PII and privacy handling in sampling | Missing | Named but not enforced | Automated redaction and strip of identifying fields with audit log |
| 7 | Dashboards and alerts | Missing | Alerts exist | Alerts + recording rules + dashboard; alerts are tickets, not pages |
| 8 | Audit replay possible | No | Partial | Past release verdict can be reconstructed with the eval versions used |
| 9 | Review cadence named | Missing | Named but vague | Weekly rotation + monthly cluster-sweep + quarterly ownership review |
| 10 | Integration with upstream modules | Isolated | Names one | Names `mod-111 / exercise-05` release gate and `mod-104 / exercise-04` eval tooling |

## 5. Common mistakes

- **Treating the eval set as a static artifact.** An eval set that has not changed in a year is almost certainly irrelevant to today's traffic. Review cadence has to be named and enforced.
- **Only measuring ground truth, not representativeness.** An eval with perfect gold answers but no coverage of today's intents is a very expensive way to not notice regressions.
- **Only measuring representativeness, not ground truth.** An eval with perfectly covered intents but stale expected answers will fail correct new versions and pass stale ones.
- **Automating the gold-answer edits.** A gold-answer change is a change to the control used to gate every release. It gets manual approval, the same as a prompt change (exercise 01).
- **Sampling from production without PII handling.** Eval items are shared, replayed, and often attached to tickets. Redaction has to be part of the sampling job, not a later step.
- **Alerts that page.** Eval rot moves in days. A pageable alert on eval rot burns the rotation; a ticketing alert aimed at a joint team gets fixed.
- **Rebaselining the eval centroids without rebaselining the embedding version.** The cluster-shortfall statistic is only meaningful when prod and eval are embedded by the same model version. Keep them paired.

## 6. References

- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `MEASURE 2.11` and `MEASURE 3.2` on evaluation documentation and risk-tracking in hard-to-assess settings.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML05 model theft` and `ML09 output integrity attack` motivate why the eval suite is an organisational asset to version and protect, not a notebook.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the `AML.T0019 — Publish Poisoned Datasets` and `AML.T0020 — Poison Training Data` technique classes have a direct analogue in eval-set sampling: untrusted inputs promoted to gold answers.
- Google SRE Workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — informs the choice to use ticketing alerts rather than paging for slow-moving signals.
- Prior-module references: `mod-111 / exercise-05` for the joint eval + SLO release gate contract, `mod-104 / exercise-04` for the LLM trace / eval platform the rubric judges live in.
