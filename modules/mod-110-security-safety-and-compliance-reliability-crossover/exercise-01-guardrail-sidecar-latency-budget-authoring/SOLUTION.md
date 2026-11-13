# Exercise 01 — Guardrail Sidecar Latency Budget Authoring — Reference Solution

Paired learning file: `lessons/mod-110-security-safety-and-compliance-reliability-crossover/exercises/exercise-01-guardrail-sidecar-latency-budget-authoring.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below
> is derived from this module's chapter on guardrails-as-a-reliability-surface. If the
> authored exercise narrows or widens scope on a subsequent content cycle, this solution
> should be regenerated.

> **Scope boundary.** A guardrail sidecar is a reliability surface the AI SRE owns jointly
> with the safety / security peer. The AI SRE's authoring task is the *latency budget* and
> its enforcement machinery — SLO contribution, stage allocation, burn-rate alerts,
> fallback-on-breach policy. The safety peer owns *what the guardrail decides* and the
> policy content. This exercise is the reliability half.

## 1. Solution overview

The deliverable is a committed document `docs/reliability/guardrail-sidecar-latency-budget.md`
plus two machine-readable artifacts:

1. A Prometheus recording-rule / alert file that encodes the per-stage budget and the
   burn-rate alerts on budget-breach.
2. A policy file for the sidecar (YAML) that encodes per-stage soft and hard timeouts,
   the fail-open / fail-closed stance per stage, and the fallback the request takes when a
   stage breaches.

The reference solution below is written against an illustrative chat surface
(`/chat/completions`) with a representative guardrail sidecar in front of the model
provider. The scenario is pedagogical; the budget structure, the stage taxonomy, the
burn-rate math, and the fallback policy are the reusable shape.

The budget is sized by working **top-down from the SLO**: the user-facing SLO allocates a
share to the sidecar, and the sidecar's share is subdivided across its stages with
headroom reserved for the unlucky case (a cold-path configuration fetch, a policy-model
cold start, a provider retry). A latency budget that is sized bottom-up from per-stage
means is wrong by construction because real latency is a long-tail composition.

## 2. Implementation — worked answer

### 2.1 Context — what the sidecar does

The sidecar sits in-line on the `/chat/completions` route and performs the following
stages per request, in order:

| # | Stage | What it does | Fail-closed? |
|---|-------|--------------|---------------|
| 1 | `auth` | Validates the caller's tenant token and resolves the policy set. | Yes (reliability + security peer agree). |
| 2 | `policy-fetch` | Loads the tenant's policy bundle (cache-fronted). | Soft-fail (stale-ok). |
| 3 | `input-scan` | Runs the input against the policy: content classes, prompt-injection heuristics, PII patterns. | Decided per policy class; see §2.5. |
| 4 | `rate-shape` | Applies tenant-shaped admission (quota, concurrency). | Yes. |
| 5 | `model-call` | Proxies to the model provider (primary, then fallback on breaker). | n/a — this is the main request. |
| 6 | `output-scan` | Runs the output against the policy: safety classes, data-exfil patterns. | Decided per policy class; see §2.5. |
| 7 | `audit-emit` | Emits the structured audit record. | Soft-fail (buffered). |

Everything on the request path that is **not** `model-call` is **sidecar overhead**. The
budget partitions sidecar overhead across stages 1–4, 6, 7.

### 2.2 Budget — top-down derivation

Working top-down from the user SLO (illustrative values; a real authoring would pull the
SLO target from `mod-103`'s SLO document for this route):

```
user SLO: p95 end-to-end latency on /chat/completions <= 2500 ms (28-day window, 99% attainment)
```

The SLO is partitioned as follows. Partition shares are *contributions* to the
end-to-end p95; they are not independent additions (real latency composes with tails),
and the budget uses conservative additive sizing to leave tail headroom.

| Component | p95 share | Notes |
|-----------|-----------|-------|
| Edge + ingress + routing | 50 ms | Load balancer, mesh, ingress proxy. Owned by platform. |
| Sidecar overhead (this doc) | 350 ms | Stages 1, 2, 3, 4, 6, 7. Owned by AI SRE jointly with safety peer. |
| Model call | 2000 ms | Primary provider; see `mod-107` for provider-side budget. |
| Egress + client-side TTFT | 100 ms | Owned by platform. |

The sidecar's 350 ms is subdivided:

| Stage | p95 budget | p99 soft timeout | Hard timeout | Rationale |
|-------|-----------|------------------|---------------|-----------|
| `auth` | 20 ms | 50 ms | 100 ms | Cached JWT verification; policy-set lookup is from a local cache fronted by Redis with LRU. The budget covers cache-miss. |
| `policy-fetch` | 30 ms | 80 ms | 150 ms | Policy bundle is cached in-process; fetch is only on cold start or TTL expiry. The budget covers cold-start and the fallback to a stale-ok bundle. |
| `input-scan` | 120 ms | 250 ms | 400 ms | Dominant stage. Includes pattern matches (fast) + classifier inference (slow). Classifier is the long tail. |
| `rate-shape` | 5 ms | 10 ms | 20 ms | Token-bucket decrement + concurrency gate. |
| `output-scan` | 150 ms | 300 ms | 500 ms | Larger than input-scan because output size is typically bigger (full completion) and the classifier runs on the same model class. Streaming mode scans in chunks; the hard timeout applies to the *last-token-emit* deadline. |
| `audit-emit` | 15 ms | 30 ms | n/a (async) | Non-blocking in nominal mode; the hard timeout does not apply because this is a background emit. |
| Bookkeeping + context propagation | 10 ms | 20 ms | 50 ms | Per-request state, trace context, header fan-out. |
| **Sum** | **350 ms** | — | — | p95 budget sums to the sidecar's SLO share. |

**Why these numbers.** Each stage's p95 was derived by (a) measuring the current stage's
p95 against the last 28 days in staging + shadow-prod, (b) adding a 2× headroom because
the current measurement is in isolation and real production has correlated tails, and (c)
checking the sum against the SLO share. The `input-scan` and `output-scan` stages are
budgeted deliberately large — they are the two stages the safety peer is actively iterating
on, and a tight budget would couple safety-policy changes to a reliability regression.

**Why not smaller.** A sidecar that targets p95 at 200 ms looks faster on paper but is a
reliability trap: the p99 tail routinely runs 3–5× the p95 for classifier-based stages,
and a 200 ms p95 implies a p99 above the SLO's hard deadline, which converts long-tail
runs into user-visible errors.

**Why not larger.** The `mod-103` SLO would not be met. The sidecar must live within its
top-down share; the way to buy more sidecar time is to negotiate the SLO or to reduce the
model-call share (e.g., by batching, smaller models, caching), not to inflate the sidecar
unilaterally.

### 2.3 Document — `docs/reliability/guardrail-sidecar-latency-budget.md`

```markdown
---
owner: ai-sre-function
co-owner: safety-engineering-function
reviewed: quarterly
scope: /chat/completions guardrail sidecar
---

# Guardrail sidecar latency budget — v2026-11-01

## 1. SLO reference

The sidecar's budget is a partition of the `/chat/completions` end-to-end latency SLO
(see `mod-103/slo-chat-completions.md`). Any change to the SLO target or the SLO
attainment window triggers a re-partitioning; the sidecar's share is **never** adjusted
in isolation.

- Route: `/chat/completions`
- SLO: p95 <= 2500 ms, 99% attainment, 28-day window
- Sidecar share: **350 ms p95** (14% of the SLO target)

## 2. Stage budget

(Reproduce the table from §2.2.)

## 3. Enforcement

### 3.1 Per-stage timeouts

The soft timeout is advisory: when a stage exceeds its soft timeout, the request
continues (unless the stage is a hard-closed stage; see §3.3) and the
`guardrail_stage_soft_timeout_total{stage=...}` counter increments. Soft timeouts drive
alerts; they do not short-circuit requests.

The hard timeout is enforced: at the hard timeout the stage is cancelled. What happens
next depends on the stage's fail-open / fail-closed stance.

### 3.2 Fail-open vs. fail-closed per stage

| Stage | Stance | What happens at hard timeout |
|-------|--------|-------------------------------|
| `auth` | Closed | Request rejected with HTTP 401; audit emitted. |
| `policy-fetch` | Open-to-stale | Request proceeds with the last-known-good policy bundle (TTL < 24h); counter increments. |
| `input-scan` | Policy-decided | Per policy class: `block-on-timeout` (CBRN, CSAM, self-harm) → request rejected; `allow-on-timeout` (style, brand) → request proceeds. |
| `rate-shape` | Closed | Request rejected with HTTP 429; audit emitted. |
| `output-scan` | Policy-decided | Per policy class: `block-on-timeout` → response withheld + replaced with a safe completion; `allow-on-timeout` → response delivered. |
| `audit-emit` | Open | Response delivered; emit goes to the buffered queue. |

The policy-decided stances mean the **policy content** (owned by the safety peer)
determines the stance per class; the **enforcement machinery** (owned by the AI SRE)
executes it uniformly. Any class without an explicit `block-on-timeout` label defaults to
`block-on-timeout` (safety-peer-set default; chapter §4.3 principle).

### 3.3 Budget-breach fallback (sidecar-wide)

When the sidecar's end-to-end stage sum exceeds the 350 ms p95 budget for a sustained
window (burn-rate alert fires — see §4), the sidecar enters a degraded mode:

1. **Policy-class downshift.** Non-critical policy classes (style, brand, adjacency) are
   bypassed; safety-critical classes (CBRN, CSAM, self-harm, PII-exfil) remain on.
2. **Classifier downshift.** Classifier calls fall back to the pattern-matcher-only
   configuration for the duration of the degraded mode.
3. **Signal.** `guardrail_degraded_mode_active{route=...}` gauge = 1.
4. **Audit.** Every request served in degraded mode has a `guardrail.mode=degraded`
   attribute on its audit record; safety peer receives a daily digest of degraded-mode
   requests.
5. **Exit condition.** Three consecutive 1-minute windows below the soft threshold.

### 3.4 Rollback and preservation

A policy change that pushes the budget out of attainment is a reliability regression. The
release-gate (see exercise-04) halts promotion. The already-promoted version is subject
to automatic rollback per `mod-111`'s rollback runbook. The budget breach, the policy
change, and the rollback are preserved as an incident (see exercise-05 on sensitive-data
handling when the breach exposes customer content).

## 4. Observability — recording rules and alerts

The recording rules compute per-stage latency distributions; the alerts fire on burn-rate
exceedance of the stage budget or the sidecar-wide budget.

- **Burn-rate alerts** follow the multi-window multi-burn-rate shape from `mod-103`:
  - Fast burn: 2% of 28-day budget burned in 1 hour (sensitive; pages).
  - Slow burn: 10% of 28-day budget burned in 6 hours (ticket-level).

- **Per-stage alerts** fire when a stage's p95 is above its budget for 30 consecutive
  minutes. These are warn-level; they drive a review conversation with the safety peer
  rather than a page.

- **Degraded-mode gauge** is an alert in its own right at any sustained value > 0:
  degraded mode is a pager event because the user population is seeing a reduced
  guardrail.

(See §2.4 for the recording rule and alert definitions.)
```

### 2.4 Prometheus rules — `observability/guardrail-sidecar.rules.yaml`

```yaml
groups:
  - name: guardrail-sidecar-recording
    interval: 15s
    rules:
      - record: guardrail:stage_latency_seconds:p95
        expr: |
          histogram_quantile(
            0.95,
            sum by (stage, le) (rate(guardrail_stage_latency_seconds_bucket[5m]))
          )

      - record: guardrail:total_latency_seconds:p95
        expr: |
          histogram_quantile(
            0.95,
            sum by (le) (rate(guardrail_total_latency_seconds_bucket[5m]))
          )

      - record: guardrail:sidecar_budget_burn_ratio
        # Observed sidecar p95 / budget. > 1 means the budget is being burned.
        expr: |
          guardrail:total_latency_seconds:p95 / 0.350

  - name: guardrail-sidecar-alerts
    rules:
      - alert: GuardrailSidecarSLOFastBurn
        # Multi-window multi-burn-rate per mod-103: fast burn on the 2%/1h window.
        # Error-budget consumption is measured as fraction of requests where total
        # sidecar latency exceeds the 350 ms stage budget.
        expr: |
          (
            sum(rate(guardrail_total_latency_seconds_count{le="+Inf"}[1h]))
            -
            sum(rate(guardrail_total_latency_seconds_bucket{le="0.350"}[1h]))
          )
          /
          sum(rate(guardrail_total_latency_seconds_count{le="+Inf"}[1h]))
          > (14.4 * 0.01)
        for: 2m
        labels:
          severity: page
          slo: chat-completions-latency
          owner: ai-sre-oncall
        annotations:
          summary: "Guardrail sidecar burning latency budget at fast-burn rate"
          runbook: rb-guardrail-latency-regression

      - alert: GuardrailSidecarSLOSlowBurn
        expr: |
          (
            sum(rate(guardrail_total_latency_seconds_count{le="+Inf"}[6h]))
            -
            sum(rate(guardrail_total_latency_seconds_bucket{le="0.350"}[6h]))
          )
          /
          sum(rate(guardrail_total_latency_seconds_count{le="+Inf"}[6h]))
          > (6 * 0.01)
        for: 15m
        labels:
          severity: ticket
          slo: chat-completions-latency
          owner: ai-sre-oncall

      - alert: GuardrailStageOverBudget
        # Per-stage warn: the stage's p95 is above its budget for 30m.
        expr: |
          guardrail:stage_latency_seconds:p95
            > on (stage) group_left
          guardrail_stage_budget_seconds
        for: 30m
        labels:
          severity: warn
          owner: ai-sre-oncall
        annotations:
          summary: "Stage {{ $labels.stage }} over budget for 30m"

      - alert: GuardrailDegradedModeActive
        expr: guardrail_degraded_mode_active == 1
        for: 1m
        labels:
          severity: page
          owner: ai-sre-oncall
        annotations:
          summary: "Guardrail degraded mode active on {{ $labels.route }}"
          runbook: rb-guardrail-degraded-mode
```

### 2.5 Policy file — `config/guardrail-sidecar-budget.yaml`

```yaml
version: 2026-11-01
owner: ai-sre + safety-engineering
route: /chat/completions

slo:
  end_to_end_p95_ms: 2500
  attainment: 0.99
  window: 28d

sidecar_budget:
  p95_ms: 350
  stages:
    - name: auth
      p95_ms: 20
      soft_timeout_ms: 50
      hard_timeout_ms: 100
      stance: closed
    - name: policy-fetch
      p95_ms: 30
      soft_timeout_ms: 80
      hard_timeout_ms: 150
      stance: open-to-stale
      stale_bundle_ttl_s: 86400
    - name: input-scan
      p95_ms: 120
      soft_timeout_ms: 250
      hard_timeout_ms: 400
      stance: policy-decided
    - name: rate-shape
      p95_ms: 5
      soft_timeout_ms: 10
      hard_timeout_ms: 20
      stance: closed
    - name: output-scan
      p95_ms: 150
      soft_timeout_ms: 300
      hard_timeout_ms: 500
      stance: policy-decided
    - name: audit-emit
      p95_ms: 15
      soft_timeout_ms: 30
      stance: open
      async: true
    - name: bookkeeping
      p95_ms: 10
      soft_timeout_ms: 20
      hard_timeout_ms: 50
      stance: closed

degraded_mode:
  trigger_alert: GuardrailSidecarSLOFastBurn
  exit:
    consecutive_minutes_below_soft_threshold: 3
  downshift:
    bypass_classes:
      - style
      - brand
      - adjacency
    keep_classes:
      - cbrn
      - csam
      - self-harm
      - pii-exfil
    classifier_fallback: pattern-matcher-only
  audit_attribute: guardrail.mode=degraded
  digest: safety-engineering-oncall@daily
```

## 3. Validation steps

1. **Budget-sum check.** Automated: parse the YAML and verify that the stages' `p95_ms`
   sum to `sidecar_budget.p95_ms`. Any deviation > 1 ms is a lint failure.

   ```bash
   python3 - <<'EOF'
   import yaml, sys
   cfg = yaml.safe_load(open('config/guardrail-sidecar-budget.yaml'))
   stages_sum = sum(s['p95_ms'] for s in cfg['sidecar_budget']['stages'])
   total = cfg['sidecar_budget']['p95_ms']
   assert abs(stages_sum - total) <= 1, f"stage sum {stages_sum} != budget {total}"
   print("OK")
   EOF
   ```

2. **SLO consistency check.** Automated: the sidecar `p95_ms` must be <= `slo.end_to_end_p95_ms`
   minus the platform and model shares declared in the SLO document. If the SLO document
   is updated, this check fails until the sidecar budget is re-partitioned.

3. **Alert replay.** The fast-burn and slow-burn alerts are replayed against a
   captured 7-day trace from staging to confirm they fire on known-bad windows and are
   silent on known-good windows. (The multi-window burn-rate math is validated against
   the `mod-103` calculator.)

4. **Fail-closed drill.** Semi-annual: inject a 10× latency on `input-scan` in staging
   and verify (a) the hard timeout engages, (b) policy-class-decided requests fail the
   correct way per class, (c) the degraded-mode gauge turns on when the burn-rate alert
   fires, (d) the gauge clears after the exit condition.

5. **Policy-peer sign-off.** Every change to a stage's `stance` or to `degraded_mode` is
   co-reviewed by the safety peer. The AI SRE never changes a policy-decided stance
   unilaterally; the safety peer never changes a budget unilaterally. The joint-authorship
   discipline from chapter §2.2 is enforced in the PR template.

6. **Audit coverage.** Every request has at least one audit record; degraded-mode
   requests have `guardrail.mode=degraded`. A daily job counts audit records against
   request-count by route and fails the build on ratio < 0.999.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at >= 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Budget is top-down from SLO *(hard gate)* | Budget derived without reference to SLO | SLO referenced but partition unclear | Partition documented from SLO to sidecar to stages |
| 2 | Per-stage budget with p95 + soft + hard *(hard gate)* | One or more stages missing a value | All present but no rationale | All present + rationale per stage |
| 3 | Fail-open vs. fail-closed per stage *(hard gate)* | Unset or implicit | Set but inconsistent | Set per stage + policy-decided stance uses the safety peer's class map |
| 4 | Budget-breach fallback behaviour | None | Named but vague | Named, specific downshift + exit condition + audit attribute |
| 5 | Burn-rate alerts | None | Single-window | Multi-window multi-burn-rate per mod-103 |
| 6 | Per-stage alerts | None | Static threshold | Budget-coupled: alert fires when stage p95 > budget |
| 7 | Degraded-mode gauge + alert | Missing | Present as metric | Alert + runbook + daily digest to safety peer |
| 8 | Co-authorship with safety peer *(hard gate)* | Single author | Reviewed by safety | Joint-owner header + policy stance changes require safety-peer sign-off |
| 9 | Rollback interlock with release gate | Not referenced | Referenced | Release gate halts on budget-breach; see exercise-04 |
| 10 | Audit coverage | None | Partial | Every request audited; degraded-mode attributed; preservation discipline from exercise-05 |

## 5. Common mistakes

- **Sizing bottom-up from measured means.** Real tails compose; the budget needs
  headroom for correlated p99s. Size top-down from the SLO.
- **Partitioning in isolation from the SLO document.** A sidecar budget is a share of
  the end-to-end SLO. A silent change here is a silent change to the SLO's attainment.
- **Treating the budget as the mean target.** The p95 is a tail target; the mean may be
  far lower. Operators sometimes confuse "we're at 150 ms" (mean) with "we're at 150 ms"
  (p95).
- **Fail-open on safety-critical classes.** The default stance for safety-critical
  classes is `block-on-timeout`. A policy that is silent on a class is treated as
  safety-critical by default (safety-peer-set default).
- **Dropping classifier calls silently under load.** A classifier that is bypassed
  without a degraded-mode gauge and an audit attribute is an invisible loss of the
  guardrail. Make the degradation visible and audible (gauge + digest).
- **Co-locating the budget change with a policy change in one PR.** Separate PRs; the
  reviewer composition is different (AI SRE vs. safety peer).
- **Static per-stage thresholds that drift from the budget.** Alerts must be
  budget-coupled; a static threshold rots when the budget changes.
- **Treating `audit-emit` as a blocking stage.** `audit-emit` is async; the hard
  timeout does not apply on the request path. If the audit queue is backed up, that's a
  separate reliability concern with its own SLO.
- **Forgetting the rollback interlock.** A policy change that regresses the budget must
  trigger a rollback (see exercise-04 for the release-gate reconciliation).
- **Partitioning only p95 and ignoring p99.** The SLO is p95, but the stage's
  `hard_timeout_ms` is effectively the p99.9 ceiling. Both numbers should be in the
  policy file.

## 6. References

- Module chapter on guardrails-as-a-reliability-surface (this module) — the joint-authorship
  discipline, the sidecar taxonomy, the policy-decided stance, and the degraded-mode
  pattern that this exercise implements.
- `mod-103`, *SLO engineering for model-backed services* — the SLO partition shape and
  the multi-window multi-burn-rate alert math referenced in §3.2 and §2.4.
- `mod-104`, *Observability for LLM and GPU workloads* — the telemetry conventions for
  per-stage latency histograms.
- `mod-107`, *Vendor and model-provider risk* — the provider-side budget and the
  breaker-to-fallback pattern.
- `mod-111`, *Release engineering and progressive delivery* — the rollback runbook
  interlocked with the budget-breach signal.
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary for the policy classes the sidecar enforces (notably ML01 input
  manipulation, ML02 data poisoning when the policy touches training-data egress).
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — adversarial-technique
  vocabulary the input-scan policies reference (prompt injection, jailbreak evasions).
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the Measure / Manage functions the latency budget operationalises.
- NIST AI 600-1, *Generative AI Profile*, [nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
  — GAI-specific risks (GV-, MP-, MG- actions) that map to the policy-decided classes.
