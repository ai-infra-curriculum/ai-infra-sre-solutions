# exercise-02 — Silent Drift Detection with Canary Eval and Output Distribution — Solution

> The learning exercise prompt is a `TBD` stub. This solution interprets the exercise against chapter `03-silent-output-drift-and-canary-evals.md` and the module's stated objective to build **detection discipline** for class 2.1. The deliverable is a design-plus-skeleton: a canary-set design document, an output-distribution-comparison panel specification, a set of three alert rules in Prometheus-style syntax, and a 15-minute first-responder checklist.

---

## 1. Solution overview

Silent output drift — chapter 01's class 2.1 — is the one failure in the taxonomy that **has no primary signal unless the on-call builds the reference first**. For every other class, a counter fires; for 2.1, a reference distribution has to be *held* somewhere and a periodic comparison has to *run*, or the drift lives undetected until the quality SLI burns.

The exercise's deliverable therefore has two tracks that must both land:

- **The reference** — a canary set and its pipeline. Frozen, versioned, scrubbed, scored against the current production model, re-runnable on a schedule.
- **The on-traffic complement** — output-distribution signals computed on live production traffic, independent of the canary. Length histogram, finish-reason distribution, first-token histogram, output-embedding centroid, (and the expensive but authoritative) judge-pass rate.

A learner's submission that only ships one of the two is incomplete: the canary alone has coverage gaps against production inputs it was not drawn from; the on-traffic signals alone have no "known-good" to anchor against. Section 3 of chapter 03 is explicit — both feed the same drift dashboard; disagreement between them is itself informative.

The three sub-classes (weight / kernel / numeric) are recognised by *which signals move together*. The canary set is sensitive to all three; the output-distribution signals disambiguate them. The design must say which signal classifies which sub-class.

## 2. Implementation — worked answer

### 2.1 Canary-set design document

The design document a learner should produce has eight sections. Numbers below are defensible starting points for a mid-sized production surface; a learner may justify alternatives.

**1. Purpose and scope.**
"The canary set is a frozen, versioned, scored reference distribution for the `<surface-name>` production path. It exists to detect silent output drift (class 2.1) against the current production model with enough signal-to-noise to classify weight / kernel / numeric sub-class within 15 minutes."

**2. Size and representativeness.** 500–2 000 examples. Stratification dimensions chosen from: task type, input length (short / medium / long), language, tenant class, surface variant. Each stratum's share is documented against a sampled week of production traffic; a `representativeness.json` captures the sampling plan so later audits can confirm the set is still representative.

**3. Construction procedure.** Per chapter 03 section 2.2:

1. Sample one week of production inputs with the stratification plan.
2. Scrub PII / PHI / secrets. Options: paraphrase, public-corpus analog, consented internal accounts. The scrubbing procedure is a signed-off checklist, not a model's judgement.
3. Freeze, version (`canary-set/v3`), check into the eval-dataset registry alongside a `README` with scope and limits.
4. Score on the current production model: capture outputs, output-embeddings, judge-model verdicts, exact-match / regex signals, output length, token counts, finish reasons.
5. Document "known-good" ranges per signal (judge-pass rate central tendency and spread; output-length mean and p95; finish-reason mix).
6. Wire the schedule.

**4. Schedule.** Hourly against the hosted endpoint; every N minutes (15 is reasonable) against a self-hosted deploy; on every deploy; on-demand from the incident channel.

**5. Tagging.** Every canary request carries `request.source = canary` (OTel attribute), `canary.set_version = v3`, `canary.example_id = <uuid>`, `canary.model_under_test = <model-label>`. No canary request counts toward user-facing SLIs; no canary request counts against the user-facing error budget or cost budget.

**6. Reference store.** Last-known-good run of the canary set stored as a Parquet file in object storage (or the eval-dataset registry's rowset). Promoted on every intentional model change; pre-promotion version retained so rollback can re-pin the comparison. Access-controlled — reference outputs can reveal prompt wording and capability patterns.

**7. Metrics and emission.** For each canary run: per-example scores → Prometheus histogram labelled by `canary_set_version`, `model_version`, slice. Aggregates → gauges. Illustrative names:

```
canary_pass_rate{model="<prod>", canary_set_version="v3", slice="<s>"}
canary_output_length_mean_delta{model="<prod>", canary_set_version="v3", slice="<s>"}
canary_output_embedding_centroid_cosine_distance{model="<prod>", canary_set_version="v3"}
canary_first_token_js_divergence{model="<prod>", canary_set_version="v3"}
```

(Exact metric names depend on your pipeline and emission conventions; chapter 02 of `mod-104` enumerates the semantic-convention attributes they must carry.)

**8. Decay plan.** Quarterly representativeness refresh; per-release reference re-capture; PII-scrub review on the same quarterly cadence; include in the quarterly SLO ritual (`mod-103` chapter 06) so a canary set that nobody looks at never ships.

### 2.2 Output-distribution-comparison panel specification

A Grafana (or equivalent) panel with the following rows, labelled by `model` / `slice`, over a 7-day window with a 30-day overlay for baseline shape:

| Row | Signal | Visualisation | Why |
|---|---|---|---|
| 1 | `rate(output_tokens_bucket[5m])` histogram → distribution plot | Stacked area or heatmap | Cheapest signal; sensitive to weight change (sub-class 1.1) via verbosity shift |
| 2 | Finish-reason mix (`gen_ai.response.finish_reasons`) — stop / length / content_filter / tool_calls | Stacked bar | Cheapest signal; the `content_filter` share also feeds the 2.4 refusal detector |
| 3 | First-token histogram KL / JS divergence against baseline window | Time-series | Cheap and surprisingly sensitive to kernel (1.2) and numeric-format (1.3) changes |
| 4 | Output-embedding centroid cosine distance against baseline centroid | Time-series | Medium cost; direct distribution-shape signal |
| 5 | Judge-pass rate per quality dimension | Time-series with burn-rate overlay | Expensive but authoritative |
| 6 | `gen_ai.response.model` label value over time | Status table | Sub-class 1.1 confirmer |
| 7 | Engine version / image tag over time | Status table | Sub-class 1.2 confirmer |
| 8 | Registry-entry quantization / calibration version over time | Status table | Sub-class 1.3 confirmer |
| 9 | Cost per request + TPOT (proxy signals from `mod-103` chapters 01 and 04) | Dual-axis time-series | Fastest leading indicator |

Deploy, template, and registry promotion markers overlay all rows (`mod-104` chapter 07 authors deploy markers).

### 2.3 Alert rules

Three alerts. The first two are per chapter 03 section 6; the third is the joint proxy-signal alert.

```yaml
# Alert A — canary-pass-rate burn (multi-window multi-burn-rate)
# fast window
- alert: CanaryPassRateFastBurn
  expr: |
    (
      avg_over_time(canary_pass_rate{canary_set_version="v3"}[15m])
      < on(canary_set_version) group_left()
      (canary_pass_rate_baseline{canary_set_version="v3"} - 0.05)
    )
    and
    (
      avg_over_time(canary_pass_rate{canary_set_version="v3"}[6h])
      < on(canary_set_version) group_left()
      (canary_pass_rate_baseline{canary_set_version="v3"} - 0.02)
    )
  for: 15m
  labels:
    severity: page
    class: 2.1
    sub_class: ambiguous
  annotations:
    summary: "Canary pass-rate burn vs. baseline (v3); candidate class 2.1 — silent output drift."
    runbook: "runbooks/drift-triage.md"

# Alert B — output-distribution shift (production traffic)
- alert: OutputTokenLengthDistributionShift
  expr: output_token_length_psi{model=~"prod-.*"} > 0.2
  for: 30m
  labels:
    severity: page
    class: 2.1
  annotations:
    summary: "Population stability index > 0.2 on output-token-length distribution; drift candidate."

# Alert C — joint proxy signal (cost + TPOT + no traffic-shape change)
- alert: DriftProxyJoint
  expr: |
    (
      (abs(cost_per_request_delta_pct{model=~"prod-.*"}) > 10)
      and (input_token_distribution_psi{model=~"prod-.*"} < 0.1)
    )
    or
    (
      (abs(tpot_p95_delta_pct{model=~"prod-.*"}) > 10)
      and (model_version_stable{model=~"prod-.*"} == 1)
      and (capacity_headroom_ok{model=~"prod-.*"} == 1)
    )
  for: 20m
  labels:
    severity: page
    class: 2.1
    route: drift-triage
  annotations:
    summary: "Proxy-signal joint: cost or TPOT shift without traffic-shape or capacity cause."
```

Metric names are illustrative. Real deployments wire the emission in `mod-104` chapter 02 (Prometheus) and chapter 05 (drift pipelines — Evidently / Arize / WhyLabs for the PSI computation).

### 2.4 First-fifteen-minutes checklist (sub-class classification)

Per chapter 03 section 7; the on-call walks this on any alert from section 2.3.

1. **Which alert fired?** Canary (A), on-traffic distribution (B), or joint proxy (C)? The firing alert biases the sub-class guess.
2. **Check `gen_ai.response.model` label.** Value shifted in the window → sub-class **1.1 model-weight change**. Capture old vs. new. Next question: who promoted?
3. **Check engine version / image tag.** Rolled in the window → sub-class **1.2 kernel regression**. Cross-check the engine-release channel.
4. **Check model-format metadata.** New quantization or calibration pass → sub-class **1.3 numeric-format change**.
5. **Read the secondary signal table from chapter 03 section 1.4 against the panel.** Output-length shifted → more likely 1.1. First-token histogram shifted with unchanged length → more likely 1.2 or 1.3. `gen_ai.response.model` unchanged → rules out 1.1.
6. **Side-by-side canary vs. production.** Both moving → drift is real. Canary moves and production does not → production is not hitting the affected path (slice with the surface label). Production moves and canary does not → canary-set representativeness gap; note for quarterly refresh.
7. **Confirm user-facing impact.** Quality SLI, customer-support signal, eval peer's spot-check.
8. **Pick a reversible lever.** Rollback, route-around (pin provider snapshot), throttle. Chapter 03 section 8.
9. **Hand-off ticket.** Sub-class classification + canary evidence + production evidence + blast radius + mitigation in place → route to ML platform (1.1 internal / 1.3), inference-engine peer (1.2), or vendor-risk (1.1 provider alias move).

By minute 15: sub-class classification, suspect owner, blast radius, and a reversible lever selected.

## 3. Validation steps

A grader (or the learner during self-review) walks these checks.

1. **The canary set is specified in enough detail to build.** Size range, stratification plan, PII-scrub procedure, version identifier, schedule, tagging convention, reference-store location, metric names, decay plan.
2. **The on-traffic complement is specified, not hand-waved.** The panel specification names at least five of the nine rows from section 2.2.
3. **The three sub-classes are distinguishable in the design.** For each of 1.1 / 1.2 / 1.3, the design names the signal(s) that distinguish it from the other two. The chapter 03 section 1.4 table is the model; the learner's version may elide rows but must retain the three distinguishing columns: `gen_ai.response.model`, output-length, TTFT / TPOT.
4. **Alerts are joint-signal where chapter 03 says they should be.** The proxy-signal alert must require more than a single-signal condition; it must combine cost or TPOT with a stability check (input-token PSI or model-version-stable or capacity-headroom-ok).
5. **No user-facing SLI counts canary traffic.** The tagging and the budget-exclusion statement are both present.
6. **PII scrubbing is a signed-off checklist, not a model's judgement.** The submission names the policy and the responsible owner.
7. **Decay plan is calendar-bound.** Quarterly representativeness refresh, per-release reference re-capture, and inclusion in the quarterly SLO ritual.
8. **The first-15-minutes checklist is actionable without opening a search engine.** Each step is a dashboard panel, a label, or an engine channel.
9. **Hand-off ticket contents are enumerated.** Sub-class, evidence (canary + production), blast radius, mitigation.
10. **No invented metric names shipped as if real.** Metric names are documented as illustrative where they are not from a canonical `mod-104` chapter.

## 4. Rubric / review checklist

0–3 per criterion; 30 points total; pass threshold 24.

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Canary-set design complete | Missing | Partial (3–4 of 8 sections) | Most present (5–7 of 8) | All 8 sections present and defensible |
| 2 | On-traffic panel specification | Absent | 1–2 rows | 3–5 rows | 6+ rows with deploy markers overlay |
| 3 | Sub-class distinguishability | Not treated | One sub-class differentiated | Two differentiated | All three differentiated with the chapter 03 table shape |
| 4 | Alert rules (A / B / C) | None | One single-signal alert | Two alerts; one is joint | All three; C is joint on cost / TPOT + stability check |
| 5 | Canary traffic excluded from user-facing SLIs | Not addressed | Tag-only | Tag + budget-exclusion statement | Tag + budget exclusion + routing excluded from cost SLI |
| 6 | PII-scrub is a checklist, not model-judged | Model-judged | Manual-only | Checklist named | Checklist + owner + review cadence |
| 7 | Decay plan is calendar-bound | None | Quarterly only | Quarterly + per-release | Quarterly + per-release + tied to SLO ritual |
| 8 | 15-minute first-responder checklist | Missing | 1–3 steps | 4–7 steps | All 9 steps from chapter 03 section 7 |
| 9 | Hand-off ticket contents enumerated | Missing | Two fields | Four fields | All five fields (sub-class, canary ev, prod ev, blast, mitigation) |
| 10 | Metric names flagged illustrative where applicable | Invented without note | Mostly real; some invented | All grounded in `mod-104` or marked illustrative | Grounded + `mod-104` chapter reference per metric |

## 5. Common mistakes

- **Treating the canary set as the release-gate eval set.** The release-gate set is the eval peer's property; it is larger, slower, offline, and used *before* promotion. The canary set is small, online, *post-promotion*, and lives in the drift pipeline. Chapter 03 section 2.1 is explicit.
- **Hourly canary runs against a frontier-tier model without a budget note.** Cost is real; the design document should quantify dollars-per-day as operational insurance and frame it to the sponsor. 500-example hourly canary against a cheap-tier model is a few dollars a day; 5000-example canary against a frontier model is hundreds per month.
- **Alerting on cost alone.** Cost is a proxy signal; the joint alert must combine it with a stability check (input-token PSI < 0.1) to avoid firing every traffic-shape shift.
- **Letting canary traffic count toward user-facing SLIs.** Every row of the panel must slice canary vs. user traffic; metrics must label by `request.source`.
- **Omitting the reference store.** Without a last-known-good run, "comparison" has no second operand. The reference store is as important as the canary set itself.
- **Not distinguishing the three sub-classes.** A design that treats "silent drift" as a single class loses the triage distinction between weight / kernel / numeric and routes every incident to "the ML team." The point of the sub-classes is different owners.
- **Shipping metric names as if they were canonical when they are not.** Prometheus metric naming is per-engine, per-pipeline, per-team. The design says so.
- **Running the canary against *only* the hosted endpoint.** For stacks with a self-hosted path, the canary should run against both; otherwise sub-class 1.2 kernel regression is invisible to the canary.

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/03-silent-output-drift-and-canary-evals.md` — the authoritative in-module source; sections 1 (sub-classes), 2 (reference problem), 3 (production-traffic signals), 4 (proxy signals), 5 (architecture), 6 (alert design), 7 (first fifteen minutes), 8 (mitigation and route), 9 (decay).
  - `lessons/mod-104-observability-for-llm-and-gpu-workloads/` chapters 02, 05, 07 — Prometheus semantic conventions, drift pipelines (Evidently / Arize / WhyLabs), deploy-marker overlay.
  - `lessons/mod-103-slo-engineering-for-model-backed-services/` chapters 02 (quality SLI), 04 (cost SLI), 05 (multi-window multi-burn-rate), 06 (SLO ritual).
  - `lessons/mod-111-release-engineering-and-progressive-delivery/` chapter 03 — release-gate design.
- Official standards:
  - NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. The **Measure** function names output monitoring and drift measurement as a required activity; aligning your SLIs to it makes the burn directly reportable upward.
  - NIST AI 600-1 (Generative AI Profile) — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. *Information integrity* is the risk category most-adjacent to silent output drift; *Confabulation* if the drift's business impact is a hallucination-rate shift.
- Authoritative-project references:
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. (Adjacent coverage of ML-side risk vocabulary.)
  - MITRE ATLAS — <https://atlas.mitre.org/>. (Adversarial-view counterpart; useful where drift is caused by adversarial inputs rather than infra.)

<!-- follow-up-research: before shipping the exact PSI threshold (0.2) and the delta-threshold choices (5 pp / 2 pp canary, 10% cost / TPOT) into a production alert pack, calibrate against your baseline noise. The chapter 03 section 6.2 text names 0.2 as "widely-cited but not sacrosanct"; the right threshold for your traffic is domain-specific and should be documented against the baseline it was chosen from. -->
