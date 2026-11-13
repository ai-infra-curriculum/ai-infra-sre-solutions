# Exercise 01 — Reliability Program Cadence and Ritual Authoring — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-01-reliability-program-cadence-and-ritual-authoring.md`.

> **Scope note.** The paired exercise asks the learner to design the recurring rituals that make up an AI-SRE team's "reliability program" — the fixed-cadence meetings, reviews, and written artefacts that keep SLOs, incident learning, capacity planning, and cross-team coordination from drifting. The output is a design artefact: a calendar of rituals, a per-ritual charter (purpose, inputs, owner, outputs, exit criteria), and a rubric against which a candidate program can be scored. The reference solution assumes the illustrative `chatcat` AI product used throughout this module.

## 1. Solution overview

A reliability program is a set of rituals that convert *continuous* signals (SLO burn, cost, eval drift, incident load, capacity headroom) into *discrete* decisions (rollout gating, hiring, roadmap trade-offs, vendor replacement). Without a codified cadence, three failure modes dominate:

1. **Signal pile-up.** Burn-rate alerts, eval regressions, and provider-incident tickets all compete for the same "we should talk about this" slot. Discussion happens ad-hoc, late, or on the wrong channel, and the most senior voice wins by volume.
2. **No closed loop on error budget.** Error-budget policy exists on paper, but no ritual *consumes* it. A quarter ends with the budget blown and no written decision about what moved.
3. **Program drift.** A ritual that was useful six months ago (daily standup during an incident push) survives into a quarter where it just reads yesterday's dashboards aloud. Nobody has authority to kill it.

The reference program solves these by defining a **ritual matrix** — a small, stable set of named rituals with explicit charters — plus a **governance layer** (a quarterly ritual retro) that authorises adding, killing, or reshaping rituals.

### 1.1 Design principles

- **Cadence follows signal frequency, not habit.** SLO review is weekly because burn-rate data refreshes in minutes. Capacity planning is monthly because GPU reservations settle in weeks. Career / headcount is quarterly because hiring cycles are quarterly. Rituals faster than their input signal re-read yesterday's data; rituals slower miss the decision window.
- **Every ritual has an owner and an exit criterion.** The owner is accountable for whether the ritual fired, had the right inputs, and produced the written outputs. The exit criterion is what has to be true at the end of the meeting for the time to have been worth taking. A ritual that cannot name its exit criterion is a status update, not a ritual.
- **Every ritual writes a durable artefact.** A decision that is not written in a ledger (SLO ledger, release ledger, incident ledger, program ledger) did not happen. Minutes in a shared doc with a stable path, not a chat message.
- **Error-budget policy drives veto authority, not opinion.** When the budget is blown, the SLO review has the authority to freeze non-reliability work. When the budget is healthy, the roadmap review has the authority to spend it. Which ritual holds which lever is named up front.
- **Cross-functional seats are named, not optional.** Eval, ML platform, inference performance, security, and product each have a *named* seat at the rituals that need them. "Whoever can make it" produces drift.
- **Quarterly ritual retro.** Every quarter one ritual decides which rituals survive, change, or die. Rituals are deleted on a default-off rule: any ritual with < 50 % attendance or no written artefact in the last two cycles is killed unless its owner defends it.

### 1.2 The ritual matrix at a glance

| Ritual | Cadence | Owner | Signal source | Primary output |
|---|---|---|---|---|
| Reliability standup | Daily (15 min) | On-call | Paging dashboard, open incidents, release ledger | Spoken coverage + open-action list |
| SLO review | Weekly (45 min) | SLO owner | SLO burn + error-budget ledger | Written SLO ledger entry; freeze / unfreeze verdict |
| Release & gate review | Weekly (30 min) | Release manager | Release ledger, DORA keys | Go / hold list for the next week's rollouts |
| Incident learning review | Bi-weekly (60 min) | Incident commander rotation | Incident ledger (exercise `mod-108`) | Action items with owners + due dates, linked to post-mortems |
| Capacity & cost review | Monthly (60 min) | Capacity owner | GPU / token / provider spend dashboards | Capacity plan deltas, procurement asks |
| Vendor & provider risk review | Monthly (45 min) | Vendor-risk owner | Provider SLA breaches, vendor-risk ledger (`mod-107`) | Updated vendor-risk register; fallback activation decisions |
| Chaos / DR rehearsal | Monthly or quarterly | DR owner | Rehearsal results (`mod-109`, `mod-113`) | Written rehearsal report; RTO / RPO delta |
| Cross-peer contract review | Quarterly (90 min) | SRE function lead | Peer-contract ledger (exercise 03) | Updated contracts with eval / perf / security / MLPlatform |
| Reliability program retro | Quarterly (60 min) | SRE function lead | Program ledger, ritual attendance + artefact logs | Ritual add / kill / change decisions |
| Career & certification review | Quarterly, per person (30 min) | Manager | Portfolio ledger (exercise 06), individual IDP | Updated IDP, named next milestone |

## 2. Implementation — worked answer

The deliverable is a repo folder `reliability-program/` containing one markdown charter per ritual, a program calendar, and the governing retro charter.

### 2.1 Program calendar — `reliability-program/calendar.md`

```markdown
# chatcat reliability program — calendar

All times US-Pacific; all meetings have a published Google Calendar invite with a
stable link; all meetings have an owner recorded in `charters/`.

## Weekly
- Mon 09:15 — Reliability standup (daily Mon–Fri)
- Tue 11:00 — SLO review (45m)
- Wed 10:00 — Release & gate review (30m)

## Bi-weekly
- Odd Thursdays 14:00 — Incident learning review (60m)

## Monthly
- Second Tuesday 13:00 — Capacity & cost review (60m)
- Third Tuesday 13:00 — Vendor & provider risk review (45m)
- Last Friday 10:00 — Chaos / DR rehearsal (full morning when a full failover;
  60m when a tabletop; see `charters/dr-rehearsal.md` for cadence mix)

## Quarterly
- First Monday of quarter 10:00 — Cross-peer contract review (90m)
- First Thursday of quarter 10:00 — Reliability program retro (60m)
- Per-person, within first three weeks of quarter — Career & certification review (30m)

## Freeze windows
- The two business days before and the day of any marketing-driven launch — no
  non-critical SLO review changes, no non-critical capacity changes. See
  `charters/release-gate-review.md` for the freeze protocol.
```

### 2.2 Sample charter — `reliability-program/charters/slo-review.md`

Every charter in the program follows this template. The SLO-review charter is shown in full below; the other charters in the matrix follow the same shape.

```markdown
# SLO review — charter

**Cadence.** Weekly, Tuesday 11:00 PT, 45 minutes.

**Owner.** SLO owner (named rotation; current owner recorded in
`reliability-program/owners.yaml`). The owner is accountable for the meeting
having run, for the inputs being pre-read, and for the ledger entry being
written and linked from the program ledger.

**Required attendees (named seats).**
- SLO owner (chair)
- On-call for the week
- Eval seat (one named engineer from the eval team)
- Inference-performance seat
- Product seat (one named PM for the surfaces under review)

Any attendee may be out for a week; the seat may not be unfilled for two
consecutive weeks without triggering a program-retro note.

**Inputs (must be pre-read; chair gates the meeting on these existing).**
1. The SLO dashboard snapshot for the past 7 and 28 days.
2. The error-budget ledger entry for the current quarter.
3. Any burn-rate alerts that fired since the previous review, with the paging
   incident links.
4. Any route or product change that would alter the SLO target (new route,
   new regulated class) proposed since the last meeting.

**Agenda (fixed; chair runs to time).**
1. 5 min — Burn since last review, by route class.
2. 15 min — Any route that burned > 25 % of its weekly budget: cause class
   (SLO-only, eval, capacity, provider), containment status, whether to
   extend freeze.
3. 10 min — Error-budget policy decisions: open freezes, candidates to unfreeze,
   non-reliability work that is currently blocked.
4. 10 min — New or changed SLOs for the coming week. Each change requires a
   written proposal and is logged in the SLO ledger.
5. 5 min — Action items + owners + due dates.

**Decision authority.**
- SLO owner may *freeze* non-reliability work on a route whose budget is blown
  for two consecutive weeks. The freeze is written in the SLO ledger and linked
  from the program ledger.
- SLO owner may *unfreeze* a route after two consecutive weeks of green budget
  and a written statement of what changed.
- Product seat may *appeal* a freeze to the SRE function lead; the appeal is
  written and the freeze stays in force until the appeal is decided.

**Exit criteria (the meeting is not done until all are true).**
- Every route that burned > 25 % budget has a named owner for its containment
  action and a due date.
- Every proposed SLO change has been accepted, rejected, or deferred — not left
  open.
- An SLO ledger entry is written and linked from the program ledger within
  24 hours of the meeting.

**Linked artefacts.**
- SLO ledger: `reliability-program/ledgers/slo-ledger.md`
- Error-budget policy: `reliability-program/policies/error-budget.md`
- Burn-rate alert catalogue: `observability/alerts/slo-burn/`

**Why this ritual exists.**
NIST AI RMF `MANAGE-2.3` names "ongoing monitoring of AI risks" as a required
control; `MEASURE-2.1` names "performance against defined metrics". Google
SRE's *Alerting on SLOs* names a weekly budget review as the canonical
rhythm for consuming burn-rate data. This ritual is the single place both
controls are discharged for `chatcat`.
```

### 2.3 Other charters — single-paragraph summaries

The full charter folder contains one file per ritual. Each file has the same six sections (Cadence / Owner / Required attendees / Inputs / Agenda / Decision authority / Exit criteria / Linked artefacts / Why this ritual exists). The key differences from SLO review:

- **Reliability standup** — 15 minutes, daily Mon–Fri. Chair rotates with on-call. Inputs: open incident list, release ledger entries since yesterday, paging dashboard. Exit: every open paging incident has a named owner for the day; every rollout in flight has a go / hold call. No decisions made here; this is a coverage ritual.
- **Release & gate review** — 30 minutes, weekly. Chair is the release manager. Inputs: release ledger entries since last review, open gate failures, DORA four-keys dashboard (produced by `mod-111` exercise-06). Exit: a written go / hold list for the next week's rollouts, linked from the release ledger.
- **Incident learning review** — 60 minutes, bi-weekly. Chair rotates with incident commanders. Inputs: all post-mortems closed since the last review, action-item tracker, any still-open incident-triggered work. Exit: every action item has an owner and a due date; stale (> 2 cycles overdue) items are escalated to the SRE function lead. The ritual explicitly does *not* re-litigate blame.
- **Capacity & cost review** — 60 minutes, monthly. Chair is the capacity owner. Inputs: GPU reservation utilisation, token-spend trend, provider cost breakdown, forecast model from `mod-106`. Exit: a written capacity-plan delta (buy more, reshape, re-contract, stop); any procurement ask has a named approver.
- **Vendor & provider risk review** — 45 minutes, monthly. Chair is the vendor-risk owner. Inputs: provider SLA breaches since last review, vendor-risk register (from `mod-107`), active and pending contracts. Exit: an updated risk register and a list of fallback activations (or de-activations) with written reason.
- **Chaos / DR rehearsal** — Monthly (tabletop) and quarterly (full failover); see `mod-109` and `mod-113` for mechanics. Exit: a written rehearsal report with observed RTO / RPO and the delta from target; any delta is a tracked action item.
- **Cross-peer contract review** — 90 minutes, quarterly. Chair is the SRE function lead. Inputs: current peer contracts (exercise 03), any contract violations observed in the quarter, any new peer surface added. Exit: every contract has been re-affirmed, re-signed, or updated; contract versions are bumped and linked from the program ledger.
- **Reliability program retro** — 60 minutes, quarterly. Chair is the SRE function lead. Inputs: ritual attendance log, ritual artefact log, any ritual that failed the default-off rule (< 50 % attendance or missing artefacts). Exit: an explicit add / kill / change decision for each ritual in the matrix; every change is written in the program ledger.
- **Career & certification review** — 30 minutes per person per quarter. Chair is the manager. Inputs: individual IDP, portfolio ledger (exercise 06), recent incidents the engineer led or participated in. Exit: an updated IDP with one named next milestone (project, certification, rotation, or promotion artefact) and its due date.

### 2.4 Program ledger — `reliability-program/ledgers/program-ledger.md`

The program ledger is the single append-only document that tracks whether the program *happened*, not what was decided inside each ritual. One row per ritual per occurrence:

```markdown
| Date | Ritual | Chair | Attendees (seats filled) | Artefact link | Notes |
|---|---|---|---|---|---|
| 2026-11-03 | SLO review | (SLO owner) | 5 / 5 | slo-ledger#2026-11-03 | freeze extended on `legal-help` |
| 2026-11-04 | Release & gate review | (release mgr) | 4 / 5 (eval seat out) | release-ledger#2026-11-04 | hold: chatcat-serving-g42a3 |
| 2026-11-05 | Reliability standup | on-call | 6 / 7 | standup-notes#2026-11-05 | — |
```

The retro reads this ledger to decide which rituals are drifting.

### 2.5 Error-budget policy — `reliability-program/policies/error-budget.md`

```markdown
# Error-budget policy (chatcat)

**Who owns the budget.** The SLO owner, per route class.

**What triggers a freeze.** A route class whose budget is below 0 at the end
of a weekly SLO review enters freeze. Freeze means: no non-reliability feature
work ships on that route class. Reliability work (burn-rate reduction, fallback
hardening, cost-correctness fixes) continues.

**What triggers an unfreeze.** Two consecutive weekly SLO reviews with positive
budget AND a written statement of what changed (which burn source was fixed).

**Who may appeal.** The product seat on the SLO review. Appeals are decided by
the SRE function lead, in writing, within three business days. During the
appeal, the freeze remains in force.

**What is not frozen.** Security and compliance fixes. Any fix that reduces
burn on the frozen route. Any fix that reduces risk on a related regulated
surface (per `mod-110`).

**How this policy connects to other rituals.**
- Release & gate review enforces the freeze: a frozen route's non-reliability
  releases are held.
- Incident learning review may convert an incident's action item into a freeze
  trigger if the pattern repeats.
- Reliability program retro reviews the number of freezes and their average
  duration; freezes that never unfreeze indicate a signal or threshold
  problem, not a product problem.
```

## 3. Validation steps

1. **Charter coverage check.** Every ritual in the matrix in §1.2 must have a corresponding file under `reliability-program/charters/`. A linter (`scripts/validate-program.py`) walks the matrix and fails if any charter is missing or if any charter lacks the required sections (Cadence / Owner / Required attendees / Inputs / Agenda / Decision authority / Exit criteria / Linked artefacts / Why this ritual exists).

2. **Calendar / charter consistency.** The calendar and the charters are generated from the same `owners.yaml` + `cadence.yaml` data file (optional, but the linter must confirm the cadence string in each charter matches the calendar entry).

3. **Program-ledger backfill test.** Run through the past four weeks and confirm there is a program-ledger row for every expected ritual occurrence. If a ritual has no row, the ritual did not happen; the retro is the place to decide what that means.

4. **Default-off rule rehearsal.** Simulate a ritual that missed artefacts for two cycles. Confirm the program retro agenda template surfaces it. Confirm the retro's written output either kills the ritual, reshapes it, or records a defender's written justification.

5. **Dry-run SLO review.** Walk through an SLO review using the `chatcat` dashboards. Confirm the meeting hits every exit criterion in ≤ 45 minutes. If any single agenda item runs over, the ritual is mis-sized; either the ritual's scope is too broad or the incoming signal is too chaotic and needs upstream cleanup.

6. **Cross-reference check.** Every charter's "Why this ritual exists" paragraph must cite either an upstream exercise (`mod-103`, `mod-106`, `mod-107`, `mod-108`, `mod-109`, `mod-110`, `mod-111`, `mod-112`, `mod-113`) or an official source (NIST AI RMF function, Google SRE workbook chapter). Rituals with no justification are candidates for the next retro.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 16 / 24 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Ritual matrix *(hard gate)* | < 6 rituals or missing cadence | 6+ rituals with cadence only | 6+ rituals, each with cadence, owner, inputs, outputs |
| 2 | Each ritual has an owner *(hard gate)* | Unnamed | Role named | Named person + rotation defined |
| 3 | Each ritual has exit criteria *(hard gate)* | Missing | Prose "we'll know when we see it" | Testable criteria a meeting can be gated on |
| 4 | Each ritual produces a written artefact | None | One ritual has an artefact | Every ritual has a named ledger / doc path |
| 5 | Error-budget policy wired to rituals *(hard gate)* | No policy | Policy exists but unlinked | Policy names which ritual freezes, unfreezes, appeals |
| 6 | Cross-functional seats named | "Whoever can make it" | Named by role | Named by person per seat, with backup |
| 7 | Governance ritual (retro) | Missing | Exists but no default-off rule | Retro + default-off rule + defender escape |
| 8 | Cadence matches signal frequency | All rituals weekly or all monthly | Mixed but unjustified | Cadence justified by input signal frequency |
| 9 | Freeze protocol | Missing | Mentioned | Written protocol + how release review enforces it |
| 10 | Linked to upstream module exercises | No links | 1–2 links | Each charter cites module or official source |
| 11 | Ritual artefacts are append-only ledgers | Ad-hoc docs | Shared doc without versioning | Append-only ledger with stable paths |
| 12 | Freeze windows named | Missing | Mentioned | Calendar + freeze charter |

## 5. Common mistakes

- **Standup as the SLO review.** A 15-minute standup cannot decide a freeze and should not try. Keep standup for coverage; put decisions in the ritual that owns them.
- **No written artefact.** A meeting that produces only slack messages did not happen. The ledger is the artefact.
- **Attendance by title, not name.** "Someone from eval" eventually means "nobody from eval". Name the seat.
- **Error-budget policy on paper only.** If no ritual *consumes* the budget, the policy is decoration. The SLO review must have the authority to freeze; the release review must have the authority to enforce.
- **Rituals multiplied without a retro.** New rituals accrete; old rituals never die. The quarterly retro with a default-off rule is the only thing that bounds program size.
- **Blame in the incident learning review.** The incident learning review exists to produce action items, not to decide who was at fault. A learning review that drifts into blame stops being useful within two cycles.
- **Peer contracts "owned by everyone".** A quarterly peer-contract review without a single chair (SRE function lead) never happens. One owner, one chair.
- **Career review folded into 1:1.** A 1:1 is weekly and tactical; the career review is quarterly and portfolio-driven. Folding them together means the career review never happens.
- **Skipping the retro the quarter you need it most.** When the quarter was hard, the retro is the thing that defends the program against knee-jerk reorg. Do not skip.

## 6. References

- Google SRE workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — the SLO-review cadence and error-budget policy framing this program adopts.
- Google SRE book, *Postmortem Culture* — [sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/) — the blameless-review discipline the incident learning review requires.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `GOVERN`, `MAP`, `MEASURE`, and `MANAGE` functions. The ritual matrix maps each ritual to the function(s) it discharges; in particular `MANAGE-2.3` (ongoing monitoring), `MEASURE-2.1` (performance against metrics), and `GOVERN-1.3` (organisational accountability).
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — motivates the vendor & provider risk review (ML06 ML Supply Chain) and the security seat on the incident learning review.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-ML threat catalogue that justifies a standing security seat on the incident learning review.
