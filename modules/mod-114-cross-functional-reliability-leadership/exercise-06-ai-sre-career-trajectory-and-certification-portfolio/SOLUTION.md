# Exercise 06 — AI-SRE Career Trajectory and Certification Portfolio — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-06-ai-sre-career-trajectory-and-certification-portfolio.md`.

> **Scope note.** The paired exercise asks the learner to author their own *trajectory artefact*: the plan that describes where they are, where they are going, what evidence they are accruing, and what external credentials fit. The deliverables are (a) a self-assessment against a published AI-SRE skill matrix, (b) a portfolio ledger of evidence tied to the modules in this track, (c) a certification-portfolio plan (what, when, why), and (d) a quarterly-review ritual that keeps the trajectory honest. The reference solution is a worked example filled in for a fictitious mid-level engineer; the **template is the deliverable**, and a learner's output is the same template filled in for themselves.

## 1. Solution overview

Career planning for AI-SRE suffers the same failure modes as capacity planning:

1. **No baseline.** "I want to be a senior AI-SRE" with no honest statement of current level produces a plan that cannot be tracked.
2. **Certification collecting.** Stacks of credentials without applied evidence read as résumé-padding to any experienced hiring manager; they also don't develop the judgment the certifications are a proxy for.
3. **Evidence not written down.** The engineer led two incidents, designed a release-gate contract, and ran four SLO reviews — and none of it is in a form that transfers to a new role, a promotion packet, or a résumé.
4. **No review cadence.** The plan is written, filed, forgotten, and is a year stale by the time of the next conversation.

The reference solution is a **portfolio** anchored on three things:

1. **An AI-SRE skill matrix** that names the capability bands and the evidence that supports each band.
2. **A portfolio ledger** that records real artefacts (incident posts, contracts authored, SLO ledgers run, RFCs landed) tied to modules in this track.
3. **A certification plan** where every credential is justified against a gap in the matrix, with a cost / time / expected-benefit estimate — and a null option (don't take it) as a legitimate choice.
4. **A quarterly review ritual** (named in exercise 01's program) where manager + engineer walk the ledger and update the matrix.

### 1.1 Design principles

- **Evidence beats credentials.** A credential is a proxy; an artefact is the thing itself. The portfolio puts artefacts first.
- **Credentials are instrumental, not aesthetic.** Every credential in the plan names the matrix gap it fills, the cost, and the kill criterion (don't bother if…). Credentials without a named gap are deferred.
- **Null option is legitimate.** "Don't take any certification this quarter; ship exercise X instead" is a valid trajectory move.
- **The ledger is honest.** Draft artefacts and co-authored artefacts are noted as such; the engineer owns the entry but the manager confirms the shape (co-author, author, approver, reviewer).
- **Reviews happen whether or not promotion is in scope.** A promotion conversation one quarter is not a reason to skip the review next quarter.
- **Framework literacy is recorded explicitly.** Familiarity with NIST AI RMF, OWASP ML Top 10, and MITRE ATLAS is a skill band in its own right, not a bullet point on the résumé.

### 1.2 AI-SRE skill matrix (bands)

The matrix below is the self-assessment surface. It intentionally mirrors the module structure of this track so that evidence from the modules maps cleanly to bands. Bands are not seniority levels; they are capability areas. A given engineer's *level* across these bands is what seniority describes.

| Band | What it covers | Evidence shape | Related modules |
|---|---|---|---|
| **B1 — SLO engineering** | SLI selection, SLO target negotiation, error-budget policy authorship, burn-rate alerting | Owned SLOs; SLO ledger entries; a budget policy they authored | 103, 104 |
| **B2 — Observability for AI** | LLM + GPU workload observability; cost telemetry; eval instrumentation | Alert catalogue entries; dashboards; an instrumentation RFC | 104, 112 |
| **B3 — Non-deterministic diagnosis** | Failure diagnosis on model-backed services; eval-signal interpretation | Incident posts where cause class was eval or drift | 105 |
| **B4 — Capacity & cost** | GPU / token forecasting; reservation strategy; cost correctness | A capacity plan or procurement memo they authored | 106 |
| **B5 — Vendor / provider risk** | Vendor risk register; fallback design; contract-term negotiation | Vendor evaluations; fallback activations | 107 |
| **B6 — Incident response** | Incident command; blameless post-mortem; incident ledger discipline | Incident commander posts; post-mortems where they were PM | 108, 110 |
| **B7 — Chaos / DR / resilience** | Chaos experiments; DR rehearsal design; multi-region failover | Written chaos reports; DR rehearsal artefacts | 109, 113 |
| **B8 — Security / safety crossover** | Guardrail SLOs; reliability-gate vs safety-gate reconciliation; PII in incidents | Jointly authored incident + a safety-gate reconciliation memo | 110 |
| **B9 — Release engineering** | Progressive delivery; release-gate contract; DORA on model releases | A release-gate contract or DORA-adapted pipeline | 111 |
| **B10 — Data / config pipelines** | Reliability of feature stores, index rebuilds, prompt stores | A reliability analysis on a pipeline surface | 112 |
| **B11 — Leadership / program** | Ritual authoring; peer contracts; upward comms; build-vs-buy; facilitation | Charters, contracts, memos, facilitated meetings | 114 |
| **B12 — Framework literacy** | NIST AI RMF, OWASP ML Top 10, MITRE ATLAS, EU AI Act article hooks | Citations in authored artefacts; workshop attendance; tabletop drills | 110, 114 |

Each band is scored:
- **0** — no evidence.
- **1** — Ic-shaped evidence (contributed under guidance).
- **2** — Owner-shaped evidence (authored and ran).
- **3** — Teach-shaped evidence (ran a workshop, mentored a peer, authored a durable artefact others now use).

## 2. Implementation — worked answer

### 2.1 Portfolio template — `portfolio/README.md`

```markdown
# AI-SRE portfolio — <engineer name>

**Current role.** <company> · <title>
**Manager.** <name>
**Portfolio opened.** <YYYY-MM-DD>
**Last reviewed.** <YYYY-MM-DD> (every quarter)

## Contents

- `self-assessment.md` — current band scores with evidence
- `ledger.md` — append-only evidence ledger
- `certifications.md` — plan + status of external credentials
- `development-plan.md` — next-quarter focus bands + milestones
- `review-notes/YYYY-QN.md` — one file per quarterly review
```

### 2.2 Worked self-assessment — `portfolio/self-assessment.md`

Fictitious engineer "J. Patel", 3 years as an SRE, 9 months in an AI-SRE seat. Filled in at the top of a quarter.

```markdown
# Self-assessment — J. Patel — 2026 Q4

**Rule for scoring.** 0 = no evidence; 1 = contributed; 2 = owned; 3 = taught.
Only the evidence in `ledger.md` is admissible; every score links to the ledger entry.

| Band | Score | Key evidence (ledger ref) | Target next review |
|---|---|---|---|
| B1 SLO engineering | 2 | L-012, L-023 | 2 |
| B2 Observability for AI | 2 | L-008, L-015 | 2 |
| B3 Non-det diagnosis | 1 | L-019 | 2 |
| B4 Capacity & cost | 1 | L-021 (co-author) | 2 |
| B5 Vendor / provider risk | 1 | L-026 | 1 |
| B6 Incident response | 2 | L-003, L-014, L-025 (IC) | 2 |
| B7 Chaos / DR | 0 | — | 1 |
| B8 Security / safety crossover | 1 | L-018 | 2 |
| B9 Release engineering | 2 | L-029 (release-gate contract) | 2 |
| B10 Data / config pipelines | 0 | — | 1 |
| B11 Leadership / program | 1 | L-030 (co-chaired SLO review) | 2 |
| B12 Framework literacy | 2 | L-004, L-022 | 2 |

## Narrative (one paragraph per band moving this quarter)

- **B3 — Non-det diagnosis.** One incident this quarter (L-019) where the
  cause class was eval; I was second-chair. To move to 2 I need to be
  primary on two eval-cause incidents. Realistic by end of quarter if the
  rotation puts me on nights for the eval re-baseline.
- **B4 — Capacity & cost.** Co-authored the GPU reservation memo for Q1
  (L-021). To move to 2 I need to author a capacity plan solo; I will
  volunteer for the retrieval-cluster scoping that fell out of the Nov 11
  incident.
- **B7 — Chaos / DR.** No evidence. I will partner with the DR owner for
  the December full-failover rehearsal and author the report.
- **B8 — Security / safety crossover.** Co-authored an incident where PII
  handling discipline was the finding (L-018). Target: author a safety-
  gate reconciliation memo with the security peer next quarter.
- **B10 — Data / config pipelines.** No evidence. Opportunity: the index-
  rebuild schedule memo the Nov 11 incident surfaced.
- **B11 — Leadership / program.** Co-chaired an SLO review (L-030). To
  move to 2 I need to chair the SLO review solo for a quarter; my
  manager is slotting me into the rotation.
```

### 2.3 Worked ledger — `portfolio/ledger.md`

An append-only list. Each entry has a stable id (`L-NNN`), a date, an artefact type, a shape (author / co-author / contributor / reviewer), and a link. Example rows:

```markdown
| ID | Date | Artefact | Shape | Band(s) | Link |
|---|---|---|---|---|---|
| L-003 | 2026-06-14 | Incident post-mortem — provider error spike | IC | B6 | <link> |
| L-004 | 2026-06-30 | OWASP ML Top 10 tabletop drill | attendee | B12 | <link> |
| L-008 | 2026-07-18 | Dashboard: chatcat GPU utilisation + cost | author | B2 | <link> |
| L-012 | 2026-07-28 | SLO proposal: `legal-help` TTFT tightening | author | B1 | <link> |
| L-014 | 2026-08-21 | Incident post-mortem — eval-cause regression | IC (co) | B3, B6 | <link> |
| L-015 | 2026-09-02 | Instrumentation RFC: eval signals Prom namespace | author | B2 | <link> |
| L-018 | 2026-09-28 | Joint incident post-mortem — PII in logs | co-author | B6, B8 | <link> |
| L-019 | 2026-10-10 | Incident post-mortem — eval drift on `faq` | second-chair | B3 | <link> |
| L-021 | 2026-10-20 | Capacity memo: Q1 2027 GPU reservation | co-author | B4 | <link> |
| L-022 | 2026-10-25 | NIST AI RMF function-mapping memo | author | B12 | <link> |
| L-023 | 2026-10-28 | Error-budget policy revision | author | B1 | <link> |
| L-025 | 2026-11-11 | Incident IC — retrieval + index overlap | IC | B6 | <link> |
| L-026 | 2026-11-12 | Vendor-risk review note: provider X breach | contributor | B5 | <link> |
| L-029 | 2026-11-13 | Release-gate contract draft — chatcat v16 | author | B9 | <link> |
| L-030 | 2026-11-14 | SLO review — co-chaired | co-chair | B11 | <link> |
```

### 2.4 Worked certification plan — `portfolio/certifications.md`

For each credential, the plan records: the matrix gap, the official source URL (so the engineer can verify the current syllabus), the cost estimate, the time estimate, the expected quarter, the kill criterion. The plan deliberately includes a "do nothing" option as a first-class item.

```markdown
# Certification plan — J. Patel — 2026 Q4 → 2027 Q4

Credentials are instrumental. Every row names the matrix gap it fills and
has a kill criterion (when not to take it). "Do nothing" is a legitimate row.

## Active candidates

### 1) Google Professional Cloud DevOps Engineer

- **Fills.** B1 SLO engineering (reinforces the Google SRE idiom), B9
  release engineering (continuous delivery), B11 program (post-mortem
  culture).
- **Official source.** `https://cloud.google.com/learn/certification/cloud-devops-engineer`
- **Current syllabus.** Verify at the above URL at the time of booking.
- **Cost / time estimate.** Sticker $200; preparation ~30 h over 6 weeks.
  Verify current price on the certification page at booking time — the
  $200 figure is the long-standing list price but should be reconfirmed
  against the official page before registering.
- **Expected quarter.** 2027 Q1.
- **Kill criterion.** If I have authored a release-gate contract that is
  in production and have run two SLO reviews solo by 2027 Q1, I may defer
  this; the band it fills will already be at 2.

### 2) Certified Kubernetes Administrator (CKA)

- **Fills.** B2 observability (operational Kubernetes), B9 release
  (progressive-delivery mechanics), B7 chaos / DR (platform resilience).
- **Official source.** `https://www.cncf.io/training/certification/cka/`
- **Cost / time estimate.** Sticker ~$395 per the CNCF page; preparation
  ~60 h over 10 weeks for someone with prior Kubernetes exposure.
  Reconfirm the current CKA price on the CNCF page at booking time.
- **Expected quarter.** 2027 Q2.
- **Kill criterion.** If my team outsources all cluster administration to
  a managed-platform peer (effectively, if B2 is covered by their team),
  deprioritise.

### 3) Certified Kubernetes Security Specialist (CKS)

- **Fills.** B8 security / safety crossover (platform-side).
- **Official source.** `https://www.cncf.io/training/certification/cks/`
- **Prereq.** CKA.
- **Expected quarter.** 2027 Q3 (contingent on CKA).

### 4) NIST AI RMF workshop / training

- **Fills.** B12 framework literacy.
- **Official source.** `https://www.nist.gov/itl/ai-risk-management-framework` — NIST maintains the framework and references training; verify current training offerings there. The framework itself is free, but the specific workshop / training market around it is maturing, so confirm which offerings NIST currently endorses or lists at the time of registration.
- **Expected quarter.** 2027 Q2 — attend a workshop.
- **Kill criterion.** If I have authored two function-mapping memos and
  run the AI RMF portion of an incident learning review, the band is at
  2 without workshop attendance.

## Deferred (reviewed next quarter, not scheduled)

- **Cloud provider ML specialty certifications** — available from each major cloud provider; defer until my role touches that provider's AI control-plane in production. Specific provider-ML-specialty cert names, prices, and syllabi change frequently; confirm current offerings on each provider's certification page at the time the role moves in that direction.

## Not planned (explicitly)

- **Any certification that does not fill a named band gap.** I am not
  planning to take credentials for CV decoration.
- **Any credential that requires training > 100 h without a career move
  that uses it.** The time trades against ledger artefacts, and the
  ledger wins.

## Review cadence

Reviewed at the quarterly career & certification review (exercise 01).
Rows move among Active / Deferred / Not planned; new rows require a
named band gap.
```

### 2.5 Worked development plan — `portfolio/development-plan.md`

```markdown
# Development plan — J. Patel — 2026 Q4

**Top-line.** Focus bands this quarter: B3, B4, B7, B10. Everything else
holds. Reviewed at the quarterly review in 2027 Q1.

## Band goals

1. **B3 Non-det diagnosis → 2.** Act as primary IC on at least one
   eval-cause incident. If no eval-cause incident fires, lead a drift
   root-cause write-up on a near-miss (L-019 follow-up).
2. **B4 Capacity & cost → 2.** Author the retrieval-cluster capacity memo
   solo, feeding the next monthly capacity review.
3. **B7 Chaos / DR → 1.** Partner on the December full-failover
   rehearsal; co-author the report.
4. **B10 Data / config pipelines → 1.** Author the index-rebuild
   scheduling memo that fell out of the Nov 11 incident.

## Non-band goals

- Attend 2027 Q1 OWASP ML Top 10 workshop (B12 reinforcement).
- Defer Google Cloud DevOps cert until the rotation lands in 2027 Q1.

## Risks to the plan

- No eval-cause incident fires: mitigated by taking the drift root-cause
  write-up path.
- Index-rebuild memo is assigned to MLOps: raise with manager; may own
  the SRE-side section instead of the whole memo (would score 1, not 2).

## How my manager helps

- Puts me on the SLO-review rotation to let B11 move next quarter.
- Reviews the capacity memo before it goes to the monthly review.
```

### 2.6 Quarterly review template — `portfolio/review-notes/YYYY-QN.md`

```markdown
# Quarterly review — <YYYY> Q<N>

**Engineer.** <name>  **Manager.** <name>  **Date.** <YYYY-MM-DD>

## Band score deltas since last review

| Band | Prev | Now | Evidence |
|---|---|---|---|
| ... |

## What landed (from last development plan)

- <landed> — link to ledger entry
- <did not land> — reason, next step

## Next quarter development plan

- Top-line focus bands:
- Band goals:
- Non-band goals:
- Risks:

## Certification plan diff

- Added:
- Moved to deferred:
- Dropped:

## Open questions for the next review

- ...

## Signatures

- Engineer:
- Manager:
```

## 3. Validation steps

1. **Baseline check.** Every band in the matrix has a score; every non-zero score has a ledger reference. A band scored without a ledger entry is a draft, not a self-assessment.
2. **Shape honesty.** The ledger records *shape* (author / co-author / contributor / reviewer / IC / attendee), not just a title. A review walks the shape column and flags any entry that is suspicious (an engineer who co-authored but recorded "author" should be corrected in the next review, not disputed in the moment).
3. **Credential-to-gap mapping.** Every planned credential names at least one band it fills. A credential with no named gap is moved to Deferred.
4. **Null option visible.** The certification plan has a "Not planned (explicitly)" section with at least one entry. A plan with only active credentials is probably CV-driven.
5. **Verify-at-booking rule.** Every cost / syllabus claim about an external credential is accompanied by the official source URL and an explicit "verify at booking / registration" note where the specific number is unverified at write time. That inline caveat is a feature, not a defect: credential offerings change, and it is honest to flag when a specific number should be reverified.
6. **Development-plan feasibility.** Each band goal names a *mechanism* (an upcoming incident rotation, a memo, a review) rather than a wish ("I will become good at capacity"). Mechanisms failing is OK; wishes failing is not actionable.
7. **Review cadence enforced.** The quarterly review ritual from exercise 01 is the same review that updates this portfolio. A portfolio with no quarterly review file for the previous two quarters is stale.
8. **Manager sign-off on evidence shape.** The manager confirms (or corrects) the shape column at each review. Self-assessment without manager confirmation drifts toward overclaiming within two cycles.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 16 / 22 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Skill matrix *(hard gate)* | Missing | Bands named but no scoring rubric | Bands + evidence-based scoring rubric |
| 2 | Ledger of evidence *(hard gate)* | Missing | Exists but no shape column | Append-only, shape recorded, module-linked |
| 3 | Current baseline | Overclaimed | Mixed | Every score linked to ledger entry |
| 4 | Certification plan ties to gap *(hard gate)* | Credential list without justification | Some tied to gaps | Every credential tied to a band gap with kill criterion |
| 5 | Null option present | Missing | Mentioned | Explicit "Not planned" section |
| 6 | Credential claims verifiable | Prices / syllabus asserted without sources | Official URLs provided | Official URLs + explicit verify-at-booking caveats on moving numbers |
| 7 | Development plan has mechanisms | Wishes | Mixed | Each goal names an upcoming artefact, rotation, or memo |
| 8 | Quarterly review cadence | Not referenced | Mentioned | Template + prior-review files + manager sign-off |
| 9 | Framework literacy recorded | Missing | Mentioned | Band B12 with specific artefacts |
| 10 | Credential cost / time not inflated | Overstated | Mixed | Honest estimate + verify-at-booking note |
| 11 | Manager-engineer co-ownership | Engineer-only | Manager named | Manager sign-off on evidence shape |

## 5. Common mistakes

- **Credential collecting.** Stacking credentials without named gap mapping reads as padding to experienced reviewers and does not develop the judgment the credentials proxy for.
- **Overclaiming shape.** Marking a co-authored artefact as "author" in the ledger. Short-term flattering, medium-term corrosive to credibility when the manager or a peer notices.
- **Evidence not written.** The engineer led two incidents and authored a contract this quarter and none of it is in the ledger. By next quarter's review, "what did you do" has no durable answer.
- **No null option.** A certification plan with every row Active implies all credentials are necessary. They are not; "Not planned" is where judgment is visible.
- **Prices quoted as if fixed.** Credential prices and syllabi change. Quote the official source, and add an explicit verify-at-booking caveat next to any number you cannot confirm at write time.
- **Wishes instead of mechanisms.** "I will become a stronger IC" is a wish. "I will act as primary IC on the next eval-cause incident, or author the drift root-cause write-up if none fires" is a mechanism.
- **Skipping the review the quarter the plan matters most.** When the plan is behind, the review is the thing that catches it. Skipping only compounds the gap.
- **Framework literacy as a résumé line.** Citing NIST AI RMF or OWASP ML Top 10 in bio text does not demonstrate literacy. Function-mapping memos and tabletop drills do.
- **Portfolio reads like a CV.** A portfolio is for the engineer's trajectory, not for external audiences. The CV is downstream; the portfolio is upstream.

## 6. References

- Google SRE book, *Making Work Visible* (practitioner-themed sections of `sre.google/sre-book/`) — [sre.google/sre-book/](https://sre.google/sre-book/) — the discipline of writing work down (incident posts, post-mortems, SLO ledgers) is what the portfolio ledger formalises for career evidence.
- Google SRE workbook, *On-Call* — [sre.google/workbook/on-call/](https://sre.google/workbook/on-call/) — the skill bands B3 / B6 inherit their shape (primary / secondary / IC) from the workbook's on-call framing.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — framework literacy (B12) requires function-mapping memos that cite specific controls; NIST maintains the canonical text and crosswalks.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — framework literacy (B12) requires the engineer to recognise each OWASP ML Top 10 class in incident retrospectives.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-ML threat catalogue; framework literacy (B12) requires the engineer to be able to map an incident cause to an ATLAS technique when applicable.
- Google Cloud Certification — Professional Cloud DevOps Engineer — [cloud.google.com/learn/certification/cloud-devops-engineer](https://cloud.google.com/learn/certification/cloud-devops-engineer) — one worked candidate credential; the engineer verifies current syllabus and price at booking time.
- CNCF / Linux Foundation — CKA / CKS — [cncf.io/training/certification/cka/](https://www.cncf.io/training/certification/cka/), [cncf.io/training/certification/cks/](https://www.cncf.io/training/certification/cks/) — two worked candidate credentials; same verify-at-booking rule applies.
