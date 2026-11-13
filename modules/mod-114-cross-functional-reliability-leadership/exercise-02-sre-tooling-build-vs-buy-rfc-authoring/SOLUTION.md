# Exercise 02 — SRE Tooling Build vs Buy RFC Authoring — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-02-sre-tooling-build-vs-buy-rfc-authoring.md`.

> **Scope note.** The paired exercise asks the learner to author a *build-vs-buy* RFC for an AI-SRE tooling decision. The output is a written RFC with explicit decision criteria, costed options, a recommendation, and a rubric to score candidate RFCs. The reference solution uses a concrete decision as the worked example: whether to **build** a release-gate verdict service (as in `mod-111` exercise 05) or **buy** a commercial LLM-eval gating product. The shape of the RFC transfers to any AI-SRE tooling build-vs-buy decision.

## 1. Solution overview

Build-vs-buy decisions are where SRE leadership either earns or loses credibility with product, finance, and security peers. The common failure modes:

1. **Vibe decision.** "Buy is faster" or "build is cheaper" with no numbers. The decision survives one quarter and then re-opens at the first cost shock.
2. **Framework-shopping.** The author picks the framework (TCO, RICE, Wardley mapping) that produces the answer they already wanted. The RFC looks rigorous; the decision is not.
3. **Missing the "null option".** Neither build nor buy. Sometimes the correct answer is: don't do this yet. An RFC that cannot name the status-quo option as a candidate is not honest.
4. **No reversibility analysis.** A buy that embeds vendor identifiers deep in the stack and cannot be ripped out in a quarter is a *strategic* purchase, not a tooling purchase. Many RFCs miss this axis.
5. **Hidden ownership cost.** "We'll buy it" skips the on-call ownership of the integration. Buying the SaaS does not buy you out of operating it.

The reference RFC solves these by:

- Scoring every option against a **fixed six-axis criteria set** (fit, total cost over three years, time-to-decision-worth, reversibility, strategic lock-in, security / compliance posture), with no option exempt.
- Including an explicit **null option** ("defer 1 quarter"), scored on the same axes.
- Separating **sticker-TCO** (license + infra) from **ownership-TCO** (on-call, integration maintenance, vendor-management overhead).
- Naming a **kill criterion** up-front for the recommended option — a measurable condition under which the decision is revisited inside its first year.

### 1.1 Decision criteria

| Axis | What is measured | How it's scored |
|---|---|---|
| **Fit** | How well the option satisfies the stated functional and operational requirements, verbatim | 0 (misses hard requirement) / 1 (meets with workaround) / 2 (meets natively) per requirement; sum normalised |
| **Total cost (3 yr)** | Sticker + ownership, 3-year present value | In dollars; option with lowest cost = 2, within 25 % = 1, > 25 % higher = 0 |
| **Time-to-decision-worth** | Weeks from "yes" to the first production-blocking decision the tool makes | ≤ 8 wks = 2, ≤ 16 wks = 1, > 16 wks = 0 |
| **Reversibility** | Weeks to switch away from this option if it fails | ≤ 4 wks = 2, ≤ 12 wks = 1, > 12 wks = 0 |
| **Strategic lock-in** | Depth at which option embeds in the stack (runtime, data, auth, PII, SLO ledger) | None = 2, one surface = 1, three+ = 0 |
| **Security / compliance posture** | Regulatory artefacts, data boundary, incident-notification SLA | Meets all stated requirements = 2, meets with mitigation = 1, misses one hard requirement = 0 |

Hard requirements cannot be traded off. Any option with a 0 on fit or on security / compliance is disqualified before scoring is averaged.

## 2. Implementation — the worked RFC

The worked example that follows is a complete RFC for the `chatcat` release-gate decision. It is the artefact a learner submits for this exercise. The text is the deliverable; the scoring tables are the proof of reasoning.

---

### 2.1 Worked RFC — `rfcs/rfc-0042-release-gate-build-vs-buy.md`

```markdown
# RFC-0042 — Release-gate build vs buy

- **Author:** (AI-SRE function lead)
- **Status:** Proposed
- **Reviewers:** eval owner, inference-performance owner, security peer, finance peer
- **Decision date:** YYYY-MM-DD (two weeks from posting)
- **Decision authority:** AI-SRE function lead (recommender), engineering director (approver)

## Context

`chatcat` runs progressive-delivery rollouts (`mod-111` exercises 01, 03, 04).
Each rollout consults a release gate that reads the eval signals (owned by
the eval team) and the SLO signals (owned by SRE) and returns a single
`pass | fail | needs-review` verdict (see `mod-111` exercise 05). Today the
gate is a hand-run script invoked by on-call. We need a durable service.

## Problem statement

We need a release-gate verdict service that:

1. Reads a signed, versioned contract (routes → thresholds → owners).
2. Pulls fresh values from Prometheus (SLO) and the eval pipeline (eval).
3. Returns a verdict with full per-signal reasoning.
4. Records every verdict in an append-only ledger.
5. Is callable by Argo Rollouts, Flagger, and our shadow-promotion tool.
6. Meets freshness, auditability, and dispute-resolution requirements set
   by the paired contract.

### Hard requirements

- Verdict latency < 200 ms p95 under 50 rps (release automations are
  synchronous callers).
- Freshness check per signal (stale signal = fail).
- Full audit trail; every verdict reconstructible from the ledger.
- Runs inside our VPC; verdict data does not leave our boundary.
- No release-blocking signal depends on a third-party control plane.

## Options

### Option A — Build

A Python service matching `mod-111` exercise 05's reference. Deployed on our
existing Kubernetes platform, Prometheus data source, S3-backed ledger.

Engineering cost: 1 senior SRE × 4 weeks to v1; 0.25 FTE ongoing.
Infra cost: negligible (shared cluster).
Operational cost: on-call rotation already exists.

### Option B — Buy vendor-X "ReleaseSentinel"

Commercial SaaS. Ingests signals via webhook + our Prometheus remote-read
endpoint. Returns verdict via HTTP.

Engineering cost: 1 SRE × 2 weeks integration; 0.1 FTE ongoing for
vendor-management (contract, security review renewals, escalations).
License cost: $48 000 / yr list; expected $36 000 / yr after negotiation,
plus $12 000 / yr for the add-on required to pin thresholds per route.
Signal data leaves our VPC to vendor's eval cloud (vendor's documentation).

### Option C — Buy vendor-Y open-source distribution + support

An open-source release-gate tool with a paid support tier. Self-hosted in
our VPC.

Engineering cost: 1 SRE × 3 weeks integration + customisation for freshness
semantics; 0.25 FTE ongoing.
License cost: $18 000 / yr for support tier.
Infra cost: a standalone cluster for the tool's control plane.

### Option D — Null option: defer one quarter

Keep running the hand-run script with a documented manual-approval step.
Revisit this RFC after we have seen `mod-111` exercises 03, 04 in production
for a full quarter, so that we know which signals we actually gate on.

Engineering cost: 0.
Infra cost: 0.
Risk cost: unknown; release manager continues to be a bottleneck; two of
the last four rollouts were delayed by ≥ 6 hours because of this.

## Scoring (0 / 1 / 2 per axis)

| Axis | A (build) | B (buy SaaS) | C (buy OSS+support) | D (defer) |
|---|---|---|---|---|
| Fit | 2 — written against our contract | 1 — meets requirements with workaround on freshness semantics | 1 — meets with per-route-threshold plug-in work | 0 — misses hard requirement (manual) |
| Total cost (3 yr, present value) | **$180k** (labour, discounted) → 2 | **$144k** licence + $120k labour = $264k → 0 | **$54k** licence + $150k labour = $204k → 1 | $60k (release-manager time) → *not comparable* |
| Time-to-decision-worth | 4 wks → 2 | 2 wks → 2 | 3 wks → 2 | n/a |
| Reversibility | 2 wks (small codebase) → 2 | 12 wks (webhooks, data in vendor) → 1 | 6 wks (OSS; data local) → 2 |n/a |
| Strategic lock-in | None → 2 | Three surfaces (Prometheus, ledger, auth) → 0 | One surface (control-plane cluster) → 1 | n/a |
| Security / compliance | Meets → 2 | **Hard-requirement miss**: signal data leaves VPC → **disqualified** | Meets → 2 | Meets → 2 |
| **Score (excl. disqualified)** | **12 / 12** | **DQ** | **9 / 12** | **DQ** (fails fit hard requirement) |

## Recommendation

Adopt **Option A — Build.**

Option B is disqualified on a hard security requirement (signal data must not
leave the VPC). Option D is disqualified on a hard fit requirement (we cannot
keep a manual approval step in the loop for the automation rollouts in
`mod-111` exercises 01, 03, 04). Between Option A and Option C:

- A scores higher on every remaining axis.
- A is built directly against our contract shape; C requires a plug-in for
  per-route thresholds that we will own either way.
- A's ownership cost is the same as our existing SRE rotation; C adds a
  standalone control-plane cluster we have to patch.

## Kill criterion

If, 12 months after v1, any of the following is true, we revisit this RFC:

- Verdict-service feature backlog has grown to > 4 engineer-weeks per
  quarter of maintenance (we would then be *becoming* a vendor).
- A commercial option appears that permits in-VPC deployment and would
  score ≥ A's 12 / 12.
- The contract's shape has stabilised enough that the control-plane
  integration cost of a buy option drops below 4 weeks.

## Rollout plan

1. Week 0–1: service skeleton, contract parser, Prometheus reader.
2. Week 2: freshness enforcement, ledger writer, HTTP shape.
3. Week 3: integrate with Argo Rollouts (exercise 01). Dark-run against
   the past 2 weeks of release-ledger data.
4. Week 4: cut over the first canary; keep the hand-run script as a
   fallback for 2 weeks.
5. Week 6: retire the hand-run script.

## Reversibility plan

The service exposes the HTTP shape documented in `mod-111` exercise 05. If
we later replace it, the rollout tools call the same endpoint; the ledger
format is stable JSON. Replacement is ~ 2 weeks of integration.

## Open questions

1. Who owns the contract-change PR review when the SLO owner and the eval
   owner disagree? *(Resolved in exercise 03: cross-peer contract review.)*
2. What is the SLO on the verdict service itself? *(Proposed: 99.9 % over
   28 days, 200 ms p95; written into the SLO ledger on v1 go-live.)*

## Appendix A — Cost assumptions

- Senior SRE fully-loaded cost: $230 000 / yr (per finance's 2026 rate card).
- Discount rate: 7 % per `finance/rfc-rates-2026.md`.
- Vendor list prices from public pricing pages, dated YYYY-MM-DD; renewal
  risk assumed at 10 % / yr increase.

## Appendix B — Reference contracts

- `mod-111` exercise 05 — release-gate contract shape.
- `mod-110` exercise 04 — reliability-gate vs safety-gate reconciliation.
- NIST AI RMF `MANAGE-2.3` — the pre-release decision authority this
  service discharges.
```

---

### 2.2 How this RFC maps to the criteria

The scoring table makes two moves worth calling out explicitly:

1. **Hard-requirement disqualification is applied before the average.** Option B *looks* like a legitimate option until you check the security axis. Allowing the average to "wash out" a hard-requirement miss is how RFCs produce wrong answers; the rubric in §4 therefore treats disqualification as a hard gate.
2. **The null option is scored, not implied.** Making the defer option an explicit candidate forces the author to argue that doing nothing is worse than either build or buy. In the chatcat case, the defer option fails a hard fit requirement; in many real cases, defer wins, and the RFC is a defensible "not yet".

### 2.3 Reusable RFC template — `rfcs/_template.md`

```markdown
# RFC-NNNN — <short title>

- **Author:**
- **Status:** Proposed | Accepted | Rejected | Superseded
- **Reviewers:** <named roles / people>
- **Decision date:** <YYYY-MM-DD>
- **Decision authority:** <recommender> / <approver>

## Context

## Problem statement
### Hard requirements

## Options
### Option A — Build
### Option B — Buy (<vendor>)
### Option C — ... (as applicable)
### Option D — Null option: defer N quarters / do nothing

## Scoring (0 / 1 / 2 per axis)

| Axis | A | B | ... |
|---|---|---|---|
| Fit |  |  |  |
| Total cost (3 yr, present value) |  |  |  |
| Time-to-decision-worth |  |  |  |
| Reversibility |  |  |  |
| Strategic lock-in |  |  |  |
| Security / compliance |  |  |  |
| **Score (excl. disqualified)** |  |  |  |

## Recommendation
## Kill criterion
## Rollout plan
## Reversibility plan
## Open questions
## Appendix A — Cost assumptions
## Appendix B — Reference contracts
```

## 3. Validation steps

1. **Hard-requirement check.** Confirm every option is scored against each hard requirement individually and that any miss produces a `DQ`. An RFC that rolls hard-requirement misses into the fit average is failing the exercise.
2. **Null-option presence.** Confirm an explicit defer / do-nothing option is scored on the same axes as the build and buy options. "N/A" is acceptable only when the exercise's problem statement makes defer structurally impossible (e.g. a compliance deadline that forces action by a named date).
3. **Cost separation.** Confirm the cost cell separates sticker-TCO from ownership-TCO and uses a disclosed discount rate. An RFC that reports a single lump sum without a basis is unauditable.
4. **Reversibility weeks are supported.** The reversibility score cites a *mechanism* (how we would switch away), not a wish ("probably a few weeks"). A build option whose reversibility is "impossible to replace" is a strategic decision, not a tooling decision.
5. **Strategic lock-in surfaces counted.** The author lists each surface the option would embed in (Prometheus, auth, ledger, PII, SLO stack). An option whose only lock-in claim is "none" without an enumeration is under-examined.
6. **Kill criterion is measurable.** The kill criterion names at least one measurable condition (cost, backlog size, reversibility change). "If we don't like it we'll revisit" is not a kill criterion.
7. **Decision authority is named by role AND person.** An RFC without a named approver drifts.
8. **Peer review exists.** Finance peer signed off on the cost assumptions. Security peer signed off on the data-boundary and compliance posture. No RFC passes without both signatures (recorded on the RFC header or in the merge metadata).

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 16 / 22 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Hard requirements named *(hard gate)* | Not named | Named without disqualification rule | Named + explicit disqualification applied |
| 2 | Options enumerated *(hard gate)* | Only one | Build + buy only | Build + buy(s) + explicit null option |
| 3 | Each option costed | Lump sum | Sticker-TCO only | Sticker + ownership, discount rate disclosed |
| 4 | Reversibility measured | Missing | In prose | Weeks + mechanism |
| 5 | Strategic lock-in measured | Missing | One axis | Enumerated surfaces |
| 6 | Security / compliance axis *(hard gate)* | Missing | Mentioned | Scored per option with disqualification |
| 7 | Scoring table applied | Narrative only | Partial table | Full table + explanatory cells |
| 8 | Recommendation justified by scoring | Opinion | Narrative | Reads off the table + addresses close calls |
| 9 | Kill criterion | Missing | Prose "we'll check" | Measurable triggers with numbers |
| 10 | Peer sign-offs | None | One peer | Finance peer + security peer named |
| 11 | Decision authority named | No | Role only | Role + named approver + date |

## 5. Common mistakes

- **Framework-shopping.** Picking TCO because it makes build cheaper, or RICE because it makes buy win on reach. The scoring table forces every option through the same axes; the recommendation reads off the table.
- **"Vendor is cheaper because we save engineer time."** True only if those engineers would otherwise have built *this*. If they are already saturated on incident response, the "savings" are imaginary. Ownership-TCO captures the vendor-management labour that buys do not save.
- **No disqualification rule.** An option that fails a hard security requirement cannot be rescued by scoring well on cost. The rubric's hard-requirement gate exists because this is the single most common failure mode.
- **No null option.** Many RFCs assume we must do something; sometimes "defer one quarter" is the right answer, especially when the problem shape is still moving.
- **Missing reversibility.** A buy decision with 12-month switching cost is strategic, not tactical; the RFC must say so explicitly rather than smuggling the lock-in past review.
- **Kill criterion is a mood.** "If it doesn't work we'll switch" is not a kill criterion. A measurable trigger (backlog > X eng-wks / qtr, cost > Y, a specific vendor feature shipping) is.
- **Finance / security sign-off deferred to "after approval".** Peer sign-off is part of the RFC, not a follow-up. An RFC that lands without both is routinely re-opened the next quarter.
- **Recommendation written before the scoring.** You can see this when the scoring table's cells are all 2s for the author's preferred option and all 0s for the alternative. Score first; then write the recommendation.

## 6. References

- Google SRE workbook, *Managing Operational Load* — [sre.google/workbook/operational-load/](https://sre.google/workbook/operational-load/) — the framing for ownership-TCO (SaaS does not buy you out of operating it).
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `GOVERN-1.7` (processes and procedures for AI-system decision-making) and `GOVERN-6.1` (policies for third-party AI components) motivate the hard-gate axes on security and compliance and the formalised buy evaluation.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML Supply Chain Attacks` is the reason every buy option is scored on the security / compliance axis with disqualification rather than averaging.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-ML threat catalogue; a vendor whose control plane sits in the signal path becomes part of our supply chain and inherits ATLAS surface area.
