# Exercise 02 — Error-Budget Policy Authoring for Non-Deterministic Systems

> Reference solution. The exercise asks the learner to author an **error-budget
> policy** (not just SLOs) for an LLM inference platform that the engineering
> org, product, and leadership will all agree to in advance. The non-trivial
> part is that failure modes include *quality regressions* and *provider
> drift*, not only classical error / latency breaches.

## 1. Solution overview

An error-budget policy is the *written contract* that converts SLO burn into
action. The Site Reliability Workbook, Ch. 4 defines a good policy as: "an
explicit, written agreement that defines, in advance, what will happen when
the error budget is exhausted." For a deterministic service that usually
reduces to "freeze feature launches until burn stops." For a non-deterministic
AI service, three structural problems have to be solved on top of that:

1. **The service can be *out of budget on quality* while every HTTP status is
   green.** Policy has to trigger on quality-burn too.
2. **The cause of a burn is often outside the team's control** (provider
   degradation, upstream weight change, data corpus shift). Policy has to name
   which actions are available in each case.
3. **The remediation set is richer than "stop launching."** Model rollback,
   prompt rollback, prompt-template pinning, downgrade to a smaller model,
   retrieval shutdown, and vendor failover are all first-class policy
   responses — but they all have their own risks (cost, behavior change,
   compliance implications) that the policy has to pre-authorize.

The worked answer below is a complete, drop-in error-budget policy document
written against a fictional but realistic LLM service, plus the reasoning
behind each clause. Numbers are placeholders — the structure is the
deliverable.

## 2. Implementation — the policy document

> This is written as the actual artifact the learner should produce — a signed
> policy. Treat anything inside the `policy` block as the deliverable.

---

### 📄 LLM Inference Platform — Error-Budget Policy (v1.0)

**Owners:** AI SRE (primary), Product Eng, Platform PM.
**Review cadence:** quarterly, or on any policy-triggered action.

#### A. Scope

This policy governs the production LLM inference platform fronting the
`/chat`, `/agent`, and `/embed` routes. It applies to:
- Latency SLOs (TTFT, ITL / TPOT) per workload class.
- Streaming availability SLO.
- Quality SLOs (judge-pass, grounding where applicable).
- Cost SLO ($/1M output tokens, rolling 7-day).

It does **not** govern internal evaluation jobs or offline backfills.

#### B. SLOs and error budgets

| # | SLI | SLO | Window | Monthly budget (approx.) |
|---|---|---|---|---|
| L1 | `/chat` TTFT ≤ 1.0 s | 99.0% | 28 d | 1.0% of chat requests |
| L2 | `/chat` streaming completes | 99.9% | 28 d | 0.1% of chat streams |
| L3 | `/agent` end-to-end wall-clock ≤ 15 s | 95.0% | 28 d | 5.0% of agent runs |
| Q1 | `/chat` judge-pass ≥ threshold | 97.0% | 7 d | 3.0% of sampled responses |
| Q2 | `/chat` grounding (RAG on) | 95.0% | 7 d | 5.0% of RAG responses |
| C1 | `/chat` $/1M output tokens ≤ budget | monthly | calendar month | $-denominated |

> Thresholds are placeholders. The point of this table is: **every budget
> that can trigger the policy is enumerated here, by number.** No budget may
> trigger an action that isn't named in Section D.

#### C. Burn-rate alerting (multi-window, multi-burn-rate)

Following *The Site Reliability Workbook*, Ch. 5, each SLO has:

- **Page alert**: 2% of budget consumed in 1 hour **AND** 5% in 6 hours.
  (Equivalent to short- and long-window conditions firing together.)
- **Ticket alert**: 10% of budget consumed in 3 days.

Rationale: the two-condition page alert catches real burn fast without
paging on noise; the ticket alert catches slow drip failures (prompt
regressions, provider latency drift).

#### D. Policy triggers — what happens when the budget is exhausted

Triggers below are **cumulative**: hitting a stronger trigger does not undo
a weaker one. All triggers require an SLO ID and a burn-rate reading.

**D.1 — Launch freeze on affected SLO (soft stop).**
- **Trigger:** budget for that SLO is down to ≤ 25% remaining in-window.
- **Action:** stop merging feature changes that touch the affected path
  (inference server config, prompt templates, model weights, retrieval
  index, tool plumbing). Fix-forward work and rollbacks are still allowed.
- **Who signs off to resume:** on-call SRE + owning engineering lead.

**D.2 — Model / prompt pin (hard freeze on contributing surfaces).**
- **Trigger:** budget is exhausted (≤ 0% remaining) OR 50%+ burned in the
  last 24 h from a single identifiable change class (new weights, new
  prompt, new retriever).
- **Action:** pin the last-known-good model version, prompt template, and
  retriever config. Automated promotion pipelines are halted. New candidate
  models continue to be evaluated offline but not promoted.
- **Who signs off to resume:** AI SRE lead + model-owning team lead + product.

**D.3 — Rollback (operational, pre-authorized).**
- **Trigger:** burn rate ≥ 10× the 1-hour window AND a specific change is
  the suspected root cause.
- **Action:** the on-call is pre-authorized to roll back the change without
  additional approval. This includes reverting model version, prompt
  template, retriever index, routing config, or tool-call config.
- **Rationale:** without pre-authorization, rollback drifts into committee,
  and burn continues.

**D.4 — Degradation modes (cost / saturation).**
- **Trigger:** cost SLO (C1) burning at projected month-end breach, OR
  latency SLO burning due to sustained GPU saturation.
- **Pre-authorized actions, in this order:**
  1. Reduce `max_tokens` cap on affected routes.
  2. Route tier-2 traffic to the smaller model variant.
  3. Admission-control: fail fast with 503 + explicit `Retry-After` when
     queue depth exceeds knee threshold (preserves latency SLO at the cost
     of availability SLO — explicitly a budget trade).
- **Reversal:** on-call returns to normal config when burn rate < 1× for 24 h.

**D.5 — Vendor failover (provider-caused burn).**
- **Trigger:** upstream model provider is the identified root cause (via
  provider status + divergence from internal eval), AND on-call expects the
  degradation to exceed 30 min.
- **Action:** switch traffic for affected route to the configured
  secondary provider (see runbook). Communicate externally if customers are
  affected.
- **Budget accounting:** burn attributable to the vendor is still counted
  against our SLO — our customers don't see the vendor. It *is*, however,
  flagged in the post-incident review for the vendor-risk register.

**D.6 — Quality burn (Q1 / Q2).**
- **Trigger:** Q1 or Q2 budget exhausted in 7-day window.
- **Action:** mandatory evaluation re-run on the current production model
  and prompt. If the regression is reproducible offline → rollback (D.3).
  If it is not reproducible offline, the on-call opens a quality incident,
  snapshots the retrieval index and recent traffic sample, and escalates to
  the modeling team. Launches of new prompts or models to that route are
  frozen until the incident is closed.
- **Rationale:** a quality burn that *doesn't* reproduce offline usually
  points to data drift, prompt drift at the edges of the distribution, or
  silent config change. Freezing promotion while it's uncharacterized is
  the safer default.

**D.7 — Executive escalation.**
- **Trigger:** two full-window budget exhaustions in a rolling 60 days on
  the same SLO, OR single catastrophic breach (>10% of users impacted >1 h).
- **Action:** written root-cause review to engineering leadership;
  investment decision (headcount, infra, vendor) within one review cycle.
- **Rationale:** this is the "engineering / product tension" valve from the
  SRE Book — repeated budget misses must become an *organizational*
  decision, not just an engineering backlog item.

#### E. Explicit out-of-scope items

- **The policy does not grant permission to lower an SLO to end a burn.**
  SLOs may be re-tuned only in a scheduled quarterly review with an
  explicit written rationale tied to user research or SLA changes.
- **The policy does not require a rollback decision to be unanimous.**
  Rollback authority sits with the on-call for D.3; D.2 and later require
  named sign-offs listed per clause.
- **The policy does not apply to safety or compliance incidents.** Those
  route through the safety-incident runbook, which can supersede this
  policy at any time (Section F).

#### F. Interaction with safety / compliance

A safety incident (e.g., model produces policy-violating output at a
measurable rate above threshold, or a data leak is observed) **supersedes**
this policy. The safety-incident runbook can require model shutoff,
response filtering, or external disclosure regardless of budget state. NIST
AI RMF *Govern / Manage* functions require that these pathways be
pre-authorized — see Section G.

#### G. Sign-offs

- AI SRE lead: __________________
- Platform engineering lead: __________________
- Product (owning PM): __________________
- Security / Safety representative: __________________
- Date / version: __________________

---

### 2.1 Decision rationale

- **Policy over discretion.** The Workbook's whole argument for an error-
  budget policy is that *in-the-moment* decisions during a burn are bad
  decisions. Everything in Section D is pre-authorized so the on-call can act.
- **Quality burn needs its own clause (D.6).** Latency burns rollback cleanly;
  quality burns often don't reproduce offline. If the policy only contains a
  "rollback the model" rule, it gives the wrong instruction when the cause is
  data drift or retriever drift. D.6 explicitly separates reproducible from
  non-reproducible quality burns.
- **Degradation is a budget trade (D.4).** Admission control that throws
  fast-503s *preserves latency SLO at the cost of availability SLO*. Writing
  this down in the policy makes it an explicit organizational choice rather
  than an on-call improvisation.
- **Vendor burn still counts against our SLO (D.5).** Customers don't know
  about our vendors. Counting vendor-caused burn against the budget is how
  vendor-risk gets prioritized; shielding it would hide the signal.
- **Safety overrides the budget (F).** NIST AI RMF (`GOVERN`, `MANAGE`) and
  OWASP ML Top 10 (ML04: Membership Inference, ML05: Model Theft, etc.)
  ground risk categories that are not negotiable against a latency budget.
  The policy has to say so in writing.
- **Freezing is scoped by SLO ID, not global.** A latency burn on `/chat`
  does not need to freeze `/embed`. Scoped freezes are cheaper and keep the
  policy trusted.

## 3. Validation steps

1. **Every SLO has at least one named policy trigger.** Walk the SLO table in
   Section B against the action table in Section D; any SLO without a trigger
   is a hole.
2. **Every action has a pre-authorized actor.** Rollbacks, degradations, and
   vendor failovers must name who is allowed to execute them without
   additional approval.
3. **Quality burns have a separate clause.** If the policy treats quality burn
   the same as latency burn, the policy is wrong for an AI workload.
4. **Policy resists in-the-moment relaxation.** The policy must explicitly
   forbid lowering an SLO to end a burn (Section E). Otherwise it is not an
   error-budget policy, it is a feelings document.
5. **Interacts with the safety / compliance runbook.** There must be a clause
   saying safety can override; without it the policy quietly contradicts
   NIST AI RMF `GOVERN` requirements.
6. **Signed by product + engineering + SRE + safety.** An unsigned
   error-budget policy will be ignored at the first burn.

## 4. Rubric / review checklist

Score each row independently; passing bar is **14/20** with no row at 0.

| Row | 0 | 2 | 4 |
|---|---|---|---|
| **SLO table complete** | SLOs missing or no windows. | SLO table present but no budget math. | Full SLO table with budgets per window. |
| **Burn-rate alerting** | Not addressed. | Single-threshold alerts only. | Multi-window / multi-burn-rate alerts specified. |
| **Trigger coverage** | <3 trigger types. | Latency + launch freeze only. | ≥ 5 trigger types including quality, cost/saturation, vendor. |
| **Pre-authorization** | All actions require committee approval. | Rollback pre-authorized but nothing else. | Rollback, degradation, and vendor failover all pre-authorized to named roles. |
| **Safety interaction** | No mention. | Mentioned in passing. | Explicit clause: safety / compliance supersedes budget. |

## 5. Common mistakes

- **No quality clause.** The policy only names HTTP failure as "failure." A
  quality regression that hits every eval metric but returns HTTP 200 goes
  unaddressed.
- **No pre-authorization.** Policy says "the team will decide on rollback."
  That is how burns run for hours — nobody can decide to hit the button.
- **"Launch freeze" treated as the only response.** For an AI system, launch
  freeze is one of seven useful levers (rollback, pin, degrade, fail-closed,
  vendor failover, prompt-pin, index-pin). Enumerate all of them.
- **Policy allows SLOs to be relaxed to end a burn.** This is the single most
  damaging failure mode and the Workbook warns about it directly (Ch. 4).
- **No distinction between reproducible and non-reproducible quality burns.**
  Treating them identically leads to bad rollbacks (reverting a fine model
  because of a retriever regression).
- **Vendor-caused burn isn't counted.** Customers don't know which part of
  the stack burned their request. If vendor burn doesn't hit the budget, the
  vendor risk never gets prioritized.
- **No sign-offs.** An error-budget policy that hasn't been signed by
  product is advisory, not binding.
- **Cost not treated as a budget.** If the business committed to a
  $/token number, it is an SLO. A cost burn needs the same policy triggers
  as latency.

## 6. References

- Google SRE Book, Ch. 3 "Embracing Risk", Ch. 4 "Service Level Objectives"
  — <https://sre.google/sre-book/table-of-contents/>.
- *The Site Reliability Workbook*, Ch. 4 "Service Level Objectives"
  (error-budget policy template), Ch. 5 "Alerting on SLOs" (multi-window
  burn-rate alerting) — <https://sre.google/workbook/table-of-contents/>.
- NIST AI Risk Management Framework, `GOVERN` / `MANAGE` functions
  (pre-authorized response pathways; policy & procedure requirements) —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- OWASP Machine Learning Security Top 10 (motivates the safety-override
  interaction, Section F) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
- MITRE ATLAS (threat-informed override cases for Section F) —
  <https://atlas.mitre.org/>.
