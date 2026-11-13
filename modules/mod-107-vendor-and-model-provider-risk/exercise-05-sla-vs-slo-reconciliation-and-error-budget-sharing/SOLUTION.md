# Exercise 05 — SLA vs SLO Reconciliation and Error Budget Sharing (Solution)

## 1. Solution overview

A vendor's SLA and our user-facing SLO are not the same artifact and
cannot be reconciled by picking the smaller number. The SLA is a
contractual availability target backed by service credits; the SLO is
an engineering target for a user journey that may have multiple
dependencies, redundancy, retries, and partial-failure modes.

This exercise produces three artifacts:

1. **A reconciliation worksheet** that maps each user-facing SLO to
   the vendor dependencies on its request path and shows what the
   math says the *achievable* SLO is before any additional
   mitigations.
2. **An error-budget sharing policy** that defines how much of our
   SLO's error budget vendor-caused burn is allowed to consume before
   we take specific actions (policy change, product pause, fallback
   drill, vendor escalation).
3. **A "do-this-when-vendor-SLA-breaks" runbook** that answers:
   when the vendor concedes an SLA breach and offers credits, what
   non-financial actions do we take to keep our own SLO honest?

The framing follows the SRE book's error-budget concept (Google
*Site Reliability Engineering*, chapters on SLOs and Embracing Risk)
and the NIST AI RMF `MANAGE 2.1` guidance on third-party residual-
risk mitigation.

Scope: one user-facing SLO per worksheet. Repeat the worksheet for
each tier-1 surface identified in exercise-01.

## 2. Worked implementation

### 2.1 The vocabulary — stated plainly

- **Vendor SLA.** A contract clause stating the vendor's promised
  availability (or latency, or support response) and a remedy
  (usually service credits) when they miss. SLA measurement is in
  the vendor's hands (what counts as "available", what counts as a
  "minute of downtime") and is defined in the contract.
- **Our SLO.** An internal, engineering-owned target for a specific
  user journey, measured by our telemetry on our side of the
  internet. SLOs are reviewed quarterly; they are not promises to
  users, they are commitments among engineering, product, and
  on-call.
- **Error budget.** `100% − SLO`, converted into a wall-clock
  budget over the SLO window. For an SLO of 99.5% over 30 days,
  the error budget is `0.5% × 30d ≈ 3h 36m` of allowed "bad
  time". The team spends this budget on experiments, releases,
  risk, and tolerating vendor burn.
- **Vendor burn.** The portion of the error budget consumed by
  user-visible impact traceable to an upstream vendor event. Not
  all vendor-side degradation becomes vendor burn; only the slice
  that reaches the user after the gateway's retries and fallbacks
  do their job counts.

A vendor SLA cannot "reconcile" with an SLO in the sense of
equivalence. What it can do is set a *ceiling* on what SLO is
achievable with that vendor in the request path and no additional
engineering.

### 2.2 The reconciliation worksheet

```markdown
# SLO Reconciliation Worksheet — <surface name>

Owner: inference-platform
Last reviewed: <date>

## A. User-facing SLO

| Field | Value |
|---|---|
| Surface | product-chat |
| SLI definition | ratio of successful chat requests, where "success" = 2xx + content != guardrail-blocked + TTFT <= 3 s |
| SLO target | 99.5% over rolling 30 days |
| Error budget | 0.5% × 30 d = 216 minutes |
| Measurement point | Our edge gateway, from user-facing side |

## B. Vendor dependencies on this path (from ex-01)

| Dependency | Role | Vendor SLA (contractual) | SLA measurement point |
|---|---|---|---|
| openai-chat-completions | primary | <see contract; cite verbatim> | vendor-measured, API only |
| anthropic-messages | fallback-a | <see contract> | vendor-measured, API only |
| self-hosted-vllm | fallback-b | internal, 99.9% self-managed | our-side |
| moderation-api | guardrail | <see contract> | vendor-measured |
| cloud-secrets-manager | transversal | <see contract> | vendor-measured |

<!-- cite-primary-source: the SLA numbers above are contract-specific
     and must be copied verbatim from each vendor's current MSA + order
     form. Do not infer them from marketing pages. -->

## C. Pre-mitigation achievable ceiling

For independent dependencies in series, the ceiling is the product of
their availabilities. Example (hypothetical, to illustrate method):

Suppose each vendor row claims 99.9% SLA, and both the primary and
guardrail are strictly required with no fallback. Then:

ceiling = 0.999 × 0.999 × 0.999 ≈ 99.70%

That ceiling is *before* our own code adds failures and *before*
accounting for vendor fallback. Our SLO at 99.5% fits under 99.70%
with ~0.20 percentage points of headroom — roughly 86 of our 216
minutes of monthly error budget — to spend on everything that is
not vendor-attributable (our bugs, our deploys, our chaos drills,
network).

Formula (assume independent failures — document the assumption):

    A_path = Π_i A_i                              (series, no fallback)
    A_path_with_fallback = 1 − Π_i (1 − A_i)      (parallel, perfect switch)

For the chat surface with primary + one fallback available on
chat-specific failures:

    A_chat = (1 − (1 − 0.999) × (1 − 0.999)) × 0.999_guardrail
           ≈ 0.99999801 × 0.999 ≈ 99.8998%

The fallback buys ~0.2 percentage points of ceiling — but only to the
extent the fallback actually works during a primary outage (ex-02
drill results, not advertised SLA).

## D. Mitigations that move the ceiling

| Lever | Effect on ceiling | Prerequisite |
|---|---|---|
| Add a fallback provider | Shifts path from series to parallel for the primary term | ex-02 contract with tested drill |
| Remove a tier-1 vendor from critical path | Drops a factor out of the product | Product accepts the degraded mode |
| Make a tier-1 vendor a soft dependency (async + cache) | Reclassifies as non-blocking | Caller code change |
| Partial-SLO: accept degraded quality during vendor outages | Narrows SLI to availability-only | Product sign-off |

## E. SLO sanity check

- Is our SLO ≤ the achievable ceiling? If no, the SLO is a wish, not
  a commitment. Either lower the SLO or add a mitigation from §D.
- Is our SLO ≫ the ceiling? Excess headroom is fine engineering but
  often hides where the team actually spends budget (deploys, chaos).
  Check that observed burn attribution matches intention.
```

### 2.3 Error-budget sharing policy

This policy answers: of the N minutes of error budget per window, how
much are we willing to let vendors consume, and what happens when
they do?

```markdown
# Error-Budget Sharing Policy — <surface>

## 1. Attribution rules

Each minute of "bad time" counted toward the error budget is tagged
with exactly one owner:

- `vendor:<id>` — The impacting event is traceable to a vendor
  dependency. The gateway's `x-served-by` tag + vendor-status
  pipeline (ex-04) is the primary evidence. Fallback that worked
  does not count — only impact that reaches the user.
- `own:deploy` — Caused by a change we deployed.
- `own:config` — Caused by a config or data-pipeline change.
- `own:infra` — Our compute, network, or storage.
- `unknown` — Investigate within 7 days or the budget line is
  automatically attributed to `own:infra`.

Attribution is performed within one business day of the end of each
incident.

## 2. Sub-budgets (per 30-day window)

For a 216-minute monthly error budget on a 99.5% SLO:

| Bucket | Soft cap | Hard cap |
|---|---|---|
| vendor:primary (single provider) | 60 min | 100 min |
| vendor:fallback (also down) | 30 min | 50 min |
| vendor:guardrail | 20 min | 30 min |
| vendor:transversal (secrets, egress) | 20 min | 30 min |
| own:* (sum of our own-attributed burn) | 60 min | 90 min |
| reserved | 36 min | — |

Caps are soft until the "action ladder" triggers below. They are
hard when the reserved slice is also empty.

## 3. Action ladder

Actions trigger when a bucket exceeds its soft cap within the window.

- **Soft cap exceeded, one vendor**: open a ticket to the owning team
  + the vendor-account team to review vendor SLA claims against
  observed burn. Notify product. No release freeze.
- **Soft cap exceeded, two vendors in same window**: pause non-
  critical releases that increase this surface's risk. Trigger an
  ex-02 fallback drill against the affected primary within 1 week.
- **Hard cap exceeded on any vendor bucket**: engineering lead +
  product lead jointly decide whether to:
    1. Request a schedule for vendor remediation (with their
       engineering counterparts, not only account team), AND
    2. Shift primary to the fallback for the remainder of the
       window, AND/OR
    3. Reduce traffic from the affected surface (e.g., turn off a
       secondary feature) to re-earn budget.
- **Our-attributed burn ≥ 60 min**: per-team release-freeze policy,
  independent of vendor status.
- **Reserved slice used**: the SLO is failed for this window. We do
  NOT chase SLA credits as a substitute for honest SLO reporting;
  both can be true.

## 4. Non-goals

- This policy does not grant permission to ignore a vendor outage
  because "they still owe us SLA credits." Credits compensate for
  dollars, not for user-visible failure.
- This policy does not treat vendor burn as free. Vendor-attributed
  minutes still subtract from the surface's budget.

## 5. Review

- Burn by bucket posted weekly to the reliability review.
- Policy reviewed each quarter; sub-budget caps adjusted based on
  the last four windows' actual attribution.
```

### 2.4 Worked-example math

Illustrative; numbers are stand-ins so the computation is explicit.
Replace with actual vendor SLA numbers and observed burn when
applying.

```
Window: 30 days = 43,200 minutes
SLO:    99.5% → error budget = 216 minutes

Observed burn, last window (from incident timelines + vendor status
ingester):
  vendor:openai            82 min  (primary, two events)
  vendor:anthropic          4 min  (coincident partial degradation)
  vendor:moderation         0 min
  vendor:secrets           11 min  (one brief auth-plane event)
  own:deploy               15 min  (bad release reverted in 15 min)
  own:config                6 min
  unknown                   8 min
  ------------------------------
  total                   126 min  (within 216-minute budget)

Policy check:
  vendor:primary 82 > soft cap 60 → open vendor review ticket
  vendor:primary 82 < hard cap 100 → no primary cutover
  own:*          21 < soft cap 60 → no release freeze
```

If the OpenAI entry had been 120 min, the hard cap would have been
crossed; the engineering lead would have scheduled a primary shift
to Anthropic and reduced traffic from one secondary feature for
the remainder of the window. If the policy said "SLA covers this,"
the user still experienced 120 minutes of impact; the policy
language in §4 is deliberately against trading honesty for
credits.

### 2.5 Reconciliation table — SLA vs our SLO, side by side

For the reader who wants the one-page contrast the exercise title
asks for:

| Property | Vendor SLA | Our SLO |
|---|---|---|
| Who defines it | Vendor (contract) | We do, with product |
| What's measured | Narrow service availability | User journey outcome (incl. correctness, latency) |
| Where measured | Vendor-side telemetry | Our edge |
| Window | Usually monthly billing cycle | Usually 28- or 30-day rolling |
| Breach consequence | Service credits | Engineering action (freeze, drill, cutover) |
| Reviewed when | At contract renewal | Quarterly |
| Owner | Finance + legal + eng | Eng + product |
| Composable? | No; each vendor's SLA stands alone | Yes; one SLO per surface, hierarchical |
| Useful as a target to build to? | No — too narrow | Yes — this is the engineering target |
| Useful as an upper bound? | Yes — ceiling computation (§C) | N/A |

Vendor SLA is an *input* to the SLO ceiling calculation, not a
substitute for an SLO. Teams that confuse the two end up with
either an SLO that is impossible given the vendor set, or an SLO
that ignores vendor-side risk until it burns.

## 3. Validation steps

1. **Worksheet completeness.** Every tier-1 surface in the ex-01
   surface map has a §A–§E worksheet.
2. **SLA citations.** Every SLA cell has a quote and a link to the
   current signed order form or MSA, not to the vendor's public
   pricing page.
3. **Ceiling arithmetic.** The product (or parallel formula) in §C
   is actually computed; the assumption of independence is stated.
4. **SLO ≤ ceiling.** Flag any surface whose SLO exceeds the
   computed ceiling; those are the first candidates for §D
   mitigations.
5. **Attribution dry run.** Take the last three incidents on each
   surface; attribute each minute per §2.3. If any minute cannot be
   attributed in the current observability stack, open a ticket to
   fix the gap — the policy only works if attribution works.
6. **Policy caps against observed burn.** Caps in §2.3 §2 are not
   chosen abstractly. Plot the last four windows' actual burn by
   bucket; caps should be above the median and reachable by the
   ladder. Caps that are never hit are policy theater; caps that
   are always hit are policy inflation.
7. **Runbook exists.** The "do-this-when-vendor-SLA-breaks"
   runbook from §1 is linked from the worksheet; its first step is
   to check the vendor-status ingester (ex-04) and the fallback
   contract (ex-02), not to email the vendor account team.
8. **Quarterly review record.** The worksheet has a dated review
   line. If the newest review is older than one quarter, the
   artifact is stale and fails this check.

## 4. Rubric

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Vocabulary clarity** | SLA and SLO conflated or undefined | Defined but one-line | Defined with measurement, owner, and breach consequence columns (§2.5) |
| **Dependency ceiling math** | No calculation | One-term product | Full path computed, with parallel-fallback where applicable, assumptions stated |
| **SLA sourcing** | Marketing-page numbers | Mix of marketing and contract | Every SLA cited from the signed contract; marketing pages excluded |
| **Error-budget attribution** | Not defined | Buckets exist | Buckets + per-incident attribution rule + known-unknown handling (`unknown` auto-attributes after 7 days) |
| **Sub-budget caps** | Single cap or absent | Soft/hard caps assigned by vendor | Soft + hard, evidence-based from historical burn, with quarterly review |
| **Action ladder** | "We'll figure it out" | Vendor review action only | Multi-step ladder keyed to caps, includes non-vendor actions (release freeze, traffic reduction, cutover) |
| **Policy honesty** | Treats SLA credits as substitute for SLO burn | Mentions distinction | Explicit "credits don't erase burn" clause, cited |
| **Validation runbook** | None | Shallow | Runbook links ex-02 fallback and ex-04 ingester; first step is internal detection, not vendor email |
| **Linkage** | Standalone doc | References ex-01 | References ex-01 (dependencies), ex-02 (fallback math), ex-04 (ingester timestamps for attribution) |

Passing: average ≥ 1.5 with no 0s on **Dependency ceiling math**,
**SLA sourcing**, **Error-budget attribution**, **Action ladder**.

## 5. Common mistakes

- **Setting the SLO to match the vendor's SLA.** That gives up the
  engineering target completely. The SLO should be lower than the
  computed ceiling, not equal to a single vendor's SLA.
- **Treating service credits as reliability outcomes.** Credits are
  dollars; users experienced minutes. Both must be tracked. The
  ladder's hard-cap actions are not negotiable for credit size.
- **Multiplying availabilities without an independence claim.**
  If primary and fallback share egress, auth, or CDN vendors, their
  failures are correlated and the parallel formula overstates
  resilience. Compute the ceiling only against components the ex-01
  map says are independent at those layers.
- **Omitting transversal dependencies.** Secrets manager, identity,
  egress proxy — their outages affect every surface simultaneously
  and belong in the §B table and §C math.
- **Attribution-by-feel.** "This felt like an OpenAI thing" is not
  attribution. The `x-served-by` tag + the ex-04 timeline of vendor
  status + our error-rate panels are the three sources attribution
  requires.
- **Caps that drift.** Sub-budget caps that are not reviewed against
  actual burn each quarter either lock in old pain (caps too low,
  constant freezes) or hide drift (caps too high, SLO quietly
  deteriorating).
- **Hiding "unknown" burn.** Unassigned minutes are a measurement
  gap. The 7-day auto-attribution rule to `own:infra` is
  deliberately strict; it forces investment in observability.
- **Treating the SLO as public commitment.** SLOs are internal
  engineering commitments. Share them with users only if your
  product team wants to turn that into a user-facing SLA — a
  decision with legal and support implications not covered here.

## 6. References

- **Google SRE book** — *Site Reliability Engineering* (Google,
  2016), chapters "Embracing Risk" and "Service Level Objectives".
  Online copy: <https://sre.google/sre-book/embracing-risk/> and
  <https://sre.google/sre-book/service-level-objectives/>. The
  canonical treatment of error budgets, SLI/SLO/SLA distinction, and
  sub-budget attribution.
- **Google SRE Workbook** — *The Site Reliability Workbook*
  (Google, 2018), chapter "Implementing SLOs" at
  <https://sre.google/workbook/implementing-slos/>. Covers error-
  budget policies, action ladders, and multi-stakeholder
  negotiations in detail.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MANAGE
  2.1` (risk-response options for third-party components),
  `MEASURE 4.1` (ongoing performance measurement) are the
  subcategories the worksheet is evidence for.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  `ML06 AI Supply Chain Attacks` reinforces that vendor dependence
  is a modeled risk — the error-budget policy's "vendor" buckets
  are the operational translation.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The reconciliation's
  insistence on measurement-at-our-edge is the reliability analogue
  of the "measure at the defender's boundary" principle in ATLAS
  detections.
- **Upstream artifacts.** `exercise-01-vendor-dependency-surface-map`
  provides the dependency list and independence claims; `exercise-02`
  provides the fallback math; `exercise-04` provides the
  attribution timestamps.
- **Downstream.** `exercise-06-vendor-eval-scorecard-for-single-vs-
  multi-provider` consumes the achievable-ceiling delta between
  single- and multi-provider path for the scorecard's
  "reliability" row.
