# Exercise 04 — Cost SLO per Million Tokens and per Session — Reference Solution

## 1. Solution overview

Cost SLI block for the **`acme-llm-gw`** workload introduced in
exercise 01. Four cost SLIs are derived from an explicit business
unit-economics model — the exercise is primarily about the
*derivation chain*, not the number at the end. All values are
illustrative, but the arithmetic is live: a reviewer can swap the
revenue-unit numbers and the targets recompute mechanically.

The block is designed to drop into the composed stack from exercise
02 without disturbing the sum-of-shortfalls arithmetic (cost SLIs
are one-sided ceilings and do not participate).

---

## 2. Worked answer / Implementation

### 2.1 Section 1 — Business unit-economics model

- **Revenue unit.**  `chat-stream`: $20 per subscriber per month
  (illustrative consumer price point). `code-assist`: $0.004 per
  completion accepted (illustrative per-call API price).
  `batch-summarize`: $0.02 per ticket summarized (illustrative
  internal chargeback rate to the customer-support team).
- **Target gross margin on inference.**  60% on `chat-stream` and
  `batch-summarize`; 50% on `code-assist` (tighter because the
  per-call API has less margin headroom, consistent with
  commodity-API pricing patterns).
- **Cost ceiling per revenue unit.**  Revenue × (1 − gross margin).
    - `chat-stream`: $20 × 0.40 = **$8 / subscriber-month**.
    - `code-assist`: $0.004 × 0.50 = **$0.002 / accepted
      completion**.
    - `batch-summarize`: $0.02 × 0.40 = **$0.008 / ticket**.
- **Traffic profile per revenue unit.**
    - `chat-stream`: a subscriber makes ~50 sessions / month,
      ~400 input + 800 output tokens per session → 20k input
      tokens and 40k output tokens per subscriber-month.
    - `code-assist`: a single completion consumes ~1500 input
      tokens (context window with surrounding code) and ~150
      output tokens; acceptance rate is ~25% so 4 invocations
      per accepted completion → 6000 input + 600 output tokens
      per accepted completion.
    - `batch-summarize`: per ticket, ~5000 input tokens
      (ticket + replies) and ~300 output tokens (the summary).

The derivation is the artifact. The reviewer can interrogate every
number: "why 60% margin?" is a product-finance conversation, not
an SRE conversation; "why 20k input tokens per subscriber-month?"
is a measurement question that `mod-104` capacity instrumentation
can answer.

### 2.2 Section 2 — The four cost SLIs

| SLI | Target ceiling (illustrative) | Derivation (arithmetic chain) | Window | Burn-rate alert config |
|---|---|---|---|---|
| Cost per 1M **input** tokens (`chat-stream`) | ≤ **$80 / 1M input tokens** | Allowed input-token spend per subscriber-month = $8 × (input fraction of spend; assume 30% split based on provider's 1:5 input:output price ratio and 1:2 input:output token ratio) ≈ $2.40 ÷ 20k input tokens = $120 / 1M input tokens. Set target ceiling at **$80** to reserve 33% headroom for traffic mix drift. | 28-day rolling | Medium-burn ticket (6× = $480 / 1M); slow-burn review (1× = $80 trend); fast-burn page **only when correlated** (see §2.4) |
| Cost per 1M **output** tokens (`chat-stream`) | ≤ **$120 / 1M output tokens** | Allowed output-token spend per subscriber-month = $8 × 0.70 = $5.60 ÷ 40k output tokens = $140 / 1M output tokens. Set target ceiling at **$120** to reserve ~15% headroom. | 28-day rolling | Same alert shape as input-token SLI |
| Cost per **completed request** (`code-assist`) | ≤ **$0.00035 / completion served** (not per accepted) | Cost ceiling per accepted completion $0.002 × acceptance-rate 0.25 = $0.0005 per served invocation; set target ceiling **below** that allowance to reserve 30% headroom for acceptance-rate drift → **$0.00035 per served completion** (which implies $0.0014 per accepted, inside the $0.002 margin budget) | 28-day rolling | Same alert shape |
| Cost per **successful session** (`chat-stream`) | ≤ **$0.16 / successful session** | $8 / subscriber-month ÷ 50 sessions/month = $0.16 / session. No headroom reserved — this is the *subscriber-level* contract and the business is willing to defend it at the boundary | 28-day rolling | Same alert shape, plus a correlated-paging exception when it correlates with the input-token-rate abuse signal (§2.4) |

Notes on the derivation.

- The 30% / 70% input-vs-output split is derived from the
  provider's published 1:5 input:output pricing ratio applied to
  the 1:2 input:output *token* ratio: input dollar share is
  `1 / (1 + 5 × 2) = 1/11 ≈ 9%` for pure provider cost, but
  rounded to 30% after factoring in prefix-cache spend, trace
  export cost, and other request-path spend that accrues on the
  input side.
- `batch-summarize` cost SLIs are omitted from the primary table
  because the workload is already governed by a per-ticket
  chargeback target ($0.008 / ticket). The cost SLI for the batch
  surface is `cost-per-ticket ≤ $0.008` with ticket-only alerting;
  it is not expressed as a per-1M-token SLI because that
  abstraction loses signal on a small, bounded workload.
- `code-assist` cost per successful session is **N/A** — the
  surface is single-turn with no session concept. Explicitly
  marked N/A in the stack from exercise 02.

### 2.3 Section 3 — Product-ask pushback

**Ask (chosen because it is a realistic product request).**  "Use
the frontier-tier model on 100% of `chat-stream` traffic."  Current
mix is ~20% frontier / 80% primary-tier.

1. **Model the cost of the ask.**  Provider pricing (illustrative):
   frontier $20 / 1M output tokens; primary $8 / 1M output tokens.
   At 100% frontier, output-cost mix moves from `0.2 × 20 + 0.8 ×
   8 = 10.4` to `20`. That is a **1.92× output-cost multiplier**
   on `chat-stream`. Applied to our target: cost-per-1M-output
   target would move from $120 to ~$230; cost-per-successful-session
   would move from $0.16 to ~$0.30.
2. **Translate into business margin terms.**  At current pricing,
   the ~$0.14 incremental cost per session pushes subscriber
   gross-margin from 60% to **~25%** — below the 50% floor the
   product organization has committed to externally. Over a
   100,000-subscriber base, that is a $14k/month gross-margin
   shortfall before accounting for other cost lines.
3. **Alternative.**  Route to frontier only when the lightweight
   model's quality SLI (hallucination or tool-call-success) is
   trending worse on the specific traffic slice — "frontier for
   long-reasoning turns, primary for short exchanges." Cost mix
   moves to ~30% frontier, output-cost multiplier ~1.2×,
   cost-per-1M-output ceiling moves to ~$145. Gross-margin falls
   from 60% to ~56% — a defensible number, not a floor-breaching
   one. The margin gap is the specific cost of the targeted-
   routing discipline; the stack admits that trade-off explicitly.

### 2.4 Section 4 — Ticket-only alerting with correlated-paging exceptions

**Pages (cost burn with correlated signal).**  Three correlated-burn
triggers escalate cost signals to the on-call:

- **Input-token rate spiking without a corresponding rise in
  session count.**  This is the abuse / credential-leak signature
  — a small number of adversarial clients driving a disproportionate
  share of the input-token budget. Triggers a page to
  `ai-infra-security` jointly with SRE on-call.
- **Cost-per-successful-session burning while toxic-output SLI is
  also burning.**  The hypothesis is an abuse campaign (coordinated
  jailbreaks) that both inflates cost and depresses safety. Pages
  jointly to SRE on-call and `ai-infra-security`.
- **Provider-pricing-change notification received** (via provider
  status feed) that would, applied forward, breach the cost-per-1M-
  output SLI within 24 h.  Pages the SRE on-call so they can
  engage product / finance before the contract silently breaks.
  (Also referenced in `mod-107` vendor-risk coordination.)

**Ticket to the SRE queue (routine cost burn).**

- Cost-per-1M-input-tokens medium or slow burn — a traffic-mix
  shift that isn't an incident.
- Cost-per-1M-output-tokens medium or slow burn — a model-mix shift.
- Cost-per-completed-request medium or slow burn on `code-assist`.
- `batch-summarize` cost-per-ticket medium or slow burn.

**Route directly to the SLO review (slow-burn, policy-shape
signal).**

- Any cost SLI slow-burn over 3 d that is not already explained
  by an open ticket.
- An aggregate cost increase ≥ 10% that is distributed across
  multiple SLIs (no single SLI burning) — the composite signal
  is a cost-strategy issue, not an operations one.

### 2.5 Section 5 — Instrumentation gaps

Chapter 04 §7 names five required signals. Status per `acme-llm-gw`:

- **Provider billing feed ingested.**  Partial — the primary
  provider's billing feed is ingested nightly; the secondary
  provider used for `provider-secondary` fallback is reconciled
  at month-end only. **Gap**: secondary-provider real-time feed
  needs to land before cost-per-1M SLIs on fallback traffic are
  operable. Filed ticket: `platform-cost-02`.
- **Per-request token-count on the trace / log.**  In place.
  Attributes: `gen_ai.usage.input_tokens`,
  `gen_ai.usage.output_tokens`, `gen_ai.usage.total_tokens` emitted
  per OpenTelemetry Generative AI Semantic Conventions. No gap.
- **Model + provider tag on every response.**  In place.
  Attributes: `gen_ai.response.model`, `gen_ai.system`,
  `provider.region`. No gap.
- **Session identifier linking related requests.**  **Gap** —
  session IDs exist on the `chat-stream` surface but are *not*
  propagated into the OTel span attribute. The cost-per-successful-
  session SLI is in warn-only mode until the session ID reaches
  the span root. Tracked as instrumentation ticket
  `platform-cost-03` and blocks cost SLI section going fully
  operable.
- **Self-hosted cost model.**  N/A — `acme-llm-gw` is fully
  vendor-hosted. If the workload adopts a self-hosted model in
  the future, the `GPU-hour-cost × fraction-of-GPU-used-by-
  workload ÷ tokens-produced-per-hour` formula from `mod-106`
  capacity planning becomes the SLI's basis.

Summary: **two concrete open gaps** (`platform-cost-02`,
`platform-cost-03`) blocking two of the four SLIs from being fully
operable. Both named owners (SRE); both in the next-quarter capacity
plan.

---

## 3. Validation steps

1. Confirm §2.1 has all four unit-economics elements with
   arithmetic shown (not just the result).
2. Confirm §2.2 has the four cost SLIs with a derivation column
   that traces target back to the §2.1 model.
3. Confirm §2.3 pushback has all three steps and the margin-term
   translation visible.
4. Confirm §2.4 has three alerting lists (page / ticket / review)
   and the paging exceptions are correlated-signal rules, not
   plain cost thresholds.
5. Confirm §2.5 names at least two concrete instrumentation gaps
   with owners.
6. Confirm the cost block drops into the exercise-02 stack: the
   four rows appear with "one-sided ceiling" markers and do not
   enter the sum-of-shortfalls arithmetic.
7. Word-count check against ≤ 1500 word acceptance criterion.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Unit-economics model with arithmetic | Revenue, margin, cost ceiling, traffic profile all shown with math | "Healthy margin" without a number |
| R2 | Four SLIs with derivation chain visible | Each target traces back via arithmetic to the §2.1 model | Target stated without derivation |
| R3 | Pushback translates ask into margin terms | Three steps complete with specific margin delta | Pushback stated in latency / general terms only |
| R4 | Paging exceptions are correlated-signal rules | Each paging trigger names the second signal | Paging on cost alone |
| R5 | Instrumentation gaps named with owners | Two concrete open gaps with ticket refs | "No gaps" without defence |
| R6 | Cost block integrates into exercise-02 stack | Rows listed as one-sided ceilings; composite arithmetic unchanged | Cost rows enter the sum-of-shortfalls |
| R7 | Finance peer resolves "what is the cost contract" in 15 min | Dry-run passes | Requires clarifying conversations |

---

## 5. Common mistakes

- **Setting a target by SRE intuition.**  Any cost target that
  cannot be defended by the unit-economics chain breaks on the
  first sponsor-review push.
- **Choosing the input/output split arbitrarily.**  Provider
  pricing dictates the economic split; the token ratio dictates
  the physical split. Reconciling the two is the main
  derivation step.
- **Treating cost burn as pageable by default.**  Cost burn is
  ticket-only *except* in the correlated cases named in §2.4.
  Paging on a cost spike that turns out to be a legitimate
  traffic mix shift is noise.
- **Omitting the session ID gap.**  Cost-per-successful-session
  is often the most *useful* cost SLI for a chat product. If the
  session ID is not on the span, the SLI cannot be computed
  reliably.
- **Not splitting the cost SLI by model.**  An aggregate cost-
  per-1M-output can look fine while the mix shifts to the cheap
  model (quality regression masked as cost improvement). The
  stretch goal in the learning exercise names this; the review
  should open the per-model breakouts when the mix is in flux.
- **Taking the push-back as a refusal tool.**  The push-back
  frames the *trade-off* and names the *alternative*. A review
  with the sponsor should treat it as a negotiation, not a wall.

---

## 6. References

- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source
  for `gen_ai.usage.*`, `gen_ai.response.model`, and
  `gen_ai.system` attributes referenced in §2.5.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for
  the burn-rate multipliers and the "ticket vs. page" separation
  discipline applied in §2.4.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
  Source for the governance framing on cost-and-safety-correlated
  paging exceptions.
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the abuse-signature framing in §2.4 (cost burn
  correlated with policy-violation signals).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for
  adversarial-use-pattern framing underlying the "input-token
  rate spike without session growth" abuse trigger.
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-04-cost-slo-per-million-token-and-per-session.md`
  (learning exercise this solution answers) and exercises 01–03
  SOLUTION.md (the workload this document extends).
