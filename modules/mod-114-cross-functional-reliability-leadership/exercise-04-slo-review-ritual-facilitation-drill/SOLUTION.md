# Exercise 04 — SLO Review Ritual Facilitation Drill — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-04-slo-review-ritual-facilitation-drill.md`.

> **Scope note.** The paired exercise is a facilitation drill: the learner runs a mock SLO review meeting against a provided pre-read packet and produces (a) a run-of-show the chair follows, (b) the written SLO-ledger entry the meeting must leave behind, and (c) a rubric against which a facilitator can be scored. The reference solution uses the illustrative `chatcat` product and the SLO review charter defined in exercise 01.

## 1. Solution overview

The SLO review is the single ritual with the authority to **freeze** and **unfreeze** non-reliability work. Its value collapses when the chair does not drive it to decisions. Three failure modes dominate:

1. **Status-update drift.** The chair reads the burn-rate dashboard aloud, the room nods, and no decision is made. The budget is still blown next week.
2. **Thread-trap.** One route's burn consumes 40 minutes of a 45-minute meeting while three other routes are ignored. The meeting runs long, nobody wants to come back, the ritual dies.
3. **No written artefact.** Decisions are made in the room, nothing lands in the SLO ledger. Next week there is no record and the freeze is re-litigated.

The reference solution is a **facilitator's run-of-show**: a minute-by-minute script the chair follows, a pre-flight checklist that gates the meeting from starting, and a ledger-entry template the meeting cannot adjourn without filling in.

### 1.1 Design principles

- **Pre-flight gates the meeting.** The chair cancels the meeting if the pre-read is missing. "We'll figure it out live" is how status drift wins.
- **Fixed time boxes per agenda item.** The run-of-show allocates minutes; the chair enforces the time box. Over-time on one route is a signal that the route needs its own dedicated meeting, not that the SLO review is too short.
- **Every item leaves with an owner and a due date.** No item exits in "we'll talk next week" status. The ledger entry's structure makes this mechanically obvious.
- **The chair decides, writes, and announces.** A decision not written in the ledger within 24 hours did not happen. Chairs who delegate the write-up tend to lose decisions in transit.
- **The chair is explicit about veto use.** Freeze and unfreeze decisions are read back to the room and logged with the chair's name. "The group decided" obscures accountability.

## 2. Implementation — worked answer

### 2.1 Pre-flight checklist — `rituals/slo-review/preflight.md`

The chair runs this 15 minutes before the meeting starts. Any **NO** cancels or defers the meeting:

```markdown
# SLO review — pre-flight (chair only)

[ ] 1. SLO dashboard snapshot for 7 d and 28 d is in the pre-read doc
[ ] 2. Error-budget ledger entry for the current quarter is linked
[ ] 3. Every burn-rate alert since last review has a linked incident
[ ] 4. Any proposed SLO change has a written proposal, not just a Slack thread
[ ] 5. Required seats are confirmed present or have a named stand-in
[ ] 6. Last week's action items have status (done / open / blocked)
[ ] 7. The chair has read every incident post-mortem referenced in the pre-read

If any box is NO 15 minutes before start, send:
"SLO review deferred; <missing item> must be posted by <time>. Rescheduled to <slot>."
Cancellation is logged in the program ledger (exercise 01).
```

### 2.2 Run-of-show — `rituals/slo-review/run-of-show.md`

```markdown
# SLO review — run-of-show (45 minutes, chair script)

## 0–2 min — Open

Chair reads aloud:
- "This is the SLO review for week of <date>."
- "Our exit criteria today are: every route > 25 % burn has an owner and
   due date; every proposed SLO change is decided; action items written in
   the ledger within 24 hours."
- "Seats present: <enumerate>. Standing-in: <enumerate>. Any absent seat
   two weeks in a row? <call out if yes; program-retro note.>"

## 2–7 min — Burn by route class (5 min, strict)

Chair shares screen on the dashboard:
- Walk each route class exactly once. Each route gets ~30 seconds.
- Flag any route with > 25 % weekly burn for the deep-dive segment.
- Do NOT debate here; park every discussion for later segments.

## 7–22 min — Deep dives on burning routes (15 min, time-boxed per route)

For each flagged route (hard cap 3 minutes each; if more time is needed,
spawn a side-meeting after this one):

1. "What is the burn rate and the window?" — numeric answer only.
2. "What is the cause class — SLO-only, eval, capacity, provider?" —
   one word answer; if unknown, that itself is the action item.
3. "Containment status — who owns it, when do they expect to know more?" —
   named person, date.
4. "Does this change freeze status?" — chair-level decision; read aloud.

If more than 3 routes are burning, chair truncates per-route time so each
gets ≤ 4 minutes. If the room cannot cover all burning routes in 15
minutes, that is itself the finding; call a specialised meeting.

## 22–32 min — Error-budget policy decisions (10 min)

Chair walks the error-budget ledger:
- Open freezes: any candidate to unfreeze? Chair reads criteria aloud.
- Candidates to freeze: any route burning 2 consecutive weeks?
- Appeals in flight: status.
- Non-reliability work currently blocked: list, with linked PR / ticket.

Every freeze or unfreeze decision is a *chair* decision; read aloud, and
state "I am freezing/unfreezing <route>; reason: <one sentence>; this is
being logged." Resist "let's decide next week"; the budget does not wait.

## 32–40 min — New or changed SLOs (8 min)

Each proposed SLO change is read aloud by its proposer in ≤ 90 seconds:
- Target before/after.
- Why (incident, product change, regulatory bound).
- Impact on error budget.

Chair decides: accept / reject / defer. Decision is read back; if defer,
chair names what information will unblock the decision and by when.

## 40–44 min — Action items (4 min)

Chair reads back every action item from the last 40 minutes:
- Route X: <action> — <owner> — by <date>.
- SLO Y: <proposal> — accepted / rejected / deferred.
- Freezes: list changes.

Any ambiguity is an incomplete action item; chair names it as such.

## 44–45 min — Close

Chair: "Next meeting <date>. Pre-read due by <date>. I will post the
ledger entry within 24 hours. Thank you."
```

### 2.3 Chair's phrasebook — `rituals/slo-review/phrasebook.md`

Facilitation phrases that keep the meeting on time and that make decisions visible. The learner adapts these, but the drill is graded in part on how closely the chair uses explicit verbs.

```markdown
# SLO review — chair phrasebook

## Opening a decision
- "I am going to make a call here. If anyone has new information in the
  next 30 seconds, say now."

## Closing a thread
- "We are at time on this route. The question goes to <owner> to resolve
  offline and bring back next week."

## Freezing
- "I am freezing non-reliability work on <route>. Reason: <one sentence>.
  This takes effect immediately; it will be logged in the ledger. The
  release-review chair will enforce it."

## Unfreezing
- "I am unfreezing <route>. Criteria met: two consecutive weeks positive
  budget; written statement from <owner> of what changed. Logged."

## Appeals
- "<Owner>, if you want to appeal this, the appeal goes to the SRE
  function lead in writing. The freeze remains in force until the appeal
  is decided."

## Deferring an SLO proposal
- "I am deferring this proposal. To decide next time I need <specific
  information>. The owner is <name>; due by <date>."

## Handling disagreement
- "I hear the disagreement. My call stands; the appeal path is written.
  Let's move on so the budget decisions get air."

## Handling a thread-trap
- "We have spent <X> of the <Y> minutes allocated for <item>. I am
  calling the question or parking it. Which?"

## Closing the meeting
- "I will post the ledger entry within 24 hours. The ledger is
  authoritative; please correct anything in that PR, not here."
```

### 2.4 Ledger-entry template — `rituals/slo-review/ledger-template.md`

The meeting is not done until this is filled in and posted. The chair owns the post within 24 hours.

```markdown
# SLO ledger — <YYYY-MM-DD>

**Chair.** <name>
**Attendees.** <seats filled / 5>  — <names>
**Pre-read ref.** <link>

## Burn summary (7 d)

| Route | Burn | Trend vs last week | Cause class |
|---|---|---|---|
| faq | X % | ↓ | — |
| legal-help | X % | ↑↑ | eval |
| tool-call | X % | → | slo |

## Deep-dive outcomes

- **legal-help** — burn ↑↑ on eval signal. Cause: baseline drift after
  prompt-store update. Owner: eval lead. Due: YYYY-MM-DD. Action:
  recompute baseline on sampled set; report to next SLO review.
- ...

## Error-budget decisions

- **Freeze** legal-help: already frozen last week; staying frozen; week 2.
- **Unfreeze** memory-upsert: criteria met (2 weeks positive, written
  statement from capacity lead). Unfrozen effective now.
- **Appeals.** None open; none opened today.

## SLO changes

- **Accepted.** tool-call TTFT p95 tightened from 2.0 s to 1.5 s.
  Reason: new product tier promises < 2 s end-to-end. Owner: SLO owner.
- **Rejected.** faq refusal-rate SLO loosened from 5 % to 10 %. Reason:
  no documented product change; appears to be a response to a single
  bad week. Re-submit with 4 weeks of data.
- **Deferred.** memory-upsert error-rate SLO: deferred until the
  procurement decision from the capacity review (date YYYY-MM-DD).

## Action items

| # | Action | Owner | Due | Linked |
|---|---|---|---|---|
| 1 | Recompute groundedness baseline | eval lead | YYYY-MM-DD | eval-pipeline#... |
| 2 | Prepare appeal write-up for ... | product seat | YYYY-MM-DD | — |

## Linked artefacts

- Error-budget ledger: <link>
- Release ledger entries this week: <link>
- Incident post-mortems referenced: <links>
```

### 2.5 Facilitation drill — the scenario

The drill gives the learner a pre-read packet (dashboard screenshots, an error-budget ledger excerpt, two proposed SLO changes, three post-mortems) and asks them to run the meeting. The reference run-of-show above should be followable minute-by-minute; the ledger entry above is the submission artefact.

Worked outcome for the standard `chatcat` pre-read packet:

- **legal-help** burn is 42 % of weekly budget; cause eval; week-2 of freeze; action to recompute baseline.
- **tool-call** burn is 11 % of weekly budget; cause inference-performance regression on the new runtime; already contained; no freeze.
- **memory-upsert** has been positive for two consecutive weeks; the chair unfreezes it in the meeting.
- **tool-call TTFT** target is tightened from 2.0 s → 1.5 s with documented product justification.
- **faq refusal** loosening is rejected with written reason.
- One deferred proposal with named unblocking information.

## 3. Validation steps

1. **Pre-flight gate.** Run through the pre-read packet with and without one of the required items missing. Confirm the chair cancels the meeting in the missing-item case and logs the cancellation in the program ledger.
2. **Time-box compliance.** Dry-run the meeting against the standard pre-read packet. Confirm every segment finishes within its allocated minutes. Over-run on the deep-dive segment indicates too many burning routes; the chair should spawn a side-meeting, not extend the review.
3. **Decision verb coverage.** Grade the chair's transcript against the phrasebook — specifically, how many of `freeze`, `unfreeze`, `accept`, `reject`, `defer`, `appeal` are used in the explicit opening-closing form. Chairs who use only "let's..." phrasing tend to not produce decisions.
4. **Ledger post within 24 h.** Confirm the ledger entry is posted within 24 hours. Posts later than 24 hours erode the authority of the ritual within four cycles.
5. **Action-item follow-through.** Two weeks later, check the previous ledger's action items. Items with no status update are themselves the finding; raise them at the retro (exercise 01).
6. **Cross-link check.** Confirm every incident referenced is linked, every release-ledger entry is linked, and every freeze is enforced by the release-review chair (exercise 01 release-review charter).
7. **Red-team drill.** Have a reviewer throw a thread-trap mid-meeting (a lengthy debate on a non-burning route). Confirm the chair parks it using the phrasebook within 90 seconds.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 18 / 24 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Pre-flight gate used *(hard gate)* | Not run | Run but meeting proceeded on NO | Run; meeting deferred on any NO; cancellation logged |
| 2 | Agenda followed to time *(hard gate)* | Over-ran by > 25 % | Mostly on time | Every segment within time box |
| 3 | Every burning route has an owner + due date *(hard gate)* | Missing | Some have owners | All have owners + due dates |
| 4 | Freeze / unfreeze decisions read aloud | Not read | Named but not recorded as a chair decision | Named, attributed to chair, logged |
| 5 | Each SLO proposal decided | Deferred by default | Mixed decisions | Every proposal has an explicit accept / reject / defer with reason |
| 6 | Phrasebook verbs used | Hedging | Some explicit verbs | Explicit verbs for every decision class |
| 7 | Ledger entry posted ≤ 24 h | Not posted | Posted late | Posted on time, complete, linked |
| 8 | Thread-trap handled | Not handled | Handled by room pressure | Chair parked within 90 s using the phrasebook |
| 9 | Appeals named (even if none in flight) | Never mentioned | Mentioned | Named + path stated for each contested decision |
| 10 | Action items re-checked next meeting | Not re-checked | Re-checked informally | Re-checked by ledger walk; stale items escalated |
| 11 | Seat attendance called out | Not called | Called in prose | Called at open; standing-in named; retro note if two weeks absent |
| 12 | Chair wrote the ledger entry | Delegated | Co-written | Chair-authored |

## 5. Common mistakes

- **No pre-flight.** Meetings that start without the pre-read drift to status update by minute 10.
- **Deep-dives first.** Starting with a 20-minute deep-dive on one route eats the time for every other decision in the room.
- **Hedge verbs.** "I think we should maybe freeze" does not freeze. "I am freezing" does.
- **Ledger written by someone else.** The chair's name is on the ledger; delegating the write-up transfers the authority of the decision with it.
- **"Let's talk about this next week."** If the budget is burning now, next week is one more week of burn. Decide in the room or name the specific information needed to decide.
- **Thread-trap tolerated.** One route that eats 40 minutes is a sign the route needs its own meeting, not that the SLO review is broken. Park it.
- **Appeals mentioned only when invoked.** If the room never hears the appeal path, it reads chair decisions as unilateral. Name the appeal path at every contested decision; the appeal stays theoretical.
- **No cross-link to the release review.** A freeze that the release review does not enforce is a wish. Make the enforcement link explicit (the release-review chair reads this ledger before deciding the week's rollouts).
- **Attendance drift unnoticed.** A seat that is absent two weeks in a row is a program-retro signal; the SLO review itself cannot fix it, but it must surface it.

## 6. References

- Google SRE workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — the error-budget policy and burn-rate review cadence the chair discharges in the deep-dive and budget-decision segments.
- Google SRE book, *Error Budget Policy* — [sre.google/sre-book/embracing-risk/](https://sre.google/sre-book/embracing-risk/) — the freeze / unfreeze authority the chair exercises.
- Google SRE book, *Postmortem Culture* — [sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/) — the no-blame framing the chair enforces when a route's burn is cause-class eval or capacity.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `MANAGE-2.3` (ongoing monitoring) and `MEASURE-2.1` (performance against metrics) are discharged by this ritual; the chair cites them in the opening if the meeting is being audited.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — motivates the eval-cause and safety-cause segments of the deep-dive.
