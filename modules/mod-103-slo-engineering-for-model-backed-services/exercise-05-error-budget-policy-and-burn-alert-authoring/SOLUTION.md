# Exercise 05 — Error-Budget Policy and Burn-Alert Authoring — Reference Solution

## 1. Solution overview

Operable error-budget policy plus Prometheus-shaped burn-rate alert
configuration for the **`acme-llm-gw`** SLO stack built in exercises
01–04.

- Part A — the full eight-section policy document a new on-call
  reads at 03:00.
- Part B — ready-to-drop Prometheus alert rules for five SLIs
  spanning three dimensions (latency, quality, cost). Rules are
  syntactically well-formed and the thresholds are *recomputed per
  SLO*, not copy-pasted.

The document is deliberately operable: every band has specific
actions, every correlated-burn rule names a window, every alert
annotation points at a runbook URL.

---

## 2. Worked answer / Implementation

## Part A — the policy document

### 2.1 Section 1 — SLO surface

Compact restatement of the composed stack from exercises 01–04.
Full definitions / valid-event filters are in the per-exercise
SOLUTION.md.

| SLI | Target | Window | Burn-rate alert severity |
|---|---|---|---|
| `chat-stream` TTFT cache-miss | p95 ≤ 1.2 s | 28-day | fast/med page; slow ticket |
| `chat-stream` TTFT cache-hit | p95 ≤ 400 ms | 28-day | fast/med page; slow ticket |
| `chat-stream` TPOT | p95 ≤ 80 ms / tok | 28-day | fast/med page; slow ticket |
| `chat-stream` ITL | p99 ≤ 2 s | 28-day | fast page; med/slow ticket |
| `chat-stream` E2E | p95 ≤ 10 s | 28-day | fast/med page; slow ticket |
| `chat-stream` goodput | p90 ≥ 12 tok/s | 28-day | all ticket (capacity-shaped SLI) |
| `code-assist` TTFT | p95 ≤ 700 ms | 28-day | fast/med page; slow ticket |
| `code-assist` E2E (schema-valid) | p95 ≤ 2 s | 28-day | fast/med page; slow ticket |
| `batch-summarize` E2E per-ticket | p99 ≤ 180 s | 28-day | all ticket |
| `batch-summarize` E2E job | p99 ≤ 06:00 UTC weekdays | binary daily | daily miss page; 2 consecutive incident |
| Hallucination | ≥ 97% grounded | 28-day | fast/med page (via eval-peer channel); slow ticket |
| Toxic-output | ≥ 99.9% compliant | 28-day | fast page + security notify; med/slow ticket |
| Tool-call-success | ≥ 99% | 28-day | fast/med page; slow ticket |
| Structured-output-conformance | ≥ 99.5% | 28-day | fast page; med/slow ticket |
| Refusal — warranted | ≥ 99% | 28-day | fast page + security notify; med/slow ticket |
| Refusal — over-refusal | ≥ 98% | 28-day | all ticket |
| Protocol 2xx | ≥ 99.9% | 28-day | fast/med page; slow ticket |
| Partial-success | ≥ 99% | 28-day | fast page (when operable; blocked on gap); med/slow ticket |
| Graceful-degradation (aggregate) | ≥ 99.5% | 28-day | med ticket; slow review |
| Cost-per-1M-input (`chat-stream`) | ≤ $80 | 28-day | ticket-only except correlated-paging |
| Cost-per-1M-output (`chat-stream`) | ≤ $120 | 28-day | ticket-only except correlated-paging |
| Cost-per-completed-request (`code-assist`) | ≤ $0.0015 | 28-day | ticket-only except correlated-paging |
| Cost-per-successful-session (`chat-stream`) | ≤ $0.16 | 28-day | ticket-only except correlated-paging |

### 2.2 Section 2 — Freeze-targeting map

| SLI | Change types frozen on burn | Exception-granting authority |
|---|---|---|
| `chat-stream` TTFT cache-miss | Prompt-template changes, model version promotions, admission-queue config, prefix-cache config | Incident Commander (IC) |
| `chat-stream` TTFT cache-hit | Prefix-cache config, prompt-template changes | IC |
| `chat-stream` TPOT / ITL / goodput | Model version promotions, batching/scheduler config, GPU-shard rebalances | IC |
| `chat-stream` E2E | All of the above | IC |
| `code-assist` TTFT / E2E | Prompt-template changes, model promotions, constrained-decoding toggles | IC |
| `batch-summarize` E2E | Batch scheduler config, concurrency caps | IC |
| Hallucination | Prompt-template changes, retrieval-index swaps, judge-pipeline changes, model promotions | IC + eval-peer delegate |
| Toxic-output | Moderation classifier version, prompt safety-suffix | IC + security-peer delegate |
| Tool-call-success | Tool-schema registry changes, prompt tool-use examples | IC + tool-platform delegate |
| Structured-output-conformance | Constrained-decoding toggle, schema registry | IC |
| Refusal — warranted / over-refusal | Safety classifier, moderation prompt policy, refusal templates | IC + security-peer delegate |
| Protocol / Partial-success / Graceful-degradation | Gateway code deploys, LiteLLM routing policy changes, fallback-path toggles | IC |
| **Cost SLIs (all four)** | **Does not freeze.** Cost burn routes to tickets / reviews per exercise 04. Correlated-paging exceptions (§2.5) are the only pageable cases. | n/a |

The cost-row "does not freeze" entry is explicit because omitting
it from the freeze map would be read as "frozen by default under
the protocol-level freeze," which is wrong.

### 2.3 Section 3 — Allocation rule

Default allocation of the error budget per 28-day window is
**canary 40 / feature 40 / refactor 20** per chapter 05 section 2.

`acme-llm-gw` is currently in **early-rollout phase** with an
immature SLI surface (two open instrumentation gaps —
`stream.terminated.by` tagging and session ID propagation — flagged
in exercises 03 and 04). Deviation from default for the current
quarter:

| Category | Allocation (this quarter) | Default | Justification |
|---|---|---|---|
| Canary | 30% | 40% | Fewer canary deploys allowed until the two instrumentation gaps close |
| Feature | 30% | 40% | Reduced because the eval-peer `eval-pipeline-v1` is still being stood up; features that depend on judge-pipeline measurement are constrained |
| Refactor | 40% | 20% | Doubled to pay the instrumentation debt from exercises 03 and 04 |

Allocation is revisited at the next quarterly review (first
quarterly review, per §2.8). Expected trajectory: once the
instrumentation gaps close, we move back toward 40 / 40 / 20.

### 2.4 Section 4 — Burn bands

| Budget remaining | Canary allowed | Feature allowed | Refactor allowed | Additional required action |
|---|---|---|---|---|
| **100–50%** (healthy) | Default cadence (up to 3/week) | Default cadence | Default cadence | None |
| **50–25%** (caution) | At most 1 canary deploy per week; no new model rollouts; existing canaries continue | At most 2 feature releases per week; no prompt-template rewrites | Default cadence | Weekly health check to SLO review; IC gets a pre-brief |
| **25–10%** (restrictive) | No new canaries; existing canaries roll back to primary unless demonstrably helpful | No feature releases unless they contain SLI-improvement work; prompt changes require eval-peer sign-off | At most 2 refactors / week; high-risk refactors to IC for approval | Daily standup of SRE + eval + security peers; sponsor notified weekly |
| **< 10%** (frozen) | Canary deploys blocked | Feature releases blocked; only revert/rollback deploys permitted | Refactors blocked except those that directly repair the burning SLI | SRE-only scope reductions available to IC without sponsor sign-off; sponsor notified same-day; freeze is in effect per §2.2 map |

The bands are intentionally operable. Each row names *specific*
actions. "At most 1 canary per week" is a schedule constraint the
release tooling can enforce; "canary allocation reduces by half"
would be a design discussion rather than an operating rule.

### 2.5 Section 5 — Correlated-burn rule

**Correlation window: 2 hours** (SRE Workbook default). Chosen
because:

- Shorter (30 min) misses slow-correlated incidents like
  provider-version rollouts that propagate over 1–2 h.
- Longer (8 h) collapses unrelated SLI burns into a single
  "incident" that is actually two incidents.
- 2 h matches the medium-burn alert window, so a correlated-burn
  signal and a medium-burn page arrive on the same clock.

**Correlated-burn escalation path.**

1. Any two SLIs burning at *fast* rate within the 2-hour window
   automatically escalates to an IC call. Any two across
   *dimensions* (latency + quality, quality + cost, availability
   + cost) *always* escalates to IC, even at medium rate.
2. The IC runs the single-SLI runbooks in parallel using the
   secondary on-call for the second SLI; the IC personally runs
   the correlated-burn runbook (`runbooks/correlated-burn.md`)
   which drives rollback / provider-fallback / capacity decisions.
3. The eval peer is paged *during* a correlated burn that includes
   any quality SLI; the security peer during a correlated burn that
   includes toxic-output or warranted-refusal.
4. Handoff from single-SLI runbooks to the correlated-burn runbook
   happens at the *first fast-burn page of the second SLI*, not
   the first slow signal — the signal must be strong enough to
   justify the IC role overhead.

### 2.6 Section 6 — Eval-peer coordination rule

The two rules from chapter 05 section 5, parameterised:

1. **A gate-passing release that burns the production SLO can be
   rolled back by on-call without re-invoking the eval gate.** The
   release gate signals pre-ship confidence; the SLO measures
   post-ship reality. The eval peer is notified via
   `#eval-releases` of the rollback within 30 minutes; the gate
   is re-audited at the next SLO review for correction.
2. **A gate-failing release cannot ship by spending budget.** If
   the pre-ship gate failed, the release is blocked regardless of
   available error budget. The policy does not permit budget
   spending as a substitute for passing the gate.

Eval peer channel: **`#eval-releases`** (illustrative name;
dependency flagged as "to be stood up" if the channel does not
exist on the team). Eval peer on-call schedule mirrors SRE on-call
weekly rotation per `mod-101` coordination contract.

### 2.7 Section 7 — Exception process

The five-element exception process:

- **Who grants.**  The IC of an in-flight incident may grant an
  exception for a window of up to 24 h. The SRE lead may grant up
  to 7 d during non-incident periods; anything longer requires the
  sponsor.
- **For how long.**  Default 24 h; maximum 7 d non-incident, 14 d
  with sponsor sign-off.
- **What triggers.**  (a) An external dependency outage preventing
  normal operation (provider API down > 1 h; eval pipeline down
  > 4 h). (b) A regulatory / safety requirement overriding normal
  SLO procedure. (c) A dry-run of a major release where the team
  explicitly spends budget to learn.
- **How logged.**  Entry in the exception log (append-only
  `exceptions.md` in the ops repo) with granter, duration,
  trigger, expected resolution. Entry is created before the
  exception takes effect — no retroactive entries.
- **How reviewed.**  Every exception is reviewed at the next
  monthly SLO review (exercise 06 §2.2.3). Recurring triggers
  (same exception shape ≥ 2× in a quarter) force a policy update.

### 2.8 Section 8 — Review cadence

- **Monthly SLO review** (first Thursday of each month, 30 min).
  Attendees: primary + secondary on-call, SRE lead, business
  sponsor, eval peer, capacity owner. Chair rotates quarterly.
  Standing agenda from exercise 06.
- **Quarterly allocation refresh** (first SLO review of each
  quarter, extended to 60 min). Revisits §2.3 allocation split.
  Attendees as monthly plus platform-engineering lead.
- **Annual policy audit** (January). Full re-read of the policy
  against the past year's incidents and exceptions. Attendees:
  SRE lead, eval peer, security peer, sponsor, and an external
  reader (someone not on the team — a peer SRE from a sibling
  product). The external reader catches drift that insiders have
  normalised.

---

## Part B — the burn-rate alert configuration

Five alert groups, one per selected SLI, each with three rules
(fast page / medium page / slow ticket) except cost which omits
the fast-burn page per exercise 04. Every alert has a runbook URL
and a per-SLO-recomputed threshold.

Rules use recording rules for `:ratio_rate*m` series (per SRE
Workbook §5) — convention: `<sli>:error_rate:ratio_rate<window>`
computes error rate over the window. Recording rules themselves
are not shown (they are pure PromQL transforms); the burn-rate
rules reference them.

```yaml
# runbooks/alerts/mod-103-acme-llm-gw.yml
# Generated against the exercise-01..04 SLO stack.
# Author: SRE for acme-llm-gw.
# Thresholds recomputed per SLO target; do not reuse cross-SLO.

groups:
  - name: acme-llm-gw.ttft_cache_miss
    interval: 30s
    rules:
      # SLO: p95 <= 1.2s cache-miss on chat-stream. Target = 95%
      # Fast burn: 14.4 * (1 - 0.95) = 0.72 (72% error rate over 1h)
      - alert: TTFTCacheMissFastBurn
        expr: |
          (ttft_cache_miss:slo_error_rate:ratio_rate1h > (14.4 * (1 - 0.95)))
          and
          (ttft_cache_miss:slo_error_rate:ratio_rate5m > (14.4 * (1 - 0.95)))
        for: 2m
        labels:
          severity: page
          slo: ttft_cache_miss
          surface: chat-stream
          dimension: latency
          runbook: "https://runbooks.acme.example/ttft-cache-miss-burn"
        annotations:
          summary: "TTFT cache-miss burning fast: 2% of 28-day budget in 1h"
          description: "p95 TTFT (cache-miss) breaching 1.2s SLO. 2h budget exhaust expected if burn continues."

      # Medium burn: 6 * 0.05 = 0.30 (30% error rate over 6h)
      - alert: TTFTCacheMissMediumBurn
        expr: |
          (ttft_cache_miss:slo_error_rate:ratio_rate6h > (6 * (1 - 0.95)))
          and
          (ttft_cache_miss:slo_error_rate:ratio_rate30m > (6 * (1 - 0.95)))
        for: 5m
        labels:
          severity: page
          slo: ttft_cache_miss
          surface: chat-stream
          dimension: latency
          runbook: "https://runbooks.acme.example/ttft-cache-miss-burn"
        annotations:
          summary: "TTFT cache-miss burning medium: 5% of 28-day budget in 6h"

      # Slow burn: 1 * 0.05 = 0.05 (5% error rate over 3d)
      - alert: TTFTCacheMissSlowBurn
        expr: |
          (ttft_cache_miss:slo_error_rate:ratio_rate3d > (1 * (1 - 0.95)))
          and
          (ttft_cache_miss:slo_error_rate:ratio_rate6h > (1 * (1 - 0.95)))
        for: 30m
        labels:
          severity: ticket
          slo: ttft_cache_miss
          surface: chat-stream
          dimension: latency
          runbook: "https://runbooks.acme.example/ttft-cache-miss-burn"
        annotations:
          summary: "TTFT cache-miss burning slow: 10% of 28-day budget in 3d"

  - name: acme-llm-gw.hallucination
    interval: 30s
    rules:
      # SLO: hallucination <= 3% (i.e. >= 97% grounded). Target = 97%
      # Fast burn: 14.4 * 0.03 = 0.432
      - alert: HallucinationFastBurn
        expr: |
          (hallucination:slo_error_rate:ratio_rate1h > (14.4 * (1 - 0.97)))
          and
          (hallucination:slo_error_rate:ratio_rate5m > (14.4 * (1 - 0.97)))
        for: 2m
        labels:
          severity: page
          slo: hallucination
          surface: chat-stream
          dimension: quality
          co_owner: eval-peer
          runbook: "https://runbooks.acme.example/hallucination-burn"
        annotations:
          summary: "Hallucination fast burn: 2% of 28-day budget in 1h"
          description: "Eval judge grounded-rate below 97% SLO. Page eval-peer rotation per policy §2.6."

      # Medium: 6 * 0.03 = 0.18
      - alert: HallucinationMediumBurn
        expr: |
          (hallucination:slo_error_rate:ratio_rate6h > (6 * (1 - 0.97)))
          and
          (hallucination:slo_error_rate:ratio_rate30m > (6 * (1 - 0.97)))
        for: 5m
        labels:
          severity: page
          slo: hallucination
          surface: chat-stream
          dimension: quality
          co_owner: eval-peer
          runbook: "https://runbooks.acme.example/hallucination-burn"
        annotations:
          summary: "Hallucination medium burn: 5% of 28-day budget in 6h"

      # Slow: 1 * 0.03 = 0.03
      - alert: HallucinationSlowBurn
        expr: |
          (hallucination:slo_error_rate:ratio_rate3d > (1 * (1 - 0.97)))
          and
          (hallucination:slo_error_rate:ratio_rate6h > (1 * (1 - 0.97)))
        for: 30m
        labels:
          severity: ticket
          slo: hallucination
          surface: chat-stream
          dimension: quality
          co_owner: eval-peer
          runbook: "https://runbooks.acme.example/hallucination-burn"
        annotations:
          summary: "Hallucination slow burn: 10% of 28-day budget in 3d"

  - name: acme-llm-gw.partial_success
    interval: 30s
    rules:
      # SLO: partial-success >= 99%. Target = 99%
      # Fast: 14.4 * 0.01 = 0.144
      - alert: PartialSuccessFastBurn
        expr: |
          (partial_success:slo_error_rate:ratio_rate1h > (14.4 * (1 - 0.99)))
          and
          (partial_success:slo_error_rate:ratio_rate5m > (14.4 * (1 - 0.99)))
        for: 2m
        labels:
          severity: page
          slo: partial_success
          surface: chat-stream
          dimension: availability
          runbook: "https://runbooks.acme.example/partial-success-burn"
        annotations:
          summary: "Partial-success fast burn: 2% of 28-day budget in 1h"
          description: "Streams opened but not completing to server_done. Likely mid-stream server crash or stall."

      - alert: PartialSuccessMediumBurn
        expr: |
          (partial_success:slo_error_rate:ratio_rate6h > (6 * (1 - 0.99)))
          and
          (partial_success:slo_error_rate:ratio_rate30m > (6 * (1 - 0.99)))
        for: 5m
        labels:
          severity: page
          slo: partial_success
          surface: chat-stream
          dimension: availability
          runbook: "https://runbooks.acme.example/partial-success-burn"
        annotations:
          summary: "Partial-success medium burn: 5% of 28-day budget in 6h"

      - alert: PartialSuccessSlowBurn
        expr: |
          (partial_success:slo_error_rate:ratio_rate3d > (1 * (1 - 0.99)))
          and
          (partial_success:slo_error_rate:ratio_rate6h > (1 * (1 - 0.99)))
        for: 30m
        labels:
          severity: ticket
          slo: partial_success
          surface: chat-stream
          dimension: availability
          runbook: "https://runbooks.acme.example/partial-success-burn"
        annotations:
          summary: "Partial-success slow burn: 10% of 28-day budget in 3d"

  - name: acme-llm-gw.structured_output_conformance
    interval: 30s
    rules:
      # SLO: structured-output-conformance >= 99.5%. Target = 99.5%
      # Fast: 14.4 * 0.005 = 0.072
      - alert: SchemaConformanceFastBurn
        expr: |
          (schema_conformance:slo_error_rate:ratio_rate1h > (14.4 * (1 - 0.995)))
          and
          (schema_conformance:slo_error_rate:ratio_rate5m > (14.4 * (1 - 0.995)))
        for: 2m
        labels:
          severity: page
          slo: schema_conformance
          surface: code-assist
          dimension: quality
          runbook: "https://runbooks.acme.example/schema-conformance-burn"
        annotations:
          summary: "Structured-output conformance fast burn"
          description: "code-assist JSON validation pass-rate below 99.5%. Likely constrained-decoding regression or provider version change."

      - alert: SchemaConformanceMediumBurn
        expr: |
          (schema_conformance:slo_error_rate:ratio_rate6h > (6 * (1 - 0.995)))
          and
          (schema_conformance:slo_error_rate:ratio_rate30m > (6 * (1 - 0.995)))
        for: 5m
        labels:
          severity: page
          slo: schema_conformance
          surface: code-assist
          dimension: quality
          runbook: "https://runbooks.acme.example/schema-conformance-burn"
        annotations:
          summary: "Structured-output conformance medium burn"

      - alert: SchemaConformanceSlowBurn
        expr: |
          (schema_conformance:slo_error_rate:ratio_rate3d > (1 * (1 - 0.995)))
          and
          (schema_conformance:slo_error_rate:ratio_rate6h > (1 * (1 - 0.995)))
        for: 30m
        labels:
          severity: ticket
          slo: schema_conformance
          surface: code-assist
          dimension: quality
          runbook: "https://runbooks.acme.example/schema-conformance-burn"
        annotations:
          summary: "Structured-output conformance slow burn"

  - name: acme-llm-gw.cost_per_million_output
    interval: 5m
    rules:
      # Cost SLI. Fast-burn omitted per exercise-04 policy.
      # Target ceiling: $120 / 1M output tokens.
      # Medium (6x): $120 * 6 = $720. Threshold: 6h average > $720/1M.
      - alert: CostPerMOutputMediumBurn
        expr: |
          (cost_per_m_output:rate6h > (6 * 120))
          and
          (cost_per_m_output:rate30m > (6 * 120))
        for: 15m
        labels:
          severity: ticket
          slo: cost_per_m_output
          surface: chat-stream
          dimension: cost
          runbook: "https://runbooks.acme.example/cost-per-m-output-burn"
        annotations:
          summary: "Cost per 1M output tokens medium burn (ticket, not page)"
          description: "6h observed cost-per-1M exceeds 6x SLO ceiling. Investigate model-mix shift and traffic-mix drift."

      - alert: CostPerMOutputSlowBurn
        expr: |
          (cost_per_m_output:rate3d > (1 * 120))
          and
          (cost_per_m_output:rate6h > (1 * 120))
        for: 2h
        labels:
          severity: ticket
          slo: cost_per_m_output
          surface: chat-stream
          dimension: cost
          runbook: "https://runbooks.acme.example/cost-per-m-output-burn"
        annotations:
          summary: "Cost per 1M output tokens slow burn (ticket; route to SLO review)"

      # Correlated-paging exception per exercise-04 §2.4.
      - alert: CostPerMOutputCorrelatedAbuse
        expr: |
          (cost_per_m_output:rate1h > (6 * 120))
          and
          (toxic_output:slo_error_rate:ratio_rate1h > (14.4 * (1 - 0.999)))
        for: 2m
        labels:
          severity: page
          slo: cost_per_m_output
          surface: chat-stream
          dimension: cost
          co_owner: security-peer
          runbook: "https://runbooks.acme.example/cost-abuse-correlated"
        annotations:
          summary: "Cost burn correlated with toxicity burn — abuse signature. Page SRE + security."
```

### Three-minute decision ladder for the primary on-call

Given the policy above, a new primary following the ladder from the
chapter resolves in under three minutes:

1. **SLI** — read the alert label `slo:` to identify which SLI
   fired.
2. **Band** — read the SLI's budget-remaining from the SLO
   dashboard. Compare against §2.4 bands.
3. **Correlation** — check the correlated-burn dashboard
   (`correlated-burn` panel) — any other SLI fast-burning within
   the 2-hour window?  If yes, IC escalation per §2.5.
4. **Dimension** — latency / quality / availability → freeze map
   in §2.2 applies; cost → does-not-freeze.
5. **Permitted actions** — consult §2.4 band × §2.2 freeze row.
   Execute rollback / capacity call / canary pause as named.

This ladder is the artifact. A reader should be able to resolve
"hallucination rate burning at 40% budget, no correlated burn" to
*"band = 50–25% (caution); dimension = quality; freeze: prompt-
template changes and model promotions; permitted action: run the
hallucination-burn runbook; eval peer notified per §2.6"* without
asking.

---

## 3. Validation steps

1. All eight policy sections present with no TBDs.
2. Every SLI appears in the §2.2 freeze map; cost rows say
   "does not freeze" explicitly.
3. §2.4 band table has specific numbers and actions in each row.
4. §2.5 names a correlation window (2 h) and justifies it.
5. §2.6 states both rules concretely; eval-peer channel named.
6. Part B has three rules (minus cost fast-burn) for each of
   five SLIs across ≥ 3 dimensions.
7. Every alert has a `runbook:` label and an SLO-recomputed
   threshold in the expression.
8. Thresholds visibly differ per SLO: `(14.4 * (1-0.95))` vs
   `(14.4 * (1-0.97))` vs `(14.4 * (1-0.99))` vs
   `(14.4 * (1-0.995))` vs `(14.4 * (1-0.999))` — five different
   SLOs, five different expressions. Spot-check that cross-
   contamination did not occur.
9. YAML parses (two-space indent; valid key order; `expr:` /
   `for:` / `labels:` / `annotations:` structure matches
   Prometheus alerting-rules spec).
10. Three-minute ladder resolves one hypothetical burn without
    further lookup.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | All 8 policy sections populated | No TBD, no placeholder | Any section stubbed |
| R2 | Freeze map covers every SLI incl. cost | Cost rows read "does not freeze" | Cost rows missing or ambiguous |
| R3 | Burn-band table operable | Specific numbers / actions per row | "Reduce canary allocation" |
| R4 | Correlated-burn rule names window + path | 2 h window stated and defended; IC path and peer paging explicit | Rule vague |
| R5 | Eval-peer coordination rules concrete | Both rules stated; channel named | Hidden dependency |
| R6 | Alert YAML parses | Rules follow Prometheus spec | Syntax errors |
| R7 | Thresholds recomputed per SLO | Expressions differ visibly across 5 SLOs | One threshold used across the stack |
| R8 | Every alert has a runbook URL | Runbook label on every rule | Alerts without runbook |
| R9 | Three-minute ladder visible | Policy admits a 3-min resolution on a hypothetical | Reader still searching after 3 min |

---

## 5. Common mistakes

- **Copy-pasting one burn-rate threshold across SLOs.** Fastest
  way to a broken alerting system. The multiplier is constant,
  the base `(1 − SLO)` is per-SLO.
- **Omitting cost from the freeze map.** Cost SLIs do not freeze,
  but the freeze map must *say so*. Missing rows get read as
  "default-frozen."
- **Vague burn-band actions.** "Reduce canary allocation" is a
  design discussion; "at most one canary deploy per week" is an
  operating rule.
- **Correlation window chosen without defence.** 2 hours is the
  SRE Workbook default; a different number is fine but must be
  defended against the slow-correlated and unrelated-burn
  failure modes.
- **Single-window burn-rate alerts.** The two-window guard
  (long window `and` short window) is non-negotiable; a
  single-window alert at `14.4 × budget` fires on transient
  minute-scale noise.
- **Skipping the eval-peer coordination rules.** The two rules
  are the whole bridge between the eval release gate and the
  SRE SLO surface; without them the gate is advisory and the SLO
  is unenforceable.
- **Forgetting the correlated-paging cost exception.** Cost burn
  that correlates with a safety SLI is the abuse signature; a
  policy that treats it as ticket-only misses the escalation.

---

## 6. References

- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for the
  multi-window / multi-burn-rate pattern, the 14.4× / 6× / 1×
  multipliers, and the two-window guard structure used in Part B.
- Google SRE Workbook, *Implementing SLOs* —
  <https://sre.google/workbook/implementing-slos/>. Source for
  the error-budget policy structure replicated in Part A.
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the freeze-on-burn discipline codified in §2.2.
- Prometheus alerting-rules reference —
  <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>.
  Source for the YAML shape and `expr:` / `for:` / `labels:` /
  `annotations:` convention used in Part B.
- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source
  for the attributes referenced indirectly through the recording
  rules.
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the abuse-correlated paging framing in §2.5 and in
  `CostPerMOutputCorrelatedAbuse`.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>. Source
  for the governance framing of the exception process in §2.7.
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-05-error-budget-policy-and-burn-alert-authoring.md`
  (learning exercise this solution answers) plus exercises 01–04
  SOLUTION.md (the stack this policy governs).
