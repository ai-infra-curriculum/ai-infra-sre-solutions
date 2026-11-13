# exercise-05 — Triage Decision Matrix and Escalation Tree — Solution

> The learning exercise prompt is a `TBD` stub. This solution interprets the exercise against the module's stated chapter 08 (`08-triage-decision-matrix-and-ten-walkthroughs.md`, named in the `README.md` but not yet published at the time of writing) and synthesises the matrix / escalation tree from the ten classes in chapter 01, the peer-boundary content named in chapters 03–07, and the module's stated objective: a decision matrix the on-call can follow at 3 AM.

> Because chapter 08 is still a `TBD` stub in the learning repo, where this solution names a concrete signal / owner / mitigation it cites the originating chapter 01–07 passage; invented specifics (thresholds, exact UI strings) are flagged.

---

## 1. Solution overview

The triage decision matrix is the single page the on-call opens when a burn-rate alert fires and they do not yet know the class. Its job is to narrow ten classes to one in under 15 minutes using signals that are already on the `mod-104` dashboard. The matrix is *flat*, not a tree — every row is addressable; the on-call does not descend a decision tree they have to memorise.

The escalation tree is a separate artifact: *given a declared class*, who is the first peer, who is the second peer, and when does the SRE escalate upward (severity bump, declared-incident, legal / compliance). The tree makes routing cheap once classification is done.

Both artifacts assume the dashboard rows chapters 03–07 author are live. Where they are not, the gap is upstream and the matrix is unusable; that condition is itself routeable (back to the observability / `mod-104` owner).

## 2. Implementation — worked answer

### 2.1 The decision matrix — ten classes, one page

The matrix is a table a sleep-deprived on-call reads top-to-bottom; the row whose **confirming signals** match the dashboard panel is the class. Where signals are ambiguous between rows, the **disambiguator** column names the single additional check that separates them.

| Class | Confirming signal (joint, where noted) | Disambiguator vs. neighbour | Reversible lever (SRE lane) | Owning peer (first) | Owning peer (second / composition) | Declared incident? |
|---|---|---|---|---|---|---|
| **2.1 Silent output drift** — weight / kernel / numeric | Canary-eval pass-rate drop **or** output-embedding centroid shift **or** `gen_ai.response.model` label change | `gen_ai.response.model` moved → 1.1 weight; engine image moved → 1.2 kernel; registry quantization/calibration moved → 1.3 numeric | Rollback; pin provider snapshot; route-around; throttle | ML platform (1.1 internal / 1.3) | Inference-engine peer (1.2); vendor-risk (`mod-107`) for 1.1 provider alias move | No (unless composite with 2.3) |
| **2.2 Hallucination spike** | Factuality / groundedness quality SLI burn | Retrieval-score / groundedness drop → 2.2+2.6; `gen_ai.response.model` moved → 2.2+2.1; judge-agreement slipped → 2.2+2.8 | Rollback prompt / model / retrieval-config; route-around; throttle | Application / prompt layer | Eval peer; retrieval; model layer | No (unless 2.2+2.3 composite) |
| **2.3 Toxic / unsafe output blast** | Safety-category counter spike **or** external report | Category tells you which playbook (hate / self-harm / CSAM-adjacent / CBRN / jurisdiction-specific) | Short-circuit the output path for the category; pin model version; disable tool-call path; rate-limit the input class | Security-safety-compliance (`mod-110`) | Vendor-risk (`mod-107`); application; red-team | **Yes, from minute 1** |
| **2.4 Refusal-rate spike** | `gen_ai.response.finish_reasons = content_filter` counter rise **or** refusal-SLI burn | `gen_ai.response.model` moved → 2.4+2.1; judge-recalibrated → 2.4+2.8; threshold change on filter → 2.4-only | Rollback model / prompt / filter-threshold change; route-around | Model layer | Application / prompt layer; eval peer | No (unless surface has refusal-critical SLO) |
| **2.5 Prompt-cache cliff** — prefix / template / tokenizer | **Joint**: cache-hit rate ↓ **AND** TTFT ↑ **AND** input-token cost ↑ on same deploy | Prompt / system-msg diff → 2.5.a; template structural diff → 2.5.b; tokenizer / SDK / gateway dep bump → 2.5.c; no diff → provider-side | Move varying content to suffix; rollback template; rollback dep pin; cache-warm; route-around | Inference-engine peer | Application / prompt layer; vendor-risk | No |
| **2.6 Retrieval-freshness regression** — staleness / embedding drift / corruption | **Joint**: retrieval-grounded-answer pass-rate ↓ **AND** retrieval-score shift **AND** document-age or corruption signal | Ingestion freshness SLI burning → 2.6.a; query-embedding ≠ index-embedding version → 2.6.b; chunk-length / encoding anomaly → 2.6.c | Trigger ingestion-recovery; pin embedding model; roll back ingestion change; re-ingest affected sources | Data / config pipeline (`mod-112`) | ML platform (embedding model); application (query path) | No |
| **2.7 Agent cascade** — recursion / rate-limit / schema | **Joint**: depth histogram p99 ↑ (or depth-cap hits ↑) **AND** per-session cost or wall-clock ↑ | Same tool similar args → 2.7.a; same tool 429 errors → 2.7.b; same tool schema errors → 2.7.c; mixed tools + conformance → engine | Lower depth cap; short-circuit tool; route-around; throttle surface; rollback | Application / framework | Tool owner (internal or external); inference-engine peer (conformance) | No |
| **2.8 Judge-model drift** | Judge-vs-human agreement slip **or** judge version / prompt / rubric diff | Judge version pin changed → direct; judge prompt / rubric diff → direct; 10–20-sample spot-check of flagged outputs looks fine to a human → direct | **Hold.** Do not roll the model. Pin the judge. | Eval peer | — | No (measurement incident; SLO retro-adjust per `mod-103` §5.7) |
| **2.9 Provider-degraded mode** | TTFT / TPOT / quality shift on `gen_ai.system=<provider>` × `gen_ai.response.model=<id>` slice, with status page clean | Only one provider slice regressed → provider; one region only → provider backend; `gen_ai.response.model` label shifted → silent snapshot move; multiple providers regress together → your side | Route around (multi-provider gateway); pin version to dated snapshot; scale self-hosted fallback; throttle; vendor ticket | Vendor-risk (`mod-107`) | Provider support; SRE fallback routing | Internally yes; externally per `mod-107` policy |
| **2.10 Policy / safety regression** — composite 2.1 + 2.3 + 2.4 | Either 2.3 or 2.4 fires **concurrently** with a `gen_ai.response.model` label change | Which content axis is the burn — composite is identified, not disambiguated | Short-circuit the category path; pin model version; disable tool-call injection path | Security-safety-compliance (`mod-110`) | Vendor-risk (`mod-107`); application | **Yes, from minute 1** |

### 2.2 The pre-triage filter — classical-SRE first

Before the on-call enters the matrix, a two-line filter catches classical-SRE causes that look like AI failures:

- **Is the HTTP 5xx rate shifted or the probe failing?** → classical availability playbook; this module does not apply.
- **Is DCGM reporting XID / ECC / thermal / NVLink events?** → hardware playbook (`mod-104` chapter 03); 2.1 / 2.9 can look identical at first glance. Chapter 01 section 4 is explicit: this module explicitly excludes hardware.

If neither filter fires, the matrix applies.

### 2.3 The escalation tree

Once a class is declared, the escalation tree names the first peer, the second peer, and the leadership trigger. Tree is flat in the usual case; the compound and declared-incident classes have a second branch.

```
Classified → declare in incident channel → route per matrix row.

If class in { 2.3, 2.10 }:
  immediately (minute 1):
    page security-safety-compliance (mod-110) oncall
    page vendor-risk (mod-107) if provider-caused
    engage legal / PR / executive per mod-110 severity matrix
  do not improvise comms; do not post externally without legal sign-off

If class is 2.9:
  page vendor-risk (mod-107)
  open evidence-bearing vendor support ticket
  if critical surface:
    activate multi-provider gateway traffic shift
    scale self-hosted fallback

If class is 2.8:
  page eval peer; do NOT roll the model
  flag SLO window as suspect; prepare retro-adjustment per mod-103 chapter 05 §7

If class is 2.1 / 2.2 / 2.4 / 2.5 / 2.6 / 2.7:
  engage the owning peer(s) per matrix row 2
  SRE owns reversible lever (rollback / route-around / throttle)
  owning peer owns root-cause fix

Leadership trigger (bump severity to exec):
  - 2.3 confirmed category-X event (any confirmed CSAM-adjacent, CBRN,
      regulated jurisdiction-specific event)
  - 2.10 confirmed composite
  - 2.9 critical-surface with no viable fallback
  - any class where SRE-lane mitigation is unavailable and user impact is
      still increasing after 30 minutes
```

### 2.4 How the matrix is used at 3 AM

The intended operational flow:

1. SLO burn alert fires; on-call opens the incident channel.
2. On-call opens the "triage" dashboard (chapters 03–07 panels overlaid; chapter-08 single-page view).
3. **Pre-triage filter** (section 2.2): classical SRE? hardware? If yes, exit this module.
4. **Scan matrix rows top-to-bottom** against the dashboard. The first row whose confirming-signal column matches the panel's current shape is the hypothesis.
5. **Disambiguator check** against adjacent rows that share any signal. One check separates the candidates.
6. **Declare class** in the incident channel with the matrix row name.
7. **Pull the reversible lever** from the matrix (SRE lane).
8. **Follow the escalation tree** to route to the owning peer(s) and, where applicable, to compliance / vendor-risk / leadership.
9. **Hand-off ticket** carries class, sub-class, confirming signals, disambiguator result, lever in place, blast radius.
10. **If after 30 minutes the on-call has not declared a class**: the matrix has a dashboarding gap (signal missing) or a training gap (operator does not know where to look). Chapter 01 section 5 recognition budgets are the discipline; report the gap at the next business day for the quarterly matrix review.

### 2.5 Matrix maintenance — the quarterly ritual

The matrix is a working document per chapter 01 section 6:

- **Quarterly**: review every incident from the past quarter. Did any fail to map cleanly? Did a new pattern emerge? Add / remove / re-word rows.
- **On major model / engine / platform change**: likely new classes (multi-modal, long-horizon-agent). Add rows.
- **On regulatory change**: NIST AI 600-1, EU AI Act implementing acts, sector regulations update vocabulary. Re-draw the external mappings in exercise-01's catalog.
- **On dashboarding change**: if `mod-104` adds or removes a panel, the matrix's confirming-signal column must be updated.

## 3. Validation steps

A grader walks these checks on a submitted matrix / escalation tree.

1. **Matrix has one row per class.** All ten classes present.
2. **Confirming signals are joint where chapter 01 names a joint.** 2.5 / 2.6 / 2.7 must say "joint"; a single-signal row for any of them is a fail.
3. **Disambiguator column present and specific.** For every row that shares a signal with a neighbour (2.5↔2.1 TTFT; 2.2↔2.6 groundedness; 2.3↔2.4 safety training; 2.1↔2.9 provider snapshot), the disambiguator names the *single* additional check.
4. **SRE reversible-lever column names only levers in the SRE lane.** Rollback / pin / route-around / throttle / short-circuit / depth-cap. Model retraining, prompt editing, judge rubric edits, filter-threshold edits are out-of-lane and belong to peers.
5. **Owning peer columns are specific.** No "the AI team"; the matrix names ML platform / inference-engine peer / data-pipeline peer / eval peer / security-safety-compliance / vendor-risk / application / prompt / framework / tool / provider.
6. **2.3 and 2.10 are marked declared-incident from minute 1.** The chapter 04 / section 7 comms-posture discussion requires this.
7. **2.8 reversible lever is "hold".** Not rollback. The SRE's job is to not act on suspect signal.
8. **Pre-triage filter present.** Classical availability / hardware filters precede the matrix.
9. **Escalation tree is flat where the usual case is flat.** Composite / declared classes have a second branch; linear classes do not.
10. **Maintenance cadence named.** Quarterly review + on-major-change + on-regulatory-change + on-dashboarding-change.

## 4. Rubric / review checklist

0–3 per criterion; 30 points total; pass threshold 24.

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Ten rows present, canonical class names | <10 or renamed | 7–9 | 10, minor rename | 10, canonical |
| 2 | Joint signals where chapter 01 names a joint | None marked joint | 1 of 3 | 2 of 3 | All of 2.5 / 2.6 / 2.7 marked joint |
| 3 | Disambiguator column specific and complete | Absent | Present for 1–2 rows | Present for most rows | Present for every row that shares a signal with a neighbour |
| 4 | SRE reversible-lever column stays in lane | Includes out-of-lane levers | Mostly in-lane; a few leaks | Entirely in-lane | In-lane + explicitly separates SRE lever from peer-owned fix |
| 5 | Owning peer columns are peer-specific | "The team" | One peer per row | Specific peers | Specific peers + composition where chapter names a composition |
| 6 | Declared-incident marking for 2.3 and 2.10 | Missing | Marked without minute-1 framing | Minute-1 framing | Minute-1 framing + comms posture reference `mod-110` |
| 7 | 2.8 lever is "hold" / "do not roll" | Rollback named | Mixed guidance | Hold named | Hold + pin-the-judge + SLO retro-adjust reference `mod-103` §5.7 |
| 8 | Pre-triage filter present | Absent | Availability only | Availability + hardware | Availability + hardware + explicit "this module does not apply" exit |
| 9 | Escalation tree matches chapter structure | Missing | Flat only | Flat + declared-incident branch | Flat + declared-incident + leadership trigger + vendor-risk branch |
| 10 | Maintenance cadence named | Absent | Quarterly only | Quarterly + on-major-change | All four cadences (quarterly / major / regulatory / dashboarding) |

## 5. Common mistakes

- **Writing the matrix as a decision tree.** A tree the on-call has to descend from a root slows triage. The matrix is flat: every row is directly addressable. Chapter 01 section 3 names mutual distinguishability at triage time as the design goal.
- **Omitting the disambiguator column.** Three classes commonly share one signal (2.1 ↔ 2.9 both move `gen_ai.response.model`; 2.5 ↔ 2.1 both move TTFT; 2.2 ↔ 2.6 both move groundedness). Without the disambiguator, triage stalls at "could be this or that."
- **Letting the SRE edit the safety filter at 3 AM.** The lever is *rollback of the change that moved the threshold*, not an ad-hoc threshold edit. Chapter 04 section 8 is explicit.
- **Rolling the model when the quality SLI burns and the judge is suspect.** Pattern B from chapter 07 section 1.6 — everything is changing at once — demands hold, not action. Rolling the model on a judge-drift incident is the headline failure mode.
- **Not marking 2.3 as declared-incident at minute 1.** The regulatory / reputational blast radius is nonlinear in rate. The retrospective cost of not declaring is higher than the cost of declaring. Chapter 04 section 7 is explicit; `mod-110` is the owning-module authority.
- **Routing 2.9 directly to the provider's support line.** The SRE's lane is route-around / pin / fallback; vendor-risk is the owner of the vendor interaction. Chapter 07 section 2.7 names the comms posture.
- **Missing the hardware pre-triage filter.** DCGM XID errors look like 2.1 or 2.9 at first glance; a hardware incident mis-routed to the AI matrix burns 20 minutes.
- **Not writing the leadership trigger.** The matrix is operator-first, but the escalation tree is where the sponsor / exec / legal paths live. A matrix without a trigger means the on-call makes severity-bump decisions ad hoc.
- **Freezing the matrix.** Chapter 01 section 6 is explicit: quarterly update, on-major-change update, on-regulatory-change update. A "final" matrix is a stale one.

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/01-ai-specific-failure-taxonomy.md` — the ten classes, the design constraints (mutual distinguishability, routability, grounding), the recognition budgets, the "working document" discipline.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/02-grounding-in-nist-aiid-oecd-mit.md` — external vocabulary the retrospective carries.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/03-silent-output-drift-and-canary-evals.md` — 2.1 sub-class signals; row 1 of the matrix.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/04-hallucination-toxic-refusal-spikes.md` — 2.2 / 2.3 / 2.4 signals, declared-incident comms posture.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/05-prompt-cache-cliffs-and-retrieval-regressions.md` — 2.5 / 2.6 joint signals.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/06-agent-tool-call-cascades.md` — 2.7 joint signal, hand-off envelope.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/07-judge-drift-and-provider-degraded-mode.md` — 2.8 / 2.9 "hold / triangulate" posture.
  - `lessons/mod-101-ai-sre-position-on-the-ladder/` chapter 02 — peer boundaries.
  - `lessons/mod-104-observability-for-llm-and-gpu-workloads/` chapter 07 — dashboard authoring; this matrix is the single-page view that sits atop the panels chapter 07 assembles.
  - `lessons/mod-103-slo-engineering-for-model-backed-services/` chapter 05 — multi-window multi-burn-rate; the alert that drops the on-call into the matrix.
  - `lessons/mod-108-incident-response-for-ai-systems/` — the incident lifecycle the matrix feeds.
  - `lessons/mod-110-security-safety-and-compliance-reliability-crossover/` — declared-incident comms policy for 2.3 / 2.10.
- Authoritative-project references:
  - NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. The **Manage** function is where incident routing and mitigation live; the matrix is an operator-side implementation of Manage.
  - NIST AI 600-1 — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>.
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>.
  - MITRE ATLAS — <https://atlas.mitre.org/>.

<!-- follow-up-research: chapter 08 in the learning repo (`08-triage-decision-matrix-and-ten-walkthroughs.md`) is still a scaffolded stub at the time of writing. The structure of this matrix is reconstructed from chapters 01–07 and from the `README.md`'s module objective. When chapter 08 lands, cross-check the row ordering, the disambiguator-column design, and the leadership-trigger list against the published text. -->
