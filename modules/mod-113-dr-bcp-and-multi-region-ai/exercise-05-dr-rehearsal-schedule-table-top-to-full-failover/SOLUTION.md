# Exercise 05 — DR Rehearsal Schedule, Tabletop to Full Failover — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-05-dr-rehearsal-schedule-table-top-to-full-failover.md`.

> **Scope note.** The paired exercise prompt asks the learner to design the DR exercise programme for an AI workload — the annual schedule of drills that validates the plans from exercises 01–04, in the tier structure NIST SP 800-34 §3.5 and §3.6 prescribe (tabletop → structured walkthrough → simulation → parallel → full-interruption). The reference solution works the schedule against the same fabricated `chatcat` product used across this module family and produces the schedule document, the per-drill fixtures, and the grading templates.

## 1. Solution overview

NIST SP 800-34 Rev. 1 §3.5.2 names five tiers of exercise, in increasing realism and cost:

1. **Tabletop** — a facilitated discussion. No systems touched. Cheap, fast; uncovers plan gaps and role confusion.
2. **Structured walkthrough** — the plan is executed in a conference-room review with named decision-makers and an outside facilitator. No systems touched.
3. **Simulation** — synthetic events are played against the systems (fault injection in a non-production environment). Systems touched, but production is not.
4. **Parallel test** — recovery systems are brought up and operate *in parallel* with production. No cutover; both are live, and the test is whether recovery systems can carry the load.
5. **Full-interruption** — production cuts over to the recovery path. The real test; also the most disruptive.

A viable programme is **not** "run tier 5 annually". That would be theatrical. The correct shape is a cadence that re-runs the cheap tiers often and the expensive tiers less often, with the cheap tiers serving as readiness gates for the expensive ones.

For an AI workload, two additional considerations shape the programme:

- **Quality-recovery testing (judge pipeline reconstitution) is cheap and worth running out-of-band** between major drills. It validates RTO(quality), which is the AI-specific recovery contract.
- **Provider-failover testing requires vendor coordination** and is not always allowed during the vendor's peak traffic windows. The programme schedules provider drills at opportune windows, not on a rigid calendar.

The deliverable is a committed schedule `programmes/dr-exercise-programme.md` plus:

- `programmes/exercises/` — the fixture per drill (seven fixtures below).
- `programmes/grading/` — the per-drill grading template.
- `programmes/history/` — the append-only history of past drills, used as evidence for audit (feeds into `mod-113/exercise-06`).

### 1.1 The annual cadence at a glance

| Tier | Frequency | AI-specific drill(s) | Participants |
|------|-----------|----------------------|--------------|
| Tabletop | Monthly, rotating scenarios | Monthly tabletop over a rotating menu of 12 scenarios | Oncall ICs + backup |
| Structured walkthrough | Quarterly | ISCP walkthrough + residency walkthrough | IC pool + plan owner + audit |
| Simulation | Quarterly | Chaos drill in staging (one component per quarter) + judge-pipeline reconstitution | SREs + ML Platform |
| Parallel | Semi-annual | Standby region + quality pipeline carrying shadow traffic | SREs + Data Platform |
| Full-interruption | Annual | Full regional failover window, pre-announced | Full cross-functional |

### 1.2 Why the cadence looks like this

- **Monthly tabletop.** The plan's "does the IC know what to do" muscle atrophies fast. A monthly scenario rotation keeps the role fresh. The twelve scenarios are enumerated below (§2.1).
- **Quarterly walkthrough.** Plan drift is real: runbooks referenced in the ISCP get renamed, authorities change, contact lists go stale. Quarterly is often enough to catch it and infrequent enough to not burn goodwill.
- **Quarterly simulation.** Fault injection in staging (chaos) is where surprising things happen. Coupling it with the judge-pipeline reconstitution gives the exercise a clean exit criterion (groundedness back within 2 sigma).
- **Semi-annual parallel.** Running the standby region hot-in-parallel is expensive (capacity, operator attention) but it is the only drill that exposes capacity-planning errors. Twice a year matches most teams' spend tolerance.
- **Annual full-interruption.** The only drill that proves the plan end-to-end. Done once a year, in a pre-announced window, with customer notice, and with the plan to abort if it diverges.

## 2. Implementation — worked answer

### 2.1 The twelve monthly tabletop scenarios

Each scenario is a 30-minute facilitated discussion. One scenario per month; the sequence repeats annually. Scenarios are chosen to cover the full fault taxonomy from `mod-105` and `mod-108` and the specifically AI-flavoured classes.

| Month | Scenario slug | Primary plan exercised | Key decision moment |
|-------|---------------|------------------------|---------------------|
| Jan | `s-regional-outage-eu` | ISCP + playbook exercise 02 | DNS flip authority |
| Feb | `s-provider-primary-degraded` | ISCP + exercise 02 §4 | When to pin fallback provider |
| Mar | `s-retrieval-index-corruption` | ISCP recovery §4.2 | Snapshot choice vs. replica |
| Apr | `s-weights-attestation-fail` | Exercise 01 PC-04 + §3 | Rollback to last green pin |
| May | `s-provider-silent-version-bump` | `mod-107` + ISCP §9 trigger | Declare triggered review |
| Jun | `s-both-regions-impaired` | Playbook §6 degraded mode | Enable canned-response mode |
| Jul | `s-agent-state-loss-partial` | Exercise 03 plan | Reconciliation pacing |
| Aug | `s-residency-cross-scope-attempted` | Exercise 04 invariants | Scope boundary enforcement |
| Sep | `s-judge-pipeline-outage` | ISCP §4 quality-track | Serving-only mode |
| Oct | `s-cost-blowup-during-failover` | `mod-106` + ISCP §4 | Budget guardrail activation |
| Nov | `s-cyber-incident-triggers-isc` | ISCP activation + IR handoff | Joint-command structure |
| Dec | `s-vendor-decommissions-endpoint` | `mod-107` + ISCP triggered review | Fallback-provider swap |

Scenario files live in `programmes/exercises/s-*.md`. Example fixture below.

### 2.2 Example tabletop fixture — `s-regional-outage-eu`

```markdown
# Tabletop fixture — s-regional-outage-eu

**Tier:** tabletop. **Duration:** 30 min. **Owner:** AI SRE Director.

## Roles at the table
- Facilitator (reads the scenario)
- Oncall IC (the one being tested)
- Scribe
- Vendor Liaison (silent unless cue says otherwise)
- Data Platform oncall (silent unless cue says otherwise)
- Observer (grading against the rubric in §grading/tabletop.md)

## Pre-brief (facilitator reads aloud)
"At 14:00 UTC, the cloud provider status page shows `eu-west-1` as impaired.
`chatcat_route_availability{region=eu-west-1}` has dropped to 0.72 and is still falling.
`eu-west-3` is reporting healthy. The hosted-model provider's `eu-paris` endpoint is
reporting intermittent 503s. There are ~8,200 active sessions in the EU scope."

## Beats and expected IC actions

| T+ | Facilitator cue | Expected IC action |
|----|-----------------|---------------------|
| 00 | Drops the scenario | Opens `pb-regional-outage-chatcat`; verifies §0 pre-conditions |
| 05 | "Dashboards are loading; provider status page shows yellow for eu-paris" | Confirms scope: region impairment + possible provider degradation |
| 08 | "Vendor Liaison just joined the bridge" | Assigns vendor-path assessment to Vendor Liaison |
| 12 | "Replica lag reported as 42 minutes" | Notes replica lag against the 1h RPO target; proceeds |
| 15 | "DNS flip has propagated; serving metrics are climbing on `eu-west-3`" | Begins step through §2 of the playbook |
| 20 | "Provider eu-paris now reporting all requests failing" | Authorises §4 pin to `eu-frankfurt` fallback |
| 25 | "Groundedness metric has stabilised at baseline −1.2 sigma after 2 cycles" | Declares RTO(quality) met |
| 28 | "Observer, open discussion" | Facilitator calls time; retro begins |

## Red-team twists (facilitator may inject one)
- "Replica lag is actually 2h, not 42min." — IC should pause and consult Data Platform
  on freshness vs. serving tradeoff.
- "Fallback provider is also 429-ing." — IC should enable degraded canned mode.
- "The scope-boundary telemetry shows one EU session being served from us-east-1." — IC
  should open a sub-incident for the residency violation.

## Grading
Use `programmes/grading/tabletop.md`. Pass is ≥ 14/20 on the rubric.
```

### 2.3 Quarterly structured walkthrough fixture

```markdown
# Walkthrough fixture — ISCP end-to-end

**Tier:** structured walkthrough. **Duration:** 2h. **Cadence:** quarterly.

## Pre-reading (sent 48h before)
- The current ISCP (exercise 01).
- The current regional-outage playbook (exercise 02).
- The current state-reconstitution plan (exercise 03).
- The residency-failover design (exercise 04).

## Session structure
1. **(20 min) Plan currency check.** Plan owner walks through:
   - Date of last plan update.
   - Date of last approval signature.
   - Outstanding triggered-review events.
   - Open plan action items.
2. **(40 min) Role and authority walk.** Each role named in the plans is present or
   has a named deputy present. The facilitator asks each role to describe their
   decision authority, the escalation path, and the one piece of evidence they need
   to execute (e.g., "what do you need to see to authorise the fallback pin?").
3. **(30 min) Scenario walk.** One scenario from the monthly tabletop menu is walked
   end-to-end in detail, with the plans open.
4. **(20 min) Gap discussion.** Facilitator lists every issue surfaced; scribe
   captures as action items with owners.
5. **(10 min) Close + assign updates.** Action items are added to `programmes/history/
   walkthrough-<YYYY-MM-DD>.md`.

## Pass criteria
- All roles named in the plans are present or represented.
- Every action item has a named owner and a due date.
- The ISCP's last-reviewed date is within the policy cadence.
- No scenario walk surfaces an un-resolvable blocker (if one does, it is a P0 plan
  gap and the walkthrough is graded as fail).
```

### 2.4 Quarterly simulation fixture

```markdown
# Simulation fixture — chaos drill, one component per quarter

**Tier:** simulation. **Duration:** 2h in staging. **Cadence:** quarterly (rotating).

## Rotating component menu
- Q1: retrieval-index primary drop in staging.
- Q2: session-store primary drop in staging.
- Q3: judge-pipeline worker pool kill.
- Q4: hosted-model provider forced 429 on the `canary` tag.

## Session structure
1. **(15 min) Pre-brief.** Facilitator confirms staging environment is up and
   synthetic traffic is running. IC and on-call engineers are at the keyboard.
2. **(10 min) Baseline capture.** Record pre-event metrics as the "healthy baseline".
3. **(60 min) Fault injection and recovery.** Facilitator injects the fault; team
   responds using the plans and runbooks. Facilitator does not help.
4. **(20 min) Judge-pipeline validation.** Confirm groundedness and other quality
   signals return to baseline after serving is recovered (this is the AI-specific
   stop condition).
5. **(15 min) Debrief and action items.**

## Pass criteria
- Serving metrics recover to baseline within RTO(serving) for staging.
- Quality metrics recover within RTO(quality) for staging.
- No runbook referenced by the plan is found to be broken, stale, or missing.
- Fixtures are back-fed per `mod-108/exercise-05` for any surprise.
```

### 2.5 Semi-annual parallel-test fixture

```markdown
# Parallel-test fixture — standby region shadow-serving

**Tier:** parallel. **Duration:** 4h production-adjacent. **Cadence:** semi-annual.

## Scenario
Shadow-route a copy of production traffic to the standby region. The copy is a
tee (shadow), not a cutover — production remains primary. Standby serves the
shadow and the quality-gate pipeline compares outputs between primary and standby
for divergence.

## Pre-conditions
- Shadow-routing capability is deployed (per `mod-111/exercise-03`).
- Standby region is at its daily-pre-flight-green state.
- Change-approval ticket is open.
- Customer-visible incident channel has a "DR drill in progress" entry.

## Beats
1. **(15 min)** Facilitator confirms pre-conditions.
2. **(30 min)** Shadow-routing enabled at 10% of traffic.
3. **(60 min)** Ramp to 25%, 50%, 100% of a chosen cohort (e.g., `scope=us-general`
   anonymous flows); observe standby behaviour under real load. Primary continues
   serving the user-visible response.
4. **(90 min)** Observe divergence: eval-gate compares shadow output vs. primary
   output for the sampled set; log divergence rate.
5. **(20 min)** Ramp down shadow.
6. **(15 min)** Debrief.

## Pass criteria
- Standby carries 100% shadow traffic without saturation (p99 TTFT within SLO).
- Divergence rate (shadow vs. primary) is within the known model-variance envelope
  (< 5% substantive divergence per the judge pipeline).
- No standby-region alert fires beyond the pre-known "drill in progress" channel.
- Cost of the drill is within the budget line in the programme's §5 "Cost model".

## Abort conditions
- Any SLO breach on primary attributable to the drill (should be impossible; shadow
  is not on the user-visible path).
- Standby saturation that would break a real failover.
- Divergence rate > 20% (indicates standby is serving a different model or has a
  retrieval regression).
```

### 2.6 Annual full-interruption fixture

```markdown
# Full-interruption fixture — annual regional failover window

**Tier:** full-interruption. **Duration:** 2h announced window. **Cadence:** annual.

## Scenario
Real cutover from primary region to standby region. User traffic is served by the
standby for a scheduled window.

## Pre-conditions (any missing is abort)
- Last two quarterly simulations passed.
- Last semi-annual parallel test passed within tolerance.
- Change-approval ticket is open at least 14 days in advance.
- Customer notice published at least 7 days in advance.
- Vendor Liaison has confirmed fallback-provider capacity for the window.
- SLO budget is not already distressed on the week of the drill.
- An oncall rotation is on standby with abort authority.

## Beats
1. **(T-30 min)** Pre-flight: all §0 checks from the playbook, explicitly logged.
2. **(T+0 min)** DNS flip to standby. Playbook §2 executed live.
3. **(T+10 min)** Confirm RTO(serving) met in standby.
4. **(T+45 min)** Judge pipeline confirmed; RTO(quality) met.
5. **(T+60 min)** Hold on standby for 30 min; monitor.
6. **(T+90 min)** Scheduled return-to-primary (`playbooks/return-to-primary.md`).
7. **(T+120 min)** Window closes; post-drill debrief.

## Pass criteria
- User-visible error rate during drill stays within the drill's declared SLO (which
  may be slightly loosened from normal; agreed with product).
- RTO(serving) and RTO(quality) both met within window.
- No customer complaint tied to the drill.
- Return-to-primary completed within the window.
- Audit record filed (feeds into `mod-113/exercise-06`).

## Abort conditions
- Any pre-condition fails at T-30.
- Standby does not reach RTO(serving) within 2× stated RTO.
- A real incident declared in another part of the system.
- Vendor Liaison reports fallback-provider degradation.
```

### 2.7 Grading templates

```markdown
# programmes/grading/tabletop.md

Score each dimension 0 / 1 / 2 on the IC under test. Pass at ≥ 14/20, no zero in
dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Plan activation correct *(hard gate)* | Activates on wrong signal / fails to activate | Activates late | Activates on documented criteria |
| 2 | Authority use *(hard gate)* | Takes action outside own authority or hesitates | Asks for concurrence on everything | Acts within authority; escalates when policy requires |
| 3 | Runbook / playbook use | Does not open the right document | Opens correct document, improvises steps | Follows document; notes deviations |
| 4 | Quality-track recognition *(hard gate)* | Does not consider quality recovery | Mentions | Separates quality recovery as its own track |
| 5 | Provider-path handling | Ignores | Handles only if facilitator cues | Addresses provider path proactively |
| 6 | Comms cadence | Silent | Updates on request | Updates on documented cadence |
| 7 | Role handoffs | Confused | Correct with hints | Clean |
| 8 | Scope-boundary respect (where applicable) | Violates | Nearly violates | Enforces |
| 9 | Red-team twist handling | Panics or ignores | Handles slowly | Handles cleanly |
| 10 | Debrief contributions | Blame-shifted | Narrates events | Identifies systemic action items |
```

Similar grading templates exist for the other tiers (walkthrough, simulation, parallel,
full-interruption) with tier-appropriate dimensions. The parallel and full-interruption
templates include capacity and cost dimensions.

### 2.8 Programme document (`programmes/dr-exercise-programme.md`)

```markdown
# chatcat DR Exercise Programme

**Programme version:** 2026-11-13. **Owner:** AI SRE Director.

## 1. Purpose
This programme operationalises SP 800-34 §3.6 ("Ensure plan testing, training, and
exercises") for the chatcat workload.

## 2. Tier cadence
See §1.1 above.

## 3. Training programme (SP 800-34 §3.6.1)
- New-oncall orientation: 2-hour session covering the ISCP, the playbook, and a
  ride-along with an active oncall IC. Completion required before an engineer
  takes the oncall rotation.
- Refresher: annual, before the annual full-interruption drill.

## 4. Training content by role
- **Oncall IC:** full exercise programme + all plans.
- **Vendor Liaison:** `mod-107` register + provider contract summary + fallback-
  provider pin procedure.
- **ML Platform lead:** judge-pipeline reconstitution + state-reconstitution plan
  (exercise 03).
- **Data Platform oncall:** retrieval-index snapshot + replica-promotion procedures.
- **Executive stakeholder:** abbreviated briefing + annual drill observation.

## 5. Cost model
The programme's cost line item lives in the AI SRE budget. Illustrative:

| Tier | Annual count | Cost per event | Annual cost |
|------|--------------|----------------|-------------|
| Tabletop | 12 | $X | $12X |
| Walkthrough | 4 | $Y | $4Y |
| Simulation | 4 | $Z (staging + chaos engineering time) | $4Z |
| Parallel | 2 | $W (capacity + attention) | $2W |
| Full-interruption | 1 | $V (highest) | $V |

Where X ≪ Y ≪ Z ≪ W ≪ V. The ratio is roughly 1:5:20:100:400 from monthly tabletop
to annual full-interruption; the 400× ratio is what makes "tier 5 annually" the
right cadence, not "all tiers quarterly".

## 6. Programme health metrics
- `programme_tier_last_run_days` per tier — must be below the tier's SLO.
- `programme_pass_rate` — rolling 12-month pass rate per tier.
- `programme_action_items_open` — open action items from drills, by age.
- `programme_runbook_broken_count` — runbooks found broken during drills in the last
  quarter; non-zero is an alert.

## 7. Audit evidence
The programme produces the artifacts `mod-113/exercise-06` expects:
- Dated drill records in `programmes/history/`.
- Rubric scores per drill.
- Action-item closure records.
- Training completion records per named role.

## 8. Change log
| Date | Author | Change |
|------|--------|--------|
| 2026-11-13 | AI SRE | Initial publication |
```

## 3. Validation steps

1. **Programme completeness check.** Confirm one artifact exists per tier: the twelve
   monthly tabletop fixtures, the four quarterly walkthrough + simulation fixtures,
   the two parallel fixtures, and the one full-interruption fixture. The programme
   document must enumerate all of them.
2. **Grading templates present.** One grading template per tier; dimensions sized for
   the tier.
3. **History log growing.** `programmes/history/` has one dated entry per drill run
   in the last twelve months. A gap in the history is a programme failure.
4. **Action-item closure.** For each drill in the history, the open action items are
   closed within one review cycle (quarter) or are escalated with a documented reason.
5. **Full-interruption pre-conditions.** Before the annual full-interruption drill
   fires, the pre-condition checklist in §2.6 is logged. Running the drill with any
   pre-condition failing is a programme failure.
6. **Cross-reference check.** Each scenario names the plan(s) it exercises; every
   plan from exercises 01–04 is exercised by at least one scenario.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Five-tier structure *(hard gate)* | Missing | Mentioned | All five tiers present with distinct cadence |
| 2 | Cadence justified *(hard gate)* | Uniform or absent | Non-uniform but not justified | Cadence reasoned by cost and value |
| 3 | Rotating scenario menu *(hard gate)* | Single scenario | 3-5 scenarios | 12-month menu covering fault taxonomy |
| 4 | AI-specific scenarios | None | 1-2 | ≥ 4 AI-specific (provider version bump, judge outage, residency, agent state) |
| 5 | Pre-conditions for expensive tiers | None | Mentioned | Explicit, must-log pre-conditions for parallel + full-interruption |
| 6 | Grading templates *(hard gate)* | Missing | One tier only | Per-tier rubric with tier-appropriate dimensions |
| 7 | Training programme | Missing | Mentioned | Role-specific training + refresher cadence |
| 8 | Cost model | None | Qualitative | Quantitative ratios and annual cost line |
| 9 | Programme health metrics | None | 1-2 | ≥ 4 metrics including runbook-broken counter |
| 10 | Audit evidence integration | None | Mentioned | History log + rubric scores + action-item closure feeding exercise 06 |

## 5. Common mistakes

- **"Tier 5 annually" alone.** Running the full-interruption without the cheaper
  tiers leading up to it produces an under-prepared team and a drill that either
  fails or gets quietly re-cast as "educational" so it can "pass".
- **Fixed scenario every cycle.** Running the regional-outage scenario every quarter
  produces an IC team that knows regional-outage well and does not know
  retrieval-corruption, provider version bump, or scope-boundary violations.
- **No pre-condition abort for high tiers.** A full-interruption drill that fires
  the week an SLO is already distressed turns a planned drill into a real
  incident. The pre-conditions in §2.6 are not boilerplate.
- **No red-team twist.** Tabletops without an injected surprise become a
  recitation of the plan and don't exercise the IC's decision-making.
- **Programme is a document, not evidence.** The audit expects dated history and
  closed action items, not a programme document that describes an intent. §7 of
  the programme document explicitly names the artifacts that must exist as evidence.
- **No judge-pipeline exit criterion.** A simulation that "recovers serving" and
  declares pass without checking the quality track skips the AI-specific half of
  recovery.
- **Executive briefing collapsed into the IC training.** Executives need an
  abbreviated briefing and an annual observation slot; they do not need the IC
  training content.

## 6. References

- NIST SP 800-34 Rev. 1 §3.6 — the testing, training, and exercises framework this
  programme implements. §3.6.2 enumerates the five tiers (tabletop, structured
  walkthrough, simulation, parallel, full-interruption).
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the MEASURE function ("MEASURE 1: AI systems are tested") and the MANAGE function ("MANAGE 4: AI risks and benefits are managed") grounds the dual-RTO exit criteria in the simulation and parallel tiers.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML supply chain` motivates the provider-focused scenario (Feb, May, Dec); `ML09: output integrity attack` motivates the judge-pipeline exit criteria across the programme.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the ML-inference-API-access and ML-supply-chain technique classes drive the November "cyber-triggers-ISC" scenario, which is the one where the exercise programme exercises coordination with the IR plan.
- `mod-108/exercise-05` regression-fixture back-feed — the mechanism by which simulation surprises become fixtures.
- `mod-108/exercise-06` legal-compliance handoff — triggered by the November scenario.
- `mod-113/exercise-01` ISCP — the plan the programme exercises.
- `mod-113/exercise-02` regional-outage playbook — the plan the tabletops drill against.
- `mod-113/exercise-03` state-reconstitution plan — the plan the July scenario drills.
- `mod-113/exercise-04` residency-constrained failover — the plan the August scenario drills.
- `mod-113/exercise-06` audit artifact pack — the destination for the programme's history, scores, and action items.
