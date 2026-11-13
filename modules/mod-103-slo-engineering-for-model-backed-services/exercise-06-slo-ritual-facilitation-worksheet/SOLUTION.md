# Exercise 06 — SLO Review Ritual Facilitation Worksheet — Reference Solution

## 1. Solution overview

Two artifacts for the monthly SLO review ritual on the **`acme-llm-
gw`** stack from exercises 01–05:

- **Part A** — a reusable facilitation worksheet template a chair
  can take, fill from Grafana two weeks before the meeting, and
  use to drive the 30-minute review.
- **Part B** — a dry-run of one month's review against the stack,
  demonstrating the worksheet's plumbing on a realistically messy
  month (one fast burn, one slow cost drift, one exception
  granted, one target-adjustment proposal, one new-SLI proposal).

The dry-run also surfaces one recurring-exception pattern that the
policy absorbs — the artifact that justifies running the review
every month even when the SLIs look ordinary.

---

## 2. Worked answer / Implementation — Part A: the facilitation worksheet template

> **File:** `ops/slo-review/worksheet-template.md`
> **Audience:** chair of the monthly SLO review for
> `acme-llm-gw`. Fill the worksheet from the SLO dashboards two
> weeks before the meeting; circulate 48 hours ahead as the
> pre-read.
> **Attendees (standing):** primary on-call, secondary on-call,
> SRE lead, business sponsor, eval peer, capacity owner. Chair
> rotates quarterly.
> **Time budget:** 30 minutes total.

### Section 1 — SLI health snapshot (10 min agenda cap)

| SLI | 28-day value | Target | Budget remaining | Trajectory vs. last review (↑ / → / ↓) | Owner |
|---|---|---|---|---|---|
| `chat-stream` TTFT cache-miss | _ | p95 ≤ 1.2 s | _% | _ | SRE |
| `chat-stream` TTFT cache-hit | _ | p95 ≤ 400 ms | _% | _ | SRE |
| `chat-stream` TPOT | _ | p95 ≤ 80 ms / tok | _% | _ | SRE |
| `chat-stream` ITL | _ | p99 ≤ 2 s | _% | _ | SRE |
| `chat-stream` E2E | _ | p95 ≤ 10 s | _% | _ | SRE |
| `chat-stream` goodput | _ | p90 ≥ 12 tok/s | _% | _ | Capacity |
| `code-assist` TTFT | _ | p95 ≤ 700 ms | _% | _ | SRE |
| `code-assist` E2E (schema-valid) | _ | p95 ≤ 2 s | _% | _ | SRE |
| `batch-summarize` E2E per-ticket | _ | p99 ≤ 180 s | _% | _ | Capacity |
| `batch-summarize` E2E job | _ | p99 ≤ 06:00 UTC weekdays | n/a daily | _ | Capacity |
| Hallucination | _ | ≥ 97% | _% | _ | Eval peer |
| Toxic-output | _ | ≥ 99.9% | _% | _ | Security peer |
| Tool-call-success | _ | ≥ 99% | _% | _ | SRE / tool-platform |
| Structured-output-conformance | _ | ≥ 99.5% | _% | _ | SRE |
| Refusal — warranted | _ | ≥ 99% | _% | _ | Eval + security |
| Refusal — over-refusal | _ | ≥ 98% | _% | _ | Eval peer |
| Protocol 2xx | _ | ≥ 99.9% | _% | _ | SRE |
| Partial-success | _ (warn-only until instrumentation lands) | ≥ 99% | _% | _ | SRE |
| Graceful-degradation (aggregate) | _ | ≥ 99.5% | _% | _ | SRE |
| Cost-per-1M-input (chat-stream) | _ | ≤ $80 | n/a — one-sided | _ | SRE + finance |
| Cost-per-1M-output (chat-stream) | _ | ≤ $120 | n/a — one-sided | _ | SRE + finance |
| Cost-per-completed-request (code-assist) | _ | ≤ $0.0015 | n/a — one-sided | _ | SRE + finance |
| Cost-per-successful-session (chat-stream) | _ (blocked on session-ID gap) | ≤ $0.16 | n/a — one-sided | _ | SRE + finance |

Chair instruction: circle rows where budget remaining is below
50% or where trajectory is ↓; those rows anchor the discussion.

### Section 2 — Burn-event review (7 min)

For each SLI that was in burn-rate alert since the last review,
one block:

- SLI name and incident reference.
- Duration of burn.
- Root cause (one line).
- Action taken (rollback / capacity add / exception grant / no
  action).
- Lessons captured and where (post-mortem link, runbook update,
  SLI-definition change).

Standing question: did any single burn repeat a pattern from a
prior month? If yes, flag for §6 (policy adjustment).

### Section 3 — Exception log review (3 min)

Table of every exception granted under the policy
(`ops/exceptions.md`) since the last review.

| Exception | Who granted | How long | Triggered by | Resolution |
|---|---|---|---|---|
| _ | _ | _ | _ | absorbed into policy / one-off |

Standing question: were any exceptions granted *more than once*
for similar triggers? If yes, flag for §6.

### Section 4 — Target-adjustment proposals (5 min)

For each proposed adjustment:

- SLI affected.
- Current target.
- Proposed target.
- Justification (baseline re-measured / model change / workload
  shift / sponsor renegotiation).
- Who signs off (SRE + sponsor for all; eval peer for quality
  SLIs; security peer for safety SLIs).

Decision outcome: accept / defer to next month / reject.

### Section 5 — New SLIs / retired SLIs (3 min)

For each new SLI:

- Surface and workload class.
- Proposed definition (one line).
- Target seed value (from baseline methodology).
- Dependencies (eval peer, security peer, capacity owner).

For each retired SLI, the retirement checklist:

- [ ] Alerts removed from Prometheus?
- [ ] Panels archived from Grafana?
- [ ] Baseline archived to the historical record?
- [ ] Retirement recorded in the policy?

### Section 6 — Policy adjustments (2 min)

Items escalated from §2 / §3. Each is either a motion to update
the policy (stated specifically — "change the correlation window
from 2 h to 1 h", "add refusal-over-refusal to the correlated-
burn rule", "tighten hallucination to 97.5%") or a deferred
action (named owner, due by the next review).

### Appendix — meeting-mechanics notes

- **Chair**: current quarter's chair is the SRE lead's designate.
  Co-chair (shadow for next quarter) attends from the SRE team.
- **Agenda timeboxing**: 10 / 7 / 3 / 5 / 3 / 2 minutes per section.
  When an item overruns, the chair converts it to a parking-lot
  ticket; the review *always* ends on time.
- **Overruns**: a parking-lot ticket is filed in
  `ops/slo-review/parking-lot.md` with the item, the owner, the
  resolution target (default: next monthly review). The chair
  does not extend the meeting; attendee calendars are respected
  as a rule of the ritual.
- **Minutes**: published within 24 hours of the meeting to
  `ops/slo-review/minutes/YYYY-MM-DD.md`; circulated to all
  attendees plus the engineering-leadership channel. Decision log
  (context / decision / consequences format) is embedded in each
  minutes file so the archive is searchable.
- **Quarterly allocation refresh**: triggers at the first review
  of each quarter; the chair extends the meeting to 60 minutes
  that week (noted on the calendar invite one month in advance).
  The SRE lead proposes new allocation numbers off-line from the
  prior quarter's burn history; the quarterly meeting ratifies.

---

## 3. Worked answer — Part B: the dry-run meeting

> **File:** `ops/slo-review/minutes/2026-11-05.md`
> **Date:** 2026-11-05 (first Thursday of the month)
> **Chair:** SRE lead's designate (chair-of-quarter).
> **Attendees present:** primary on-call, secondary on-call, SRE
> lead, business sponsor, eval peer, capacity owner.
> **Duration actual:** 29 minutes.

### Section 1 — SLI health snapshot (10 min used)

Chair pre-filled the worksheet from the SLO dashboards the day
before. Circled rows:

- `chat-stream` TTFT cache-miss: 48% budget remaining, ↓ from 72%
  last month. Fast-burn alert fired twice in the window.
- Hallucination: 83% budget remaining, ↓ from 95% last month.
  Fast-burn alert fired once.
- Cost-per-1M-output: slow-burn ticket open; 6-day drift from
  $112 → $118 average; approaching but not through the $120
  ceiling.
- Partial-success: budget value is still warn-only
  (instrumentation gap not yet closed); flagged for §4 blocker
  review.
- Cost-per-successful-session: still warn-only (session-ID gap).

All other rows green or trajectory flat.

Attendees confirm circled rows are the right focus. Sponsor asks
whether the goodput trajectory is drifting toward the floor;
capacity owner shows the dashboard and confirms it is stable at
p90 ≈ 14 tok/s. Noted.

### Section 2 — Burn-event review (8 min used — 1 min over, parking-lotted the overrun)

**Burn 1: `chat-stream` TTFT cache-miss, 2026-10-18.**  Duration
of burn 1 h 42 min. Root cause: a prefix-cache eviction storm
after a config change pushed the cache LRU threshold too high.
Action taken: rollback within 30 minutes; cache back to previous
threshold within 90 minutes. Lessons: cache-threshold changes
added to the §2.2 freeze map "model promotions" row (previously
implicit; made explicit). Post-mortem link:
`postmortems/2026-10-18-cache-eviction.md`. Runbook updated for
the cache-threshold rollback step.

**Burn 2: Hallucination, 2026-10-24.**  Duration 47 minutes. Root
cause: a retrieval-index corruption on one of three replicas; 30%
of `chat-stream` traffic was hitting the corrupted replica and
getting ungrounded responses. Action: eval peer alerted via
`#eval-releases` (correlated-burn policy §2.6); SRE routed
traffic off the bad replica; retrieval-index rebuilt the same
day. Lessons: retrieval-replica health check added to the SLI
surface as a dependency monitor (not an SLI); the SLI itself
behaved correctly. Post-mortem link:
`postmortems/2026-10-24-retrieval-index.md`.

**Burn 3 (slow): cost-per-1M-output ticket, open since 2026-10-29.**
No action yet; slated for §4 discussion below.

No burn repeated a pattern from a prior month, but the cache-
eviction (burn 1) is the *third* burn this quarter where a
capacity-tuning parameter escaped the freeze map. **Flagged for
§6.**

### Section 3 — Exception log review (2 min used)

One exception granted since the last review:

| Exception | Who granted | How long | Triggered by | Resolution |
|---|---|---|---|---|
| Hallucination SLI paused during 2026-10-24 incident (to prevent cascade paging from correlated burn) | IC | 2 h | Retrieval-index corruption | One-off — resolved by the incident itself |

Review question: has this exception shape been granted before?
Answer: yes — a similar "pause the SLI during correlated incident"
exception was granted in 2026-07 during a provider-pricing-feed
outage. **This is the second time in a quarter; the pattern is
recurring and is flagged for §6 to absorb into the policy.**

### Section 4 — Target-adjustment proposals (5 min used)

**Proposal A: tighten hallucination from 97% → 97.5%.**

- Justification: new retrieval model (`retrieval-v2`) deployed
  2026-09; eval peer re-baselined October and observed a
  baseline shift from 96.8% → 98.2% grounded on the standing
  eval set.
- Sign-off needed: SRE, sponsor, eval peer.
- Discussion: sponsor supportive — a 97.5% target still has
  headroom against the 98.2% baseline and strengthens the user
  commitment. Eval peer confirms the baseline is stable over
  three consecutive weeks. SRE flags that the burn-rate alert
  threshold has to move (14.4 × 2.5% = 36%, down from 43.2%)
  and the runbook already has the recomputation script.
- **Decision: accept.** Effective 2026-11-10. Chair owns the
  change ticket.

**Proposal B: accept the cost-per-1M-output drift from $112 → $118
without changing the target.**

- Justification: traffic-mix shift toward longer context sessions
  (a product-driven feature launched October); forecast is flat
  at $118–$120 until the next product change.
- Sign-off: SRE + sponsor + finance (represented by sponsor).
- Discussion: sponsor accepts; the $120 ceiling has 15% headroom
  built in per exercise 04's derivation; the drift is anticipated
  and absorbed. Finance to re-check next month.
- **Decision: no target change; leave the ticket open as a
  trend-watch item.**

### Section 5 — New SLIs / retired SLIs (3 min used)

**New SLI proposal: image-upload modality latency on `chat-stream`.**

A new multimodal feature launched 2026-10 (users can attach an
image to a chat turn; the model now vision-processes it). The
current SLI surface does not distinguish text-only turns from
multimodal turns. Proposed SLI: TTFT p95 on multimodal turns ≤
2.5 s (higher than text because vision pre-processing is heavier).

- Dependencies: eval peer to confirm hallucination judging is
  calibrated on multimodal turns (currently unclear; may require
  judge-set extension).
- Decision: **accept in principle, defer target-setting one month
  until the eval peer confirms judge calibration and we have 28
  days of multimodal traffic for baseline.** Capacity owner takes
  the baseline action.

No SLI retirements this month.

### Section 6 — Policy adjustments (2 min used)

Two policy motions, escalated from §2 and §3:

**Motion 1 — freeze-map expansion (from §2 burn 1).** Add
"capacity-tuning parameters" (prefix-cache config, admission-queue
thresholds, KV-cache eviction policy) as explicit freeze targets
on `chat-stream` latency burns. Previously implicit under
"admission-queue config"; three incidents this quarter show the
explicit enumeration is needed. **Carried.** Policy update ticket:
chair owns.

**Motion 2 — absorb the "pause SLI during correlated incident"
exception into the policy (from §3).** Add a clause to §2.7
exception process: "The IC may pause a specific SLI's paging for
the duration of an in-flight correlated incident to prevent cascade
paging, logged with granter / duration / trigger as above." Removes
the need for an ad-hoc exception on each correlated incident.
**Carried.** Eval peer confirms the bridge to the eval-release gate
is unaffected. Policy update ticket: chair owns; ships with motion
1 in a single revision.

### Timing summary

| Section | Budget | Actual |
|---|---|---|
| 1 | 10 min | 10 |
| 2 | 7 min | 8 (1 min over — parking-lotted goodput depth-dive) |
| 3 | 3 min | 2 |
| 4 | 5 min | 5 |
| 5 | 3 min | 3 |
| 6 | 2 min | 2 |
| **Total** | **30** | **29** |

Chair note: the 1-minute overrun on §2 was absorbed by §3
coming in under budget. If §3 had been average, the chair
would have cut §4's cost-drift discussion rather than overrun.

### Sponsor summary (post-meeting, stretch-goal convention)

> Three key outcomes.
> 1. Tightened the hallucination SLO (97% → 97.5%) — a stricter
>    commitment, defensible because our retrieval improvement
>    this quarter moved the baseline up.
> 2. Updated the freeze policy to protect against the capacity-
>    tuning pattern we've now seen three times.
> 3. Opened an image-upload SLI proposal — target-setting
>    deferred one month for a clean baseline.
> No asks of the sponsor this month beyond the hallucination
> tightening sign-off (requested in-meeting and granted).

---

## 3. Validation steps

1. Confirm Part A worksheet has all six sections plus the
   mechanics appendix.
2. Confirm every worksheet section is template-shaped (column
   headers, placeholder rows, chair instructions) and not filled
   with exercise-specific values.
3. Confirm Part B dry-run covers all five required scenarios:
   fast burn (cache-miss), slow burn (cost drift), exception
   granted (hallucination pause during incident), target
   adjustment (hallucination tightening), new SLI proposal
   (multimodal).
4. Confirm each dry-run scenario has a *decision* (not just a
   discussion).
5. Confirm attendees named per scenario (at the §1 chair opening
   and referenced in each scenario where sign-off was explicit).
6. Confirm timing budget: dry-run fits 30 minutes (29 actual
   with one parking-lotted overrun).
7. Confirm the recurring-exception pattern (hallucination pause
   granted twice in a quarter) is caught in §3 and absorbed in
   §6 — the mechanism the worksheet is designed to produce.
8. Word-count check: Part A ≤ 1500 words, Part B ≤ 2000 words.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Worksheet template is reusable | Template-shaped, not filled | Pre-filled with exercise-specific values |
| R2 | All six sections present | §1–§6 with chair instructions | Section missing |
| R3 | Appendix mechanics populated | Chair/co-chair, timing, overrun policy, minutes location, allocation-refresh trigger | Appendix stubbed |
| R4 | Dry-run has 5+ scenarios | Fast burn, slow burn, exception, target change, new SLI | Fewer than five |
| R5 | Each scenario has a decision | Decision line present on each | Scenario ends in "discussion" |
| R6 | Timing budget respected | 30 min total; overruns parking-lotted | Dry-run runs 45+ min with no cut |
| R7 | Attendees named | Explicit roster + per-scenario sign-off | Unnamed "we decided" |
| R8 | Recurring-pattern catch | At least one recurring-exception pattern surfaced and absorbed | Exception review is pro-forma |
| R9 | Successor can chair next month | Reviewer picks up the worksheet and runs the next review without original author | Requires hand-holding |

---

## 5. Common mistakes

- **Filling the template with the dry-run values.** The template
  must be legible *before* data lands; a chair should be able to
  copy it and not need to unfill the prior month's values.
- **Letting the meeting overrun.** The 30-minute budget is the
  ritual; overruns destroy sponsor attendance. Parking-lotting is
  the discipline.
- **Skipping the sponsor.** A review without the sponsor is a
  status meeting, not a review. The worksheet names the sponsor
  as a standing attendee.
- **Not catching recurring exceptions.** The exception log review
  is *the* mechanism for turning recurring patterns into policy.
  Pro-forma treatment (pass the table over without checking for
  repeats) misses the point.
- **Target adjustments without eval-peer sign-off on quality
  SLIs.** The sign-off column is explicit for a reason. A sponsor-
  only sign-off on a hallucination target ignores the peer whose
  pipeline *is* the SLI.
- **New SLI proposals without baseline.** The acceptance pattern
  is "accept in principle, defer target-setting until baseline
  lands" — not "accept with a guessed target."
- **Omitting minutes publication.** The decision log is the
  artifact future chairs search; if minutes are not published,
  the ritual's memory is in individuals, not the archive.

---

## 6. References

- Google SRE Workbook, *SLO review and iteration* (the chapter
  in *Implementing SLOs*) —
  <https://sre.google/workbook/implementing-slos/>. Source for the
  review-ritual structure replicated in Part A.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for the
  burn-event / exception discipline the worksheet consumes.
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the target-adjustment convention in §4.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>. Source
  for the governance framing on recurring exceptions and policy
  absorption (NIST AI RMF "Manage" function).
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the security-peer standing-attendee convention (§1
  roster).
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-06-slo-ritual-facilitation-worksheet.md`
  (learning exercise this solution answers) plus exercises 01–05
  SOLUTION.md (the stack and policy this review governs).
