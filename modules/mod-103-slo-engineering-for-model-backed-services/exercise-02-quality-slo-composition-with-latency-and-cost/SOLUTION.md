# Exercise 02 — Quality SLO Composition with Latency and Cost — Reference Solution

## 1. Solution overview

Reference quality / composed-stack artifact for the **`acme-llm-gw`**
workload established in exercise 01 (three surfaces: `chat-stream`,
`code-assist`, `batch-summarize`). This document:

1. Adds the six quality SLIs — hallucination, toxic-output, tool-call-
   success, structured-output-conformance, refusal (two-sided),
   format-drift — to the SLI surface.
2. Attaches the eval-peer handoff paragraph per quality SLI so the
   signal source is nameable at 03:00.
3. Composes latency, quality, and cost-named-only into a single
   stack with a sum-of-shortfalls ceiling.
4. Walks two correlated-burn scenarios to show the stack admits
   sensible on-call reconciliation (full policy is exercise 05).

All numbers are illustrative; the shape of the derivation is the
load-bearing part. The eval peer is referenced as
`eval-pipeline-v1` (placeholder — the dependency is flagged as
"to be stood up alongside the SRE-side work" where applicable).

---

## 2. Worked answer / Implementation

### 2.1 Section 1 — The quality SLI table

| Quality SLI | SLI definition | Target (28-day) | Window | Sampling methodology | Burn-rate thresholds |
|---|---|---|---|---|---|
| Hallucination | Fraction of `chat-stream` turns where the judge model flags a factual-ungrounded claim against the retrieval context, among valid-for-judge turns | ≥ 97% grounded | 28-day rolling | 2% random sample of `chat-stream` turns judged by `eval-pipeline-v1`; 100% sample on turns tagged `retrieval=true` | fast ≥ 43.2% / medium ≥ 18% / slow ≥ 3% |
| Toxic-output | Fraction of `chat-stream` responses the moderation classifier flags as safe-policy-compliant | ≥ 99.9% compliant | 28-day rolling | 100% coverage (rule-based classifier, cheap) | fast ≥ 1.44% / medium ≥ 0.6% / slow ≥ 0.1% |
| Tool-call-success | Fraction of `code-assist` invocations where (a) the response is schema-valid, (b) the named tool exists, and (c) the tool-side executor returned non-error for the issued call | ≥ 99% successful | 28-day rolling | 100% coverage (deterministic) | fast ≥ 14.4% / medium ≥ 6% / slow ≥ 1% |
| Structured-output-conformance | Fraction of `code-assist` responses that pass JSON-schema validation against the declared tool signature | ≥ 99.5% conformant | 28-day rolling | 100% coverage (deterministic) | fast ≥ 7.2% / medium ≥ 3% / slow ≥ 0.5% |
| Refusal — **warranted** | Fraction of prompts the moderation policy classifies as requiring refusal that are in fact refused by the model | ≥ 99% refused when warranted | 28-day rolling | 10% sample of flagged-as-sensitive turns judged by `eval-pipeline-v1` plus 100% replay of the `eval-sensitive-v1` fixture set | fast ≥ 14.4% / medium ≥ 6% / slow ≥ 1% |
| Refusal — **over-refusal** | Fraction of `eval-benign-v1` fixture prompts that are *not* refused | ≥ 98% not refused on benign prompts | 28-day rolling | 100% replay of `eval-benign-v1` fixture set (eval peer owns) | fast ≥ 28.8% / medium ≥ 12% / slow ≥ 2% |
| Format-drift | On `code-assist`, fraction of responses whose field-ordering and type-shape match the recorded production distribution | N/A — warn-only on `chat-stream`; on `code-assist` folded into structured-output-conformance | — | — | — |

Notes.

- Refusal is **two-sided** by construction. Over-refusal is a user-
  experience failure (helpful requests denied) and is judged against
  a specific benign fixture set `eval-benign-v1`; warranted-refusal
  is a safety failure judged against sensitive-prompt fixtures.
- Format-drift is downgraded to **N/A** on `chat-stream` because the
  consuming UI is free-form text that re-formats; on `code-assist`
  it is subsumed by structured-output-conformance (any drift
  produces a conformance failure).
- Burn-rate multipliers are the SRE Workbook defaults (14.4× / 6×
  / 1× over 1 h / 6 h / 3 d). Absolute thresholds are
  recomputed per SLO: at 97% the fast-burn threshold is
  `14.4 × 3% = 43.2%`, at 99.9% it is `14.4 × 0.1% = 1.44%`.

### 2.2 Section 2 — Eval-peer handoff paragraph per quality SLI

- **Hallucination.** Signal source is `eval-pipeline-v1/judge/
  grounded-response-v3`, a judge-model pipeline owned by the
  ai-eval-engineer peer. Attribute surface: `eval.judge.verdict`,
  `eval.judge.model`, `eval.judge.version`, `eval.sample_id`,
  `eval.unknown_rate` emitted on every judged span. Version-bump
  coordination: a judge-version change is announced ≥ two weeks
  before cutover in the `#eval-releases` channel and triggers a
  re-baseline of the hallucination SLO at the next monthly review
  (exercise 06). Dependency: `eval-pipeline-v1` is placeholder —
  to be stood up alongside the SRE-side work.
- **Toxic-output.** Signal source is a rule-based moderation
  classifier `moderation-v4` embedded in the gateway, not a judge
  model. Owned jointly by eval and the ai-infra-security peer.
  Attribute: `safety.moderation.flag`. A classifier-version change
  is treated as a canary (chapter 05 release gate) and reviewed
  before promotion. 100% coverage (cheap, deterministic).
- **Tool-call-success.** Signal source is deterministic, generated
  by the gateway at response time (schema check + tool existence)
  plus a signal from the tool-side executor (`tool.exec.status`).
  No judge model. Owned by SRE jointly with the tool-platform
  peer; a schema-registry change triggers re-validation of the
  SLO target (the acceptable rate changes with the tool catalog).
- **Structured-output-conformance.** Signal source is deterministic
  JSON-schema validation at the gateway (`schema.validation.pass`).
  Owned entirely by SRE; no eval-peer dependency. 100% coverage.
- **Refusal (warranted).** Signal source is a hybrid — judge model
  on the 10% sample plus 100% replay of the `eval-sensitive-v1`
  fixture set. Eval peer owns the fixture set version; a fixture
  version change triggers re-baseline. Attribute surface:
  `eval.refusal.verdict`, `eval.fixture.version`.
- **Refusal (over-refusal).** Signal source is 100% replay of the
  `eval-benign-v1` fixture set at a nightly cadence. Eval peer
  owns; the fixture set is version-pinned per SLO window. Dependency
  flagged as "to be stood up" if `eval-benign-v1` does not yet
  exist.

### 2.3 Section 3 — The composed stack

| Dimension | SLI | Target | 28-day shortfall | Window | Burn-rate thresholds |
|---|---|---|---|---|---|
| Latency | `chat-stream` TTFT cache-miss | p95 ≤ 1.2 s | 5% | 28-day | 72% / 30% / 5% |
| Latency | `chat-stream` TTFT cache-hit | p95 ≤ 400 ms | 5% | 28-day | 72% / 30% / 5% |
| Latency | `chat-stream` TPOT | p95 ≤ 80 ms / tok | 5% | 28-day | 72% / 30% / 5% |
| Latency | `chat-stream` ITL | p99 ≤ 2 s | 1% | 28-day | 14.4% / 6% / 1% |
| Latency | `chat-stream` E2E | p95 ≤ 10 s | 5% | 28-day | 72% / 30% / 5% |
| Latency | `chat-stream` goodput | p90 ≥ 12 tok/s | 10% | 28-day | capped 50% / 60% / 10% |
| Latency | `code-assist` TTFT | p95 ≤ 700 ms | 5% | 28-day | 72% / 30% / 5% |
| Latency | `code-assist` E2E (schema-valid) | p95 ≤ 2 s | 5% | 28-day | 72% / 30% / 5% |
| Latency | `batch-summarize` E2E (per-ticket) | p99 ≤ 180 s | 1% | 28-day | 14.4% / 6% / 1% |
| Latency | `batch-summarize` E2E (job) | p99 ≤ 06:00 UTC | 1% | 28-day | binary daily |
| Quality | Hallucination | ≥ 97% | 3% | 28-day | 43.2% / 18% / 3% |
| Quality | Toxic-output | ≥ 99.9% | 0.1% | 28-day | 1.44% / 0.6% / 0.1% |
| Quality | Tool-call-success | ≥ 99% | 1% | 28-day | 14.4% / 6% / 1% |
| Quality | Structured-output-conformance | ≥ 99.5% | 0.5% | 28-day | 7.2% / 3% / 0.5% |
| Quality | Refusal — warranted | ≥ 99% | 1% | 28-day | 14.4% / 6% / 1% |
| Quality | Refusal — over-refusal | ≥ 98% | 2% | 28-day | 28.8% / 12% / 2% |
| Availability (preview from ex. 03) | Protocol 2xx | ≥ 99.9% | 0.1% | 28-day | 1.44% / 0.6% / 0.1% |
| Availability (preview from ex. 03) | Partial-success | ≥ 99% | 1% | 28-day | 14.4% / 6% / 1% |
| Availability (preview from ex. 03) | Graceful-degradation (aggregate) | ≥ 99.5% on tagged fallback events | 0.5% | 28-day | 7.2% / 3% / 0.5% |
| Availability (preview from ex. 03) | Streaming-stall (ITL duplicate) | aliased from latency ITL above | — | — | — |
| Cost (preview from ex. 04) | Cost per 1M input tokens | TBD in ex. 04 | one-sided ceiling | 28-day | ticket-only |
| Cost (preview from ex. 04) | Cost per 1M output tokens | TBD in ex. 04 | one-sided ceiling | 28-day | ticket-only |
| Cost (preview from ex. 04) | Cost per completed request | TBD in ex. 04 | one-sided ceiling | 28-day | ticket-only |
| Cost (preview from ex. 04) | Cost per successful session | TBD in ex. 04 | one-sided ceiling | 28-day | ticket-only |
| **Composite** | **Sum of two-sided shortfalls** | **≈ 41.1%** (ceiling for "all SLIs green simultaneously") | — | — | — |

(Cost SLIs are one-sided ceilings and do *not* participate in the
sum-of-shortfalls; chapter 02 section 3 convention.)

### 2.4 Section 4 — The reconciliation sketch

**Scenario A — hallucination burn correlated with TTFT burn.**

On-call sees both the hallucination fast-burn (`eval-pipeline-v1`)
and the `chat-stream` TTFT cache-miss fast-burn fire within a
2-hour window.

Ordering:

1. **Declare an incident with the IC role.** Correlated-burn across
   dimensions is an IC-escalation trigger per the policy (exercise
   05 section 5). The ordering is *not* "fix latency first because
   latency is the easier burn to diagnose" — a correlated signal
   is itself the incident.
2. **Investigate the shared cause.** Common causes: a new model
   version was rolled out, retrieval index staleness, a prompt
   template change. Capacity shortfall alone would burn latency
   but not hallucination; the correlation argues against capacity
   as the first hypothesis.
3. **Rollback the most-recent gated change.** The release gate
   from `mod-101` specifies that correlated-burn in production on
   a gate-passing release still authorizes rollback by on-call
   without re-invoking the gate (exercise 05 section 6, rule 1).

**Scenario B — tool-call-success burn correlated with cost burn.**

On-call sees the `code-assist` tool-call-success SLI burning (say
80% success, well below the 99% target) and the cost-per-1M-output-
tokens ticket firing simultaneously.

Ordering:

1. **Page on quality.** Tool-call-success is a user-experience SLI
   with a page-level burn alert; cost is ticket-only.
2. **Route cost to ticket.** The IC does *not* act on the cost
   signal during the incident; it is noted in the incident channel
   for the review.
3. **Investigate the shared cause.** A likely story: a model
   provider's version update is producing malformed tool calls,
   which the gateway retries. Retries inflate the cost metric and
   depress the per-call success metric simultaneously. The quality
   SLI points to the root cause; the cost SLI is a *downstream
   consequence* and routes to post-incident cost reconciliation.

The purpose is to show the stack admits sensible ordering, not to
produce the full runbook; that is exercise 05.

### 2.5 Section 5 — The composition arithmetic

Sum-of-shortfalls ceiling (chapter 02 §3 convention):

Latency shortfalls: 5 + 5 + 5 + 1 + 5 + 10 + 5 + 5 + 1 + 1 = **43%**
(across the ten latency SLIs; goodput's 10% dominates one surface).

Quality shortfalls: 3 + 0.1 + 1 + 0.5 + 1 + 2 = **7.6%**.

Availability shortfalls (preview from exercise 03): 0.1 + 1 + 0.5 =
**1.6%** (streaming-stall aliases latency ITL, not double-counted).

**Composite ceiling = 1 − min(1, sum) = 1 − 0.522 = 47.8%.** That is
the ceiling on "all SLIs simultaneously green over a 28-day window"
under worst-case-correlated error distribution, computed
conservatively (union bound).

If the sponsor hears 47.8% and finds it low, three targets of
tightening:

- **`chat-stream` goodput** currently costs 10 points by itself.
  Tightening to p90 ≥ 15 tok/s would push some traffic to a
  smaller model; cost: a measurable quality trade-off on long-form
  replies. Defended in exercise 04 against the unit-economics
  model.
- **Hallucination** at 97% costs 3 points. Tightening to 98% is a
  ~1-point improvement but requires either a better retrieval
  index or a more conservative refusal policy (which then burns
  the over-refusal SLI). This is a visible real trade-off — the
  stack surfaces it.
- **Latency E2E** at p95 ≤ 10 s on `chat-stream` costs 5 points.
  Tightening to p99 ≤ 10 s is a shift in percentile, not
  threshold; it commits engineering capacity to the long-context
  minority and has a direct cost consequence (exercise 04).

User-experience contract from exercise 01 § 2.1 stated chat-stream
expects first token within 1 s and complete within 10 s on 95% of
turns. The composite stack *honors* that contract on the two
specific SLIs; the composite ceiling of 47.8% describes "all SLIs
simultaneously green," which is a stricter statement than the
contract. The sponsor's user-experience commitment and the engineer-
facing composite ceiling are different-shape numbers; the review
must say so explicitly.

---

## 3. Validation steps

1. Confirm the §2.1 table has eight rows (six SLIs + refusal split
   into two + format-drift present as N/A).
2. Confirm every non-N/A quality SLI has an eval-peer handoff
   paragraph in §2.2 naming pipeline, version, attributes, channel.
3. Confirm the §2.3 composed-stack table lists every SLI from the
   stack (latency ×10 + quality ×6 + availability ×3 preview +
   cost ×4 named) and ends with a sum-of-shortfalls row.
4. Confirm two reconciliation scenarios in §2.4 show ordering
   without resolving in full.
5. Confirm the §2.5 arithmetic adds up: latency 43% + quality 7.6%
   + availability 1.6% = 52.2% → composite ceiling 47.8%.
6. Confirm the eval-peer dependency is visible in §2.2 and not
   buried as a footnote.
7. Word-count check against the ≤ 2500 word acceptance criterion.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Six quality SLIs addressed; refusal split | Table has eight rows; N/As are justified in-line | Fewer than six rows; refusal not split |
| R2 | Eval-peer handoff per non-N/A quality SLI | Named pipeline, version, attribute surface, channel for each | Generic handoff paragraphs; no versions |
| R3 | Composed stack table is complete | Latency, quality, availability-preview, cost-preview rows present; sum-of-shortfalls row at bottom | Missing dimensions; no composite row |
| R4 | Two reconciliation scenarios traced | Ordering shown; not resolved; IC role and page/ticket distinction visible | Scenario resolved in full (exercise 05 overreach) or ordering unclear |
| R5 | Composition arithmetic matches targets | Arithmetic is shown step-by-step and sums to the composite | Composite stated without arithmetic; sums don't match |
| R6 | Burn-rate thresholds recomputed per SLO | 97% and 99.9% SLOs have visibly different absolute thresholds | A single generic burn-rate table reused |
| R7 | Reviewer can read the full commitment in 15 min | Dry-run with unfamiliar reviewer passes | Multiple clarifying questions needed |
| R8 | Eval-peer dependency is visible | Flagged in-line, not footnoted | Eval dependency hidden; reader cannot find the signal source |

---

## 5. Common mistakes

- **Treating refusal as one-sided.** Over-refusal and warranted-
  refusal are *different* failure modes with different users and
  different owners; a single refusal SLO is a mis-designed SLI.
- **Setting hallucination at 99.9%.** The 99.9% target is a
  commitment to a measurement precision the eval peer cannot
  deliver. Baseline-anchored targets in the 95–98% range are
  defensible; aspirational 99.9% targets fail on day one.
- **Omitting cost rows from the composed stack.** Cost SLIs are
  one-sided and do *not* participate in sum-of-shortfalls, but
  they *are* in the stack so the sponsor sees the full contract.
  Omitting them misleads the sponsor and breaks exercise 04.
- **Skipping the eval-peer handoff.** Without a named pipeline
  and version, the quality SLI is unmanaged — a prompt-level or
  model-level change to the judge *is* a change to the SLI and
  needs coordination.
- **Double-counting streaming-stall.** The streaming-stall SLI in
  exercise 03 is *aliased* from the latency ITL SLI; writing it
  as a separate SLI with its own shortfall double-counts.
- **Resolving the reconciliation scenarios.** Exercise 02 only
  needs the *ordering*. The policy (exercise 05) owns the
  resolution.
- **Confusing the composite ceiling with a user-experience
  guarantee.** The ceiling is a mathematical upper bound on "all
  simultaneously green"; it is not the SLA.

---

## 6. References

- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source for
  `eval.judge.*`, `safety.moderation.*`, `schema.validation.*`,
  `tool.exec.*` attribute surface conventions.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for the
  multi-window / multi-burn-rate thresholds used in §2.1 and §2.3.
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the sum-of-shortfalls composition discipline referenced in
  §2.5.
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the policy-compliance framing underlying the toxic-
  output SLI treatment.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>. Source
  for the measurement-governance framing in the eval-peer handoff
  paragraphs.
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-02-quality-slo-composition-with-latency-and-cost.md`
  (learning exercise this solution answers) and exercise-01
  SOLUTION.md (the latency surface this document extends).
