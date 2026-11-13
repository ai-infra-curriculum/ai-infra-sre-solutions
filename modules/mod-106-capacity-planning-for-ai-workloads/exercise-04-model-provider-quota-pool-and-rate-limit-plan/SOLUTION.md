# Exercise 04 — Model-Provider Quota Pool and Rate-Limit Plan — Solution

## 1. Solution overview

When a team serves traffic through hosted model providers (OpenAI,
Anthropic, Google, Azure/Bedrock gateways to any of the above), the
provider's quota and rate limits are a **primary capacity constraint**,
not a secondary one. Exceeding them produces 429s, 529s, or soft
throttling — which the service-level SLOs do not forgive even though the
root cause is external.

This exercise asks the learner to produce a **quota pool and rate-limit
plan**: the shared budget across workload classes, the per-class share,
the backpressure policy when a limit is hit, and the escalation path.

The reference solution below delivers:

1. A quota pool schema — what the pool represents, what it does not, and
   the units it lives in.
2. A fair-share allocation rule across workload classes with explicit
   handling for priority, burst, and fallback.
3. A client-side rate-limit discipline (token-bucket + jitter + retries)
   that keeps the pool legal.
4. A backpressure and admission-control playbook for when the pool is
   exhausted or the provider soft-throttles.
5. A rubric so the plan is scored in a fixed-length review.

The plan is provider-agnostic in shape. Provider-specific numbers
(RPM / TPM / RPD limits, specific 429 headers, specific model IDs) are
filled in by the learner from current provider documentation at the time
the plan is written — none of those are hard-coded here because they
drift. The *shape* of the plan does not drift.

## 2. Worked implementation — the plan

### 2.1 Pool schema

A provider quota pool is identified by the tuple
`(provider, account, region, model-family)`, because those are the
dimensions along which providers scope their own limits. Treating two
model IDs under the same provider as the same pool is a common mistake
that under-counts headroom on one and over-counts on the other.

| Field | Meaning |
|---|---|
| `provider` | e.g. OpenAI, Anthropic, Google, Azure OpenAI, Bedrock |
| `account` | The billing identity / org whose limits apply |
| `region` | Where request routing lands; limits differ by region on some providers |
| `model_family` | The limit scope (e.g. all `gpt-4o-*` vs `o1-*`); NOT a specific model ID unless the provider scopes per-model |
| `rpm_cap` | Requests-per-minute limit as currently documented |
| `tpm_cap` | Input+output tokens-per-minute limit (or separate input/output caps where the provider separates them) |
| `rpd_cap` / `tpd_cap` | Daily caps where they apply |
| `burst_policy` | What the provider does at the moment the cap is crossed (hard 429 vs soft throttle vs sliding window) |
| `fail_mode_signal` | The header or status the provider emits (`retry-after`, rate-limit-remaining, etc.) |
| `upgrade_sla` | How long a tier-increase request takes to land, in business days |
| `fallback_pool_id` | The pool to shed to when this one is exhausted |

Rules the schema must enforce:

- `rpm_cap` and `tpm_cap` are both *refreshed* against the provider's
  current docs at plan time and carry a date stamp. Quota tiers change.
- `model_family` is the scope the provider uses; if the provider
  switched to per-deployment quota (as on Azure), model-family becomes
  the deployment ID.
- Every pool has a `fallback_pool_id`. The only allowed exception is the
  terminal fallback (local self-hosted model, cached response, graceful
  degradation), which must be named explicitly.

### 2.2 Fair-share allocation across workload classes

The pool is divided across the workload classes from the Exercise 02
worksheet. The rule is **share by priority with a hard reservation for
interactive**:

1. Give each workload class a *reservation* (e.g. 50% for
   `chat_stream`, 30% for `rag_qa`, 20% for `batch_summary`). The sum of
   reservations must be ≤ 100%.
2. Allow classes to **borrow** unused reservation from lower-priority
   classes, but only if the borrower is at or below its reservation AND
   the lender is at or below *its* reservation.
3. **Interactive wins preemption.** When the pool is saturated and an
   interactive class is at its reservation while a batch class is above
   its reservation, batch gets throttled first (admission-control rule
   below).
4. Reservations are published, auditable, and reviewed with the
   workload-class SLOs every planning cycle. A reservation that has
   never been changed in a year is almost certainly wrong.

Worked example (using Exercise 02 classes):

| Class | Priority | Reservation (of `rpm_cap`) | Reservation (of `tpm_cap`) |
|---|---|---|---|
| `chat_stream` | interactive | 50% | 45% |
| `rag_qa` | interactive | 30% | 35% |
| `batch_summary` | batch | 20% | 20% |

The `tpm_cap` and `rpm_cap` percentages diverge because classes have
different `L_in` / `L_out`: a class with large prompts uses more TPM per
RPM than a chat stream. Run both splits independently.

### 2.3 Client-side rate-limit discipline

The client must stay below `rpm_cap` and `tpm_cap` *without* relying on
the provider to enforce them. The provider's enforcement is a last
line of defence; relying on it causes TTFT variance from retry storms.

Base pattern per workload class:

- **Token bucket** per `(pool, class)`. Capacity = class reservation of
  the pool's TPM and RPM. Refill rate = the per-second share of the
  reservation.
- **Admission check before send.** If the bucket for either RPM or TPM
  is empty, enqueue (bounded queue, see 2.4) rather than send
  immediately.
- **Retry policy on 429/529.** Honour the provider's `retry-after`
  header where present. Otherwise: exponential backoff with full
  jitter, bounded at a small number of retries (≤3) *and* with a hard
  deadline so a retry never breaches the request's TTFT SLO.
- **Pool health signal.** Expose a `rate_limit_remaining` gauge per
  bucket so the HPA and the dashboards can see headroom. The HPA in
  Exercise 05 consumes this.

What the discipline is *not*: it is not a replacement for the server-side
admission controller (see 2.4). The token bucket smooths and shapes; the
admission controller sheds.

### 2.4 Backpressure and admission-control playbook

When the pool is exhausted, the plan must specify what actually
happens to a request. Four states, in order:

1. **Green — pool at <80% of reservation.** Pass-through.
2. **Yellow — pool at 80%–95% of reservation.** Enable soft admission
   control: route batch and background classes through a bounded
   queue (queue timeout = TTFT budget of the class), start evicting
   stale queued entries, log structured events.
3. **Red — pool at 95%–100% of reservation.** Shed batch and
   background classes entirely with a 503 + `Retry-After`. Interactive
   classes keep passing through until their reservation hits 100%.
4. **Overflow — any interactive class at 100% of its reservation.**
   (a) Divert to `fallback_pool_id` if its own headroom allows it.
   (b) If no fallback has headroom, degrade gracefully (shorter output
   cap, lower-quality model, cached answer where the use case permits)
   rather than return a user-facing failure.

Each state has a dashboard panel, an alert, and a runbook link. The
plan must list them by name so the on-call does not have to improvise.

### 2.5 Tier upgrade and procurement lane

Every provider has a path to raise `rpm_cap` and `tpm_cap`. The plan
captures:

- Current tier, how it was obtained (self-serve / support case / vendor
  manager).
- Lead time to the next tier.
- The forecast threshold that triggers the upgrade request (not "when we
  run out" but "when forecast hits 70% of current cap for two consecutive
  weeks" or similar).
- The on-call name and the vendor-side contact.

A plan that says "we'll ask for more when it breaks" has failed the
exercise — the lead time is always longer than the tolerance.

### 2.6 Multi-provider / fallback pool design

The `fallback_pool_id` on each pool must point to a *different vendor
or a different account/region*, not another model on the same account.
Rate-limit overflow usually correlates across models on one account;
diversification requires a genuinely separate failure domain. If
model-equivalence testing is part of the plan, the fallback is validated
against the same evals as the primary.

The plan must state what the fallback can and cannot do (model output
equivalence, feature parity around tools and streaming, latency delta,
cost delta). A fallback pool that cannot serve tool-calls is not a
drop-in; the admission controller routes the tool-call class differently.

## 3. Validation steps

1. **Pool identity is `(provider, account, region, model-family)`.** A
   plan with one row per provider is under-specified and will miscount
   shared limits.
2. **Caps carry a date stamp** and cite the provider's documentation at
   that date. Caps change as providers reorganise tiers.
3. **Reservations sum to ≤100% for both RPM and TPM.** A plan that
   over-subscribes the pool is not a plan; it is a wish.
4. **Every pool names a `fallback_pool_id` or a terminal degradation
   mode.** No empty fallback cells.
5. **Admission-control states (Green / Yellow / Red / Overflow) each
   name an alert and a runbook link.** A playbook referenced only in
   prose is not operational.
6. **Retry policy honours `retry-after` and caps retries with a
   deadline tied to TTFT.** A retry storm is worse than a clean 429.
7. **Tier upgrade trigger is numeric and ahead of exhaustion**, with a
   lead time written in business days.
8. **Fallback pool is a different vendor/account/region**, not a model
   on the same account.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | Pool schema includes the four-tuple identity, both caps, burst and
      fail-mode policies | 2 |
| 2 | Fair-share allocation sums to ≤100% on both RPM and TPM, with
      interactive preemption rule | 2 |
| 3 | Client-side token-bucket discipline per `(pool, class)`, with
      honouring of `retry-after` and TTFT-bounded retries | 2 |
| 4 | Four-state admission-control playbook, each state with an alert
      and a runbook | 1 |
| 5 | Tier upgrade trigger is a numeric threshold ahead of exhaustion,
      with lead time in business days | 1 |
| 6 | Fallback pool is a genuinely separate failure domain (different
      vendor/account/region) and its parity is stated | 1 |
| 7 | Caps carry a date stamp and cite the provider doc | 1 |

Deductions:

- Fallback is another model on the same account: −2.
- Retry policy is unbounded or ignores `retry-after`: −2.
- Reservations sum > 100%: −3.
- No tier-upgrade trigger: −1.
- Caps have no date stamp: −1.

## 5. Common mistakes

- **Treating RPM and TPM as one limit.** They are different; a bursty
  short-prompt class can exhaust RPM while TPM is comfortable, and vice
  versa.
- **One pool per provider.** Providers scope limits by account, region,
  and (sometimes) model deployment. Collapsing to one pool mis-attributes
  headroom.
- **Letting batch starve interactive.** Without a reservation floor,
  batch workloads will consume the pool at the exact moment an
  interactive burst arrives.
- **Retrying without jitter.** Correlated retries after a 429 produce a
  second, larger 429 wave. Full jitter is non-negotiable.
- **Fallback is a different model on the same provider.** It fails with
  the primary when the primary fails.
- **"Ask for more when it breaks."** The lead time is longer than the
  budget. Trigger ahead of exhaustion.
- **Static reservations.** Reservations that are never re-tuned do not
  track the real workload mix. Review them with the capacity worksheet.
- **No client-side limiter; relying on provider enforcement.** The
  provider returns 429 with variable latency; your TTFT SLO will not
  survive this.
- **Shadow traffic counted against the pool unmarked.** Evals and
  shadow runs should either be on a dedicated quota or explicitly
  budgeted; a plan that forgets them will overflow under load.

## 6. References

- NIST AI Risk Management Framework (AI RMF 1.0) — Manage function,
  MG-3 (third-party and supplier risk) — upstream provider risk is a
  measured AI reliability risk, not a cost line item —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- OWASP ML Security Top 10 — supply-chain-style risks (vendor lock-in,
  vendor-service availability) that the fallback-pool column exists to
  address —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
- MITRE ATLAS — tactic entries on supply-chain and model-provider
  availability threats that justify multi-provider diversification —
  <https://atlas.mitre.org/>.
- Google SRE Workbook — *Managing Load*, *Addressing Cascading
  Failures*, and the admission-control patterns the Green/Yellow/Red
  playbook is a specialisation of —
  <https://sre.google/workbook/>.
- Provider documentation (OpenAI rate-limit docs, Anthropic rate-limit
  docs, Google Vertex AI quotas, Azure OpenAI tier docs, AWS Bedrock
  quota docs) — correct source for the dated `rpm_cap`, `tpm_cap`,
  `burst_policy`, and `retry-after` behaviour. These are intentionally
  not reproduced here because they drift; cite the version in the
  plan.
- Local exercise context: this plan sits downstream of the workload
  classes defined in Exercise 02 and provides the rate-limit signal the
  HPA in Exercise 05 consumes as an external metric.
