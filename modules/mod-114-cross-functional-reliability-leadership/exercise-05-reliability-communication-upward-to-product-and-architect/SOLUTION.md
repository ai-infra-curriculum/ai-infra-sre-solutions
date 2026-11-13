# Exercise 05 — Reliability Communication Upward (to Product and Architect) — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-05-reliability-communication-upward-to-product-and-architect.md`.

> **Scope note.** The paired exercise asks the learner to write the *upward* reliability artefacts an AI-SRE lead produces for their Product peer (PM / product director) and their Architect peer (principal engineer / tech lead). The deliverables are three written artefacts covering three archetypal situations (a healthy week, an incident week, a trade-off decision), a rubric for grading them, and a short style guide for upward reliability communication. The reference solution uses the illustrative `chatcat` product and the SLOs, rituals, and contracts defined in the earlier exercises of this module.

## 1. Solution overview

Upward communication is where SRE leadership often loses. Common failure modes:

1. **Dashboard dump.** A weekly note that is a screenshot of the burn-rate panel plus a sentence. Product and Architect skim it, nothing decides.
2. **Buried headline.** The reader has to read to paragraph four before understanding whether this week is good, bad, or on-fire. By the time they get there, they have already decided from tone.
3. **No ask.** An incident note that describes what happened but does not name what SRE now needs from Product (prioritisation of a fix) or from Architect (architectural change). Readers assume no ask means no action.
4. **Tech-stack prose for non-SRE readers.** "p95 TTFT regressed 12 % on vLLM continuous batching after the kv-cache resize." Both readers nod and move on. Nothing changes.
5. **Over-promising certainty.** An incident recap that implies the fix is permanent. Two weeks later the same incident happens; SRE's credibility is now the subject of the meeting.

The reference solution is a **three-artefact set**, each with a fixed shape:

- **Weekly reliability note** — one page; TL;DR + burn + what changed + what's next + ask.
- **Incident upward brief** — one page; what happened + user impact + contained-by + why-we-think-it-was-us + ask.
- **Trade-off memo** — two pages; decision needed + options + reliability stance + reversibility + recommendation.

### 1.1 Style principles

- **Headline first, evidence second.** The first sentence of every artefact says whether the week is green / yellow / red, or whether an incident is contained / ongoing, or whether a trade-off has a recommended direction. Everything after is justification.
- **Numbers, with units and windows.** Not "latency up a bit"; `p95 TTFT 1.8 s → 2.1 s over 7 d`.
- **One ask per artefact.** If you need both Product and Architect to do something, you still have one ask per artefact; split into two notes if needed.
- **Translate the stack once, then stop.** Writing for a PM means replacing `kv-cache eviction` with "the request queue overflowed under load". Doing it once per note is service; doing it every paragraph is condescension.
- **Name the uncertainty.** "We think the cause was the retrieval index rebuild" is more honest than "The cause was the retrieval index rebuild" when the eval signal is still noisy.
- **End with what SRE will do next, with dates.** Readers scan the last paragraph.
- **Attach the evidence, do not paraphrase it.** Link the SLO ledger entry, the incident ledger, the release ledger — do not transcribe them.
- **Two readerships, two openings.** The PM reads the TL;DR + the ask. The Architect reads the TL;DR + the architectural implication. Structure accordingly.

## 2. Implementation — worked answers

Each of the three worked artefacts is a complete deliverable a learner may submit. Realistic numbers; references to the other module exercises.

### 2.1 Worked artefact A — weekly reliability note (healthy week)

Posted to the `#reliability-chatcat` channel and archived under `comms/weekly/<yyyy-mm-dd>.md`. Target length: half a page. Readership: Product director + Architect + their direct reports.

```markdown
# Weekly reliability note — chatcat — 2026-11-09 to 2026-11-15

**Status: GREEN.** Error budget positive on every route class; one small
regression contained; no asks this week.

## Burn by route class (7-day window)

| Route class | Weekly burn | Trend | Status |
|---|---|---|---|
| default (faq, support) | 12 % | ↓ from 18 % | green |
| legal-adj (legal-help) | 24 % | ↓ from 61 % | green, exiting last week's freeze |
| api-call (tool-call, memory-upsert) | 8 % | → | green |

## What changed this week

- The eval-side regression on `legal-help` from two weeks ago is contained;
  the recomputed groundedness baseline landed on YYYY-MM-DD; the route is
  unfrozen as of today.
- One canary rollout on `tool-call` was held by the release gate because
  `structured_exact_match_rate` landed at 0.994 against a 0.995 bound;
  the build was reshaped (prompt-template fix) and shipped the next day.
- No incidents at paging severity this week.

## What's next (SRE side)

- Running the quarterly peer-contract review (Mon); expected signal-schema
  changes in Inference-Performance's TPOT metric. Will post a diff after.
- Capacity review Tue; no procurement ask expected; headroom still above
  the 30 % floor through end of quarter.

## Asks

None this week. Reply in thread if you want an ad-hoc deep-dive on any row
above.

## Linked artefacts

- SLO ledger — <link>
- Release ledger (this week) — <link>
- Error-budget policy — <link>
```

### 2.2 Worked artefact B — incident upward brief

Posted during and immediately after an incident. Target length: half a page. Posted to the same channel as the weekly note, with a header that marks it as an incident update. Readership: Product director + Architect (both), the engineering director (CC).

```markdown
# Incident upward brief — chatcat — 2026-11-11 15:40 PT

**Status: CONTAINED, in recovery.** User-visible impact for ~42 minutes on
the `faq` and `support` routes. No PII exposed. Full post-mortem in 3
business days.

## What users saw

- `faq`: p95 TTFT rose from ~1.1 s to ~5.8 s between 14:58 and 15:40 PT.
  Users would have seen slow responses; no error page; ~22 % of requests
  took > 4 s.
- `support`: same pattern, slightly smaller (p95 rose to ~4.4 s).
- `legal-help`, `tool-call`: unaffected.
- No request was dropped; no response was wrong; no data was exposed.

## Timeline (compressed)

- 14:58 — burn-rate alert fires on `faq` TTFT.
- 15:02 — on-call confirms via dashboards; opens incident channel.
- 15:12 — identified: an index rebuild that started at 14:50 is competing
  with serving traffic on the retrieval cluster.
- 15:28 — index rebuild paused by MLOps on-call.
- 15:40 — p95 TTFT back under SLO.
- 15:55 — error budget on `faq` burned ~18 % of the weekly budget.

## Our current working theory

The index rebuild schedule was moved earlier this week as part of a cost
optimisation; the pre-move schedule did not overlap peak hours, the new
schedule does. We think this is the cause; we will confirm against two
more rebuild cycles before closing the post-mortem.

## What we're doing now (SRE)

- The rebuild has been rescheduled to the previous window.
- The retrieval cluster capacity is being scoped for a formal change
  (`mod-106` capacity review next Tue).
- The rebuild job owner (MLOps) is adding a guard that prevents overlap
  with peak hours.
- Incident learning review next Thursday; full post-mortem link will be
  in the next weekly note.

## Asks

1. **Product:** no change expected to any customer commitments. Please
   hold on announcing the new tool-call tier until after Thursday's
   learning review so the capacity delta is in scope.
2. **Architect:** we would like a 20-minute conversation this week on
   whether the retrieval cluster should be *isolated* from the index
   rebuild (separate control plane). Reply with slot.

## Linked artefacts

- Incident channel — <link>
- SLO ledger entry for this burn — <link>
- Release ledger context — <link>
```

### 2.3 Worked artefact C — trade-off memo

Pre-read for a decision meeting. Target length: two pages. Readership: Product director + Architect + engineering director (decider).

```markdown
# Trade-off memo — chatcat — tighten `legal-help` TTFT SLO vs ship legal-help autocomplete

## Decision needed

Should we (a) tighten the `legal-help` TTFT p95 SLO from 2.5 s to 2.0 s to
match the planned autocomplete feature's latency budget, or (b) ship the
autocomplete feature with the existing 2.5 s bound and accept that the
feature's p95 will frequently land at 2.3 s?

Decision owner: engineering director. Decision date: this Friday. SRE
recommendation: option (a), with conditions.

## Why this is live now

Autocomplete lands in the next release train. Product has promised a "sub-
2 s" experience in a marketing slot. The SLO today permits up to 2.5 s on
this route; roll-out in current state means some weeks the marketing
claim is quietly false.

## Options

### (a) Tighten SLO to 2.0 s now; ship autocomplete after capacity add

- **Reliability stance.** Positive. The SLO matches what we intend to
  promise. We need to add retrieval-cluster capacity (`mod-106`) to hold
  the tighter bound at peak — scoped at $18k / mo for the next two
  quarters, re-evaluated at the capacity review.
- **Speed cost.** The autocomplete release slips by ~2 weeks while
  capacity is reserved and the release gate re-tunes.
- **Reversibility.** Easy: loosen the SLO back if autocomplete is
  dropped. Capacity reservation is monthly; one month of lead time to
  release.

### (b) Ship now with 2.5 s SLO; publicly claim sub-2 s

- **Reliability stance.** Negative. The marketing claim is a de-facto
  SLO we would be hitting ~70 % of the time at peak hours, based on the
  last four weeks. The first bad week produces a customer-facing credibility
  hit that is harder to unwind than a 2-week slip.
- **Speed cost.** None in the short term.
- **Reversibility.** Hard: a marketing claim, once made, becomes a
  reputational anchor.

### (c) Ship with 2.5 s SLO and no sub-2 s marketing claim

- **Reliability stance.** Neutral. Keeps the existing SLO; keeps the
  feature on schedule; marketing team needs to adjust messaging.
- **Speed cost.** None, but the feature loses its positioning.
- **Reversibility.** Easy.

## SRE recommendation

**Option (a), with the following conditions:**

1. Capacity add is approved at the next monthly review ($18k / mo).
2. The 2.0 s SLO tightening is logged in the SLO ledger and the release
   gate schema is updated in the same PR.
3. If the capacity add is denied, we fall back to option (c), not (b).

## Risks we are flagging

- The 2.0 s bound depends on upstream provider p95 staying at current
  levels; if the provider's p95 degrades, we would need either a
  retrieval-side cache (Architect question) or a provider fallback
  (`mod-107`).
- The capacity add is a monthly reservation; we cannot rapidly scale
  down if autocomplete under-performs on adoption.

## Questions for the room

- **Product:** can marketing hold the sub-2 s claim until the capacity
  add lands?
- **Architect:** is a retrieval-side cache a candidate before next
  quarter? (If yes, option (a)'s conditions change.)

## Linked artefacts

- SLO ledger (legal-help history) — <link>
- Capacity review pre-read — <link>
- Release-gate contract — <link>
- `mod-107` provider-risk register — <link>
```

### 2.4 Style guide — `comms/style-guide.md`

```markdown
# Upward reliability comms — style guide

1. **Headline first.** First sentence states green / yellow / red for a
   weekly note, contained / ongoing for an incident brief, recommended
   option for a trade-off memo.

2. **Numbers with unit and window.** `p95 TTFT 1.8 s → 2.1 s over 7 d`,
   not "latency is up".

3. **Two readerships.** Write the TL;DR for the PM; write the
   architectural implication in a clearly labelled section for the
   Architect. Do not blend them.

4. **One ask per artefact.** If you have two asks, write two notes.

5. **Translate the stack once.** Say "the request queue overflowed under
   load" once; after that use the SRE term and link to a definition.

6. **Name the uncertainty.** "We think" is better than "we know" when the
   signal is still noisy.

7. **Attach, do not paraphrase.** Link the SLO ledger, the release
   ledger, the incident channel.

8. **End with dates.** The last paragraph has what SRE will do next and
   when.

9. **Never hedge the ask.** "Could you consider maybe..." reads as "no
   ask". Write the ask in the imperative: "Hold the marketing claim
   until Thursday." The reader can still say no.

10. **Credibility compounds.** Every incident brief that says "we think"
    and is later confirmed earns trust. Every one that said "we know" and
    was wrong spends it.
```

## 3. Validation steps

1. **Headline test.** Ask a non-SRE reader to read only the first sentence of each artefact and answer: "Is the week good or bad? Is the incident contained? Does the trade-off have a recommendation?" If any answer is wrong or uncertain, the headline fails.
2. **Ask test.** Ask the same reader: "What does SRE want me to do?" They should answer in one sentence. If they can't find the ask, the ask is not written clearly or is missing.
3. **Numbers test.** Walk every number in every artefact and confirm it has a unit and a window. "Latency up 10 %" is a failure; "p95 TTFT up 10 % over 7 d" is a pass.
4. **Evidence link check.** Every claim that references the SLO, the release, or the incident has a working link. No paraphrased dashboard data.
5. **Uncertainty calibration.** For the incident brief, confirm the working-theory section uses "we think" or equivalent hedge, and that the post-mortem commitment has a date. For the trade-off memo, confirm each option has a reliability-stance line that does not oversell.
6. **Reversibility named.** For the trade-off memo, every option has a one-line reversibility note. Options without one are under-examined.
7. **Length audit.** Weekly note ≤ 1 page; incident brief ≤ 1 page; trade-off memo ≤ 2 pages. Longer artefacts are a sign the headline is buried.
8. **Two-reader structure.** Each artefact has a section a PM can read and act on, and a section an Architect can read and act on. A memo that is entirely one or the other is a failure for the other reader.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 16 / 22 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Headline first *(hard gate)* | Buried | Present but soft | First sentence; clear verdict |
| 2 | Numbers with unit + window *(hard gate)* | Missing | Partial | Every number has unit + window |
| 3 | Ask is explicit *(hard gate)* | No ask | Hedged | One imperative ask per artefact |
| 4 | Two readerships addressed | Only one | Both named | Each has a clearly labelled section |
| 5 | Uncertainty calibrated | Overclaims or under-claims | Mixed | Working theory flagged; post-mortem date named |
| 6 | Evidence linked, not paraphrased | Paraphrased only | Mixed | All claims linked to ledger |
| 7 | Reversibility (trade-off memo) | Missing | Mentioned | Per option, one-line |
| 8 | Length discipline | Over length | Mixed | Weekly ≤ 1pg, incident ≤ 1pg, memo ≤ 2pg |
| 9 | Freeze / release / capacity cross-link | Missing | Mentioned | Linked to the appropriate ritual / ledger |
| 10 | SRE recommendation defensible | Opinion | Reasoned | Reads off the SLO ledger / capacity / provider-risk evidence |
| 11 | Tone neither heroic nor deflecting | Heroic or deflecting | Neutral prose | Neutral prose + named uncertainty |

## 5. Common mistakes

- **Dashboard dump.** A screenshot is not a note; a note explains what changed and what SRE wants the reader to do.
- **Buried headline.** The verdict should be in the first sentence.
- **Over-translation / under-translation.** Replace jargon once, then use the SRE term and link. Repeating the translation every paragraph reads as condescension.
- **No ask.** A weekly with no ask is fine (say so). An incident brief or a trade-off memo without an ask is a problem; readers assume no ask means no action.
- **Hedging the ask.** "Could you consider..." is not an ask. Write it in the imperative; readers can still say no.
- **Overclaiming the cause.** "The cause was X" when the signal is still noisy is how credibility compounds in the wrong direction. "We think X" is defensible; it is also the honest state of the investigation at the time.
- **No reversibility line in the trade-off memo.** An option without reversibility hides the strategic weight of the decision; readers will assume "reversible" when it often isn't.
- **Tone drift during incidents.** Heroic prose ("the team battled through the night") and deflecting prose ("a transient provider issue") both erode trust. Neutral prose + named uncertainty is the discipline.
- **Mixing the PM and Architect sections.** A single section intended for both tends to serve neither. Label the sections.

## 6. References

- Google SRE workbook, *Communication and Collaboration in SRE* — [sre.google/workbook/communication-collaboration/](https://sre.google/workbook/communication-collaboration/) — framing for the "upward" communication category and for incident-communication norms.
- Google SRE book, *Postmortem Culture* — [sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/) — the uncertainty-naming discipline used in the incident-brief template.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `GOVERN-3.2` (transparency and accountability) and `MANAGE-4.3` (communication of AI risks to stakeholders) frame the upward comms as discharging specific governance controls, not as a nice-to-have.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — the incident-brief's PII / data-exposure sentences map to OWASP's data-privacy categories; the template forces the writer to say explicitly whether PII was involved.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — relevant when an incident brief covers an adversarial-ML cause class; the brief must name the ATLAS technique if applicable, and link it.
