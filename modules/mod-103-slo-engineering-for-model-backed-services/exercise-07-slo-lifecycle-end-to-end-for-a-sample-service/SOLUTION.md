# Exercise 07 — SLO Lifecycle End-to-End for a Sample Service — Reference Solution

> The learning-side exercise file for this entry is a scaffolded
> placeholder (TBD content); this solution answers the module-level
> learning objective "Build an SLO lifecycle — define, baseline,
> instrument, alert, review, retire — and tie each step to a
> Prometheus / Grafana / OTel artifact" using the `acme-llm-gw`
> service carried through exercises 01–06.

## 1. Solution overview

Walk the full SLO lifecycle for **`acme-llm-gw`** — the illustrative
LLM gateway with three surfaces (`chat-stream`, `code-assist`,
`batch-summarize`) established in exercise 01 — across the six
lifecycle phases:

1. **Define** — pick SLIs per workload class (exercise 01 + 02 + 03 +
   04 artifacts).
2. **Baseline** — measure production traffic before setting
   targets.
3. **Instrument** — wire OTel attributes, Prometheus recording
   rules, Grafana panels.
4. **Alert** — Prometheus burn-rate rules from exercise 05.
5. **Review** — monthly ritual from exercise 06.
6. **Retire** — when a surface exits scope.

The point of exercise 07 is the *end-to-end*. Each phase has a
concrete artifact; each artifact has a dependency on the prior
phase. A reviewer should see the lifecycle as a loop, not a
one-shot, with the review ritual re-entering phase 1 on target-
adjustment or new-SLI motions.

---

## 2. Worked answer / Implementation

### 2.1 Phase 1 — Define

Artifacts produced in exercises 01–04 for `acme-llm-gw`:

- Latency SLI matrix (exercise 01): 10 SLIs across three surfaces.
- Quality SLI block (exercise 02): six base SLIs, eight rows after
  splitting refusal into warranted / over-refusal.
- Streaming-availability block (exercise 03): four SLIs
  (protocol, partial-success, graceful-degradation, streaming-
  stall-aliased).
- Cost SLI block (exercise 04): four SLIs with derivation chain
  back to the business unit-economics model.

Total: 23 SLIs managed in the composed stack. Composite ceiling
~47.8% ("all SLIs green over 28 days," worst-case correlated — see
exercise 02 §5).

Define-phase exit criteria:

- Every SLI has a definition, a valid-event filter, a target shape
  (target plus percentile or threshold), a window, a dependencies-
  list (eval peer, security peer, capacity owner where
  applicable), a reviewer-readable one-line justification.
- Every N/A entry is explicit with a one-line reason; dense
  matrices are rejected in favour of defended sparsity.

### 2.2 Phase 2 — Baseline

Before any target is committed, each SLI is baselined against 28
days of production traffic. Baseline methodology (per `mod-103`
chapter 06):

1. **Collect raw traffic** for the valid-event-filter-eligible
   requests. For latency SLIs: `gen_ai.server.*` spans. For
   quality SLIs: judge-pipeline sampled spans. For cost SLIs:
   provider billing feed ingested per exercise 04 §2.5.
2. **Compute the empirical percentile / rate** over the 28-day
   window. For `chat-stream` TTFT cache-miss, the baseline
   measurement is p95 TTFT on the subset with `cache.prefix.hit
   = false`; for hallucination, the baseline is the sample mean
   grounded-rate with the `eval.unknown_rate` reported alongside.
3. **Report back to the sponsor** as a baseline document: SLI,
   measured baseline, proposed target, headroom factor (target
   is baseline ± headroom, defended).
4. **Target-setting is a trade** — baseline minus headroom in one
   direction vs. aspirational-but-measurable in the other. For
   `chat-stream` TTFT cache-miss, baseline ≈ 1.1 s and target
   p95 ≤ 1.2 s leaves ~10% headroom. For hallucination, baseline
   96.8% (from the exercise-06 dry-run) argues for a 95%–96%
   target initially; the 97% target chosen in exercise 02 is
   defended against a planned retrieval upgrade (tightened to
   97.5% in the 2026-11 review).

Baseline-phase artifact: `ops/slo-baselines/<sli>-2026-<month>.md`
per SLI, with the measurement script (PromQL or judge-pipeline
query), the raw 28-day numbers, and the headroom defence.

### 2.3 Phase 3 — Instrument

Each SLI consumes an attribute surface. The instrumentation layer
is where this gets wired.

**OTel Semantic Conventions used (Generative AI) —
<https://opentelemetry.io/docs/specs/semconv/gen-ai/>.**

- Request-shape: `gen_ai.request.model`, `gen_ai.request.max_tokens`,
  `gen_ai.request.temperature`.
- Response-shape: `gen_ai.response.model`, `gen_ai.response.finish_reasons`.
- Token usage: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`,
  `gen_ai.usage.total_tokens`.
- Timing: `gen_ai.server.time_to_first_token`,
  `gen_ai.server.time_per_output_token` (derived; module convention).
- System: `gen_ai.system` (provider name), `gen_ai.operation.name`.

**Module-extension attributes (`slo.*`).**  Per chapter 06 §4.

- `slo.surface`: `chat-stream` / `code-assist` / `batch-summarize`.
- `slo.workload_class`: `chat` / `tool-calling` / `batch`.
- `slo.cache.hit`: boolean on each request.
- `slo.fallback.path`: empty / `provider-secondary` / `model-smaller`
  / `retrieval-unavailable` / `guardrail-fail-open` /
  `cached-response`.
- `slo.stream.terminated_by`: `server_done` / `server_error` /
  `client_cancelled` — gap flagged in exercise 03 until this ships.
- `slo.session.id`: propagated session identifier — gap flagged
  in exercise 04 until this ships.
- `slo.eval.judge_verdict`, `slo.eval.judge_version`,
  `slo.eval.unknown_rate` — emitted by the eval pipeline on
  judged spans.

**Prometheus recording rules.**  One per SLI; computes the SLI's
rate / percentile over the 28-day window.

```yaml
# recording-rules/acme-llm-gw.yml (excerpt)
groups:
  - name: acme-llm-gw.recording
    interval: 30s
    rules:
      # Latency: TTFT cache-miss p95 over 28 days.
      - record: ttft_cache_miss:slo_error_rate:ratio_rate28d
        expr: |
          1 - (
            sum(rate(gen_ai_server_time_to_first_token_bucket{slo_surface="chat-stream", slo_cache_hit="false", le="1.2"}[28d]))
            /
            sum(rate(gen_ai_server_time_to_first_token_count{slo_surface="chat-stream", slo_cache_hit="false"}[28d]))
          )

      # Multiple short windows for the two-window burn-rate guard.
      - record: ttft_cache_miss:slo_error_rate:ratio_rate1h
        expr: |
          1 - (
            sum(rate(gen_ai_server_time_to_first_token_bucket{slo_surface="chat-stream", slo_cache_hit="false", le="1.2"}[1h]))
            /
            sum(rate(gen_ai_server_time_to_first_token_count{slo_surface="chat-stream", slo_cache_hit="false"}[1h]))
          )
      - record: ttft_cache_miss:slo_error_rate:ratio_rate5m
        expr: |
          1 - (
            sum(rate(gen_ai_server_time_to_first_token_bucket{slo_surface="chat-stream", slo_cache_hit="false", le="1.2"}[5m]))
            /
            sum(rate(gen_ai_server_time_to_first_token_count{slo_surface="chat-stream", slo_cache_hit="false"}[5m]))
          )
      # Analogous :ratio_rate6h / :ratio_rate30m / :ratio_rate3d / :ratio_rate6h entries.

      # Quality: hallucination grounded-rate over 28 days.
      - record: hallucination:slo_error_rate:ratio_rate28d
        expr: |
          1 - (
            sum(rate(eval_judge_grounded_total{slo_surface="chat-stream"}[28d]))
            /
            sum(rate(eval_judge_total{slo_surface="chat-stream"}[28d]))
          )
      # Analogous short-window entries.

      # Cost: $ per 1M output tokens over a 6-hour window.
      - record: cost_per_m_output:rate6h
        expr: |
          1e6 * (
            sum(rate(provider_billing_cost_usd{slo_surface="chat-stream", token_type="output"}[6h]))
            /
            sum(rate(gen_ai_usage_output_tokens{slo_surface="chat-stream"}[6h]))
          )
      # Analogous :rate30m / :rate3d / :rate1h entries.
```

**Grafana panels.**  One SLO-overview dashboard, one per-SLI
drill-down. Panel conventions:

- Top-of-dashboard: per-SLI current-28-day value, target, budget
  remaining, trajectory sparkline.
- Correlated-burn panel (chapter 06): matrix of SLIs × 2-hour
  windows, cells coloured by fast / medium / slow burn state. The
  IC consults this during the correlated-burn runbook (exercise
  05 §2.5).
- Fallback-path breakout panel: per-path graceful-degradation
  sub-SLI rates, live. The aggregate SLI is a headline; the sub-
  SLIs are the diagnostic.

**Instrumentation-phase exit criteria.**

- Every SLI has recording rules in `recording-rules/`.
- Every SLI has a Grafana panel linked from the SLO overview
  dashboard.
- Every SLI has its OTel attribute surface documented in the
  SLI-spec file (`ops/sli-specs/<sli>.md`).
- Instrumentation gaps blocking any SLI are logged with ticket
  IDs and owners; dependent SLIs are in warn-only mode.

### 2.4 Phase 4 — Alert

Artifacts from exercise 05: Prometheus burn-rate alert rules, one
group per SLI, three rules each (fast / medium / slow) except cost
(two — fast omitted per the ticket-only policy with correlated-
paging exceptions). See exercise 05 Part B for the YAML. Alert-
phase exit criteria:

- Every alert has a runbook URL label.
- Thresholds are recomputed per SLO target.
- The two-window guard pattern is used.
- Alertmanager routes are configured: `severity=page` → PagerDuty
  primary rotation, `severity=ticket` → the SRE queue in Jira /
  Linear, `co_owner=eval-peer` → eval peer's on-call channel,
  `co_owner=security-peer` → security peer's on-call channel.

### 2.5 Phase 5 — Review

Artifact: the monthly facilitation worksheet from exercise 06
plus the monthly minutes archive at
`ops/slo-review/minutes/YYYY-MM-DD.md`. Review-phase exit criteria:

- Every month: a worksheet is pre-filled, the review runs 30 min,
  minutes are published within 24 hours.
- Every quarter: the allocation split (exercise 05 §2.3) is
  revisited.
- Annually: the full policy is re-read against the past year's
  incidents and exceptions, with an external reader from a
  sibling team.
- Recurring patterns in the exception log (same shape ≥ 2× per
  quarter) force policy motions that absorb the pattern.

The review phase re-enters phase 1 whenever a new-SLI proposal
or target-adjustment motion carries — the loop is explicit.

### 2.6 Phase 6 — Retire

When a surface exits scope (product sunset, consolidation, re-
architecture that reshapes the SLI definitions), the retirement
checklist from chapter 06 §7:

- [ ] All alerts for the retired SLI are removed from Prometheus.
  Confirmed by grepping the alert config for the SLI name.
- [ ] All Grafana panels for the retired SLI are archived
  (moved to an `archive/` folder with the retirement date in
  the filename).
- [ ] The 28-day baseline at retirement is archived to
  `ops/slo-baselines/retired/<sli>-<date>.md` so future
  comparisons have a reference point.
- [ ] The retirement is recorded in the policy change-log
  (`ops/policy-change-log.md`) with a one-paragraph rationale.
- [ ] The eval peer and any co-owners are notified of the SLI
  retirement (removes their on-call liability for that SLI).

Retirement is a *review-driven* motion — chair raises it under
exercise 06 §5, attendees ratify, chair owns the retirement-
checklist execution between reviews. A unilateral SLI retirement
by on-call is treated as an exception and goes to the next
review for sign-off.

### 2.7 End-to-end dependency graph

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1 Define     → SLI spec file  ────────────────┐           │
│                      (ops/sli-specs/<sli>.md)       │           │
│                                                      ▼          │
│ Phase 2 Baseline   → baseline report       → target defended   │
│                      (ops/slo-baselines/...)         │          │
│                                                      ▼          │
│ Phase 3 Instrument → OTel attributes               + Prometheus │
│                      recording rules   + Grafana panels         │
│                                                      │          │
│                                                      ▼          │
│ Phase 4 Alert      → Prometheus burn-rate rules +               │
│                      runbook URL per alert                      │
│                                                      │          │
│                                                      ▼          │
│ Phase 5 Review     → monthly worksheet + minutes archive +      │
│                      exception log + policy change-log          │
│                       │                                         │
│                       └──re-enters Phase 1 on target-change──┐  │
│                                                               │  │
│ Phase 6 Retire     → retirement checklist complete → SLI off  │  │
│                      the stack; baseline archived; alerts      │  │
│                      removed; owners notified                  │  │
└───────────────────────────────────────────────────────────────┘
```

### 2.8 Anti-patterns prevented by the lifecycle

- **SLI without baseline** — phase 2 forces a measurement before a
  target is committed; targets set by intuition break under first
  sponsor challenge.
- **SLI without instrumentation** — phase 3 exit criteria require
  attribute coverage; a target without an instrument is a wish.
- **Alert without runbook** — phase 4 enforces runbook URLs on
  every rule; alerts without runbooks produce noise that does not
  resolve.
- **SLI without owner review** — phase 5 catches owner-abandoned
  SLIs (no one presents them at the monthly review → they are
  candidates for retirement).
- **Zombie SLIs after surface retirement** — phase 6 checklist
  catches alerts left firing after the surface is gone; zombie
  alerts corrode on-call trust in the overall alert surface.

---

## 3. Validation steps

1. All six phases named with artifacts: SLI spec, baseline
   report, recording rules + panels + attributes, alert rules,
   worksheet + minutes, retirement checklist.
2. Each phase has a named exit criterion.
3. The loop from phase 5 back to phase 1 is explicit.
4. Instrumentation dependencies (OTel attributes, Prometheus
   recording rules, Grafana panels) map to exercises 01–04 SLIs.
5. Alert artifacts from exercise 05 are referenced (not
   duplicated) and the two-window guard pattern is preserved.
6. Retirement checklist includes alert removal, panel archive,
   baseline archive, policy change-log entry, and co-owner
   notification.
7. Anti-patterns section names at least five lifecycle-prevented
   failure modes.
8. Reviewer can trace one SLI (pick any, e.g., hallucination)
   through all six phases and find the artifact per phase.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Six phases enumerated and sequenced | Define → Baseline → Instrument → Alert → Review → Retire, with loop from Review back to Define | Phase missing; loop absent |
| R2 | Artifact per phase | Named file / config block per phase | Phase stated without an artifact |
| R3 | Exit criteria per phase | Each phase has named gates to leave it | Pass-through phases without criteria |
| R4 | OTel attribute surface documented | `gen_ai.*` and `slo.*` attributes enumerated | Attributes mentioned generically |
| R5 | Recording rules shown | Prometheus recording rules for a representative SLI (not just the alert rules) | Only alert rules shown |
| R6 | Retirement checklist operable | Five items checklist-shaped; each actionable | "Retire the SLI" one-line |
| R7 | Lifecycle prevents named anti-patterns | Five anti-patterns traced to lifecycle phases | Anti-patterns listed in abstract |
| R8 | Reviewer traces one SLI end-to-end | Hallucination (or another) found in each phase | SLI appears in define only |

---

## 5. Common mistakes

- **Treating the lifecycle as one-shot.** Phase 5 re-enters phase
  1 on target changes and new-SLI motions; a lifecycle
  presentation that does not show the loop misses the discipline.
- **Skipping the baseline phase.** Setting a target before a
  measurement is a cite-your-source failure; the sponsor's first
  question is "where did that number come from?"
- **Over-engineering the instrumentation before baselining.**
  Attributes that no SLI consumes are instrumentation debt; the
  phase order (baseline before instrument) prevents this.
- **Alerts without runbook URLs.** The alert label is cheap; the
  runbook URL is the artifact that converts a page into an
  action. Phase 4 exit requires both.
- **Monthly reviews without a chair.** The ritual needs a
  designated chair per quarter; without one, timing discipline
  and recurring-pattern catch (exercise 06) both erode.
- **Retirement without baseline archive.** The baseline at
  retirement is what future comparisons use ("has the latency
  surface we retired been replaced with something better?");
  losing it is cheap now and expensive later.

---

## 6. References

- Google SRE Workbook, *Implementing SLOs* —
  <https://sre.google/workbook/implementing-slos/>. Primary
  source for the lifecycle phases in §2.1–§2.6.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for the
  phase 4 alerting discipline (two-window guard; runbook URL on
  every rule).
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the valid-event-filter / target-defence discipline in
  phases 1–2.
- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source
  for the attribute surface in §2.3.
- Prometheus recording-rules reference —
  <https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/>.
  Source for the YAML shape in §2.3.
- Prometheus alerting-rules reference —
  <https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>.
  Source for the alert-rules YAML referenced in §2.4.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>. Source
  for the governance framing in phases 5 and 6 (review and
  retirement are *Manage*-function activities).
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the safety-peer co-owner convention in phase 4
  alert labels and phase 6 retirement notification.
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced for the
  threat framing underlying the correlated-paging exception in
  phase 4 (exercise 04 §2.4).
- Local context: all seven exercise prompts under
  `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/`
  plus the companion SOLUTION.md files for exercises 01–06.
