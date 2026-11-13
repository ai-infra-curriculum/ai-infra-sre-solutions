# Exercise 02 — Provider-Fallback Contract Authoring (LiteLLM-Shape) (Solution)

## 1. Solution overview

A provider-fallback contract is the document — and the runtime config that
mirrors it — that answers: *when the primary model provider misbehaves, what
exactly does the gateway do, in what order, under what budget, and with what
user-visible contract?*

"LiteLLM-shape" in the title refers to the pattern popularized by the
open-source LiteLLM router: a single OpenAI-shaped request enters a
gateway, and the router chooses from a list of model deployments using
fallback and retry policies. We use this as the implementation example
because it is documented in the open and because it maps cleanly to the
exercise. Any gateway that supports the same primitives (an OpenAI-
compatible router in front of multiple providers with per-group fallback,
per-call timeout, per-call retry, and context-window-aware fallback) is
equivalent; the contract is portable.

The contract has two halves, and both must be authored:

1. **The human-readable contract.** What *behavior* a caller can expect
   when the primary provider is degraded. This is the artifact reviewers,
   product managers, and support engineers read. It is written in prose
   with explicit invariants, non-goals, and failure-visible contracts.
2. **The machine-readable config.** The gateway configuration that
   implements the contract. It is the artifact CI validates and the
   gateway loads at boot.

Both halves ship in the repo. If only one exists, the fallback behavior
drifts the moment someone edits the config without touching the prose,
or vice versa.

Scope: this exercise assumes one *surface* (e.g., the product chat
request path) and one *primary model-provider dependency* from the
surface map in exercise-01. Repeat the pattern per surface; do not write
one mega-contract across unrelated surfaces.

## 2. Worked implementation

### 2.1 The human-readable contract (template, filled)

```markdown
# Provider-Fallback Contract: product-chat surface

Owner: inference-platform
Last reviewed: <date>
Standards references: NIST AI RMF (MAP 4.1, MANAGE 2.1); OWASP ML Top 10
(ML06 supply-chain); MITRE ATLAS (AML.T0010 ML supply-chain compromise).

## Scope

This contract governs model-provider fallback for the `product-chat`
surface and only that surface. Non-chat surfaces (embeddings pipeline,
batch eval harness) are out of scope and have their own contracts.

## Providers in the group (ordered)

1. `primary`   — <provider-A>, model family <X>, hosted in region <R1>.
2. `fallback-a` — <provider-B>, model family <Y>, hosted in region <R2>,
                 distinct cloud and distinct inference substrate.
3. `fallback-b` — self-hosted <open-weights-model> in <R3>.

Provider identity, auth, and per-provider quotas live in the vendor
surface map (see ex-01); this contract references their IDs only.

## Invariants

- I1. **Single primary.** At most one provider is "primary" at a time.
- I2. **Equivalence class.** `fallback-a` and `fallback-b` are approved
     substitutes under the output-equivalence tests linked in §Validation.
     If a request requires a capability (tool calling / vision / structured
     output / >200k ctx) that any fallback lacks, the request is marked
     `no-fallback-eligible` and must not be silently rerouted.
- I3. **No partial responses.** We never concatenate the start of a
     streaming response from one provider with the continuation from
     another. Streaming failures either retry *the whole message* on a
     fallback or surface the error.
- I4. **Idempotency.** Every chat completion carries a client-supplied
     idempotency key. Retries across providers reuse that key so
     downstream logging and billing reconcile.
- I5. **User-visible contract preserved.** The gateway returns an
     OpenAI-shaped response regardless of which upstream served it.
     The response is tagged with `x-served-by: <provider-id>` so
     operations can distinguish; callers must not depend on provider
     identity for correctness.
- I6. **Budgeted.** Total time budget per top-level request is fixed
     (see §Budgets). Fallback attempts consume the same budget — we do
     not reset wall-clock on hop.

## Trigger conditions

A fallback is invoked when any of:

- Transport: TCP connect failure, TLS failure, DNS failure to upstream.
- Server errors: HTTP 5xx after the retry policy on the current provider
  is exhausted.
- Rate/quota: HTTP 429 with a `Retry-After` longer than our remaining
  budget, OR repeated 429 within the budget.
- Timeout: no first byte within `first_byte_timeout_ms`, or no completion
  within `request_timeout_ms`.
- Explicit signal: circuit breaker for that provider is open.
- Provider-side decode/validation error identifiable by provider error
  code (e.g., context length exceeded on a model we have not pinned a
  larger-context fallback for → do NOT fall back, surface the error).

Fallbacks are NOT invoked when:

- 4xx auth/permission: 401, 403 → surface as an incident, do not spray
  bad creds across other providers.
- 4xx policy/safety: content policy rejections are not reliability
  failures. Do not retry on a different provider to "get around" a
  safety refusal.
- Client-side validation errors (400 Bad Request): surface.

## Budgets

| Parameter | Value | Rationale |
|---|---|---|
| Top-level request budget | 30 s | Product constraint from SLO |
| First-byte timeout, per attempt | 8 s | Allows tail-safe retry within budget |
| Request timeout, per attempt | 25 s | Leaves 5 s for one hop + overhead |
| Max retries on same provider | 1 | Idempotent-retry on 5xx only |
| Max providers tried per request | 2 | Primary + one fallback |

If budget is exhausted mid-hop, the gateway returns 504 with a
structured error body (see §Error shape) rather than starting a third
attempt.

## Circuit breaker policy

- Per-provider, per-model.
- Open after `error_ratio > 0.5` sustained over a 60-second window with
  at least 20 requests in-window.
- Half-open probe: 1 request every 10 s while open; close on 5
  consecutive successes.
- While open, the provider is skipped as if it were the end of the list.

## Error shape (user-visible)

On terminal failure after all eligible fallbacks, the gateway returns:

```json
{
  "error": {
    "type": "upstream_unavailable",
    "code": "all_providers_failed",
    "retriable": true,
    "retry_after_ms": 2000,
    "request_id": "<our-id>",
    "attempted": [
      {"provider": "primary",     "status": 503, "latency_ms": 7800},
      {"provider": "fallback-a",  "status": 504, "latency_ms": 22100}
    ]
  }
}
```

Callers log `request_id` + `attempted` for support triage. `retriable`
and `retry_after_ms` are advisory to API clients; the gateway does NOT
auto-retry across the top-level request.

## Observability contract

- Each attempt emits a span with attributes:
  `gen_ai.system=<provider>`, `gen_ai.request.model=<model>`,
  `attempt.index`, `attempt.terminal_status`, `fallback.reason`.
- Each top-level request emits one metric point to
  `chat_gateway_request_total{outcome,served_by,fallback_hops}` and
  one histogram `chat_gateway_request_ttft_ms{served_by}`.
- A fallback-rate SLI, `rate(served_by!="primary") / rate(total)`, is
  alerted at >5% sustained for 10 minutes.

## Non-goals

- This contract does NOT define cost control. Fallback is reliability-
  motivated; cost-driven routing lives in a separate policy.
- This contract does NOT redefine prompts or system instructions per
  provider. Any drift between providers is handled by the equivalence
  test suite, not by per-provider prompt forks.

## Rollout / change control

Config changes land through PR; CI must execute §Validation. A dry-run
smoke run against a sandbox project is required before production
rollout. Major changes (adding or removing a fallback, changing the
primary) require a note in the next weekly reliability review.
```

### 2.2 The machine-readable config (LiteLLM-shape)

The configuration below follows the public LiteLLM proxy config schema
(`model_list`, `router_settings`, `litellm_settings`) documented in the
BerriAI/litellm project. The field names are the real LiteLLM field
names; fill vendor-specific `litellm_params` per the LiteLLM provider
docs. Even without LiteLLM, treat this as the reference shape; most
model-gateway proxies accept conceptually identical fields.

```yaml
# gateway/config/product-chat-router.yaml
# Reference: BerriAI/litellm proxy configuration.
# This config implements the "product-chat" fallback contract above.

model_list:
  # Logical group that callers address: "chat".
  # Each entry is a deployment of that logical model name.

  - model_name: chat
    litellm_params:
      model: <provider-A>/<model-X>
      api_base: https://api.<provider-A>.example/v1
      api_key: os.environ/PROVIDER_A_API_KEY
      timeout: 25            # per-attempt request timeout (seconds)
      stream_timeout: 8      # first-byte timeout for streams (seconds)
      max_retries: 1         # per-attempt idempotent retry on 5xx
    model_info:
      id: primary
      mode: chat
      supports_tool_calling: true
      supports_vision: false
      max_input_tokens: 128000

  - model_name: chat
    litellm_params:
      model: <provider-B>/<model-Y>
      api_base: https://api.<provider-B>.example/v1
      api_key: os.environ/PROVIDER_B_API_KEY
      timeout: 25
      stream_timeout: 8
      max_retries: 1
    model_info:
      id: fallback-a
      mode: chat
      supports_tool_calling: true
      supports_vision: false
      max_input_tokens: 200000

  - model_name: chat
    litellm_params:
      model: openai/self-hosted-vllm         # OpenAI-compatible self-hosted
      api_base: https://vllm.internal/v1
      api_key: os.environ/SELFHOSTED_API_KEY
      timeout: 25
      stream_timeout: 8
      max_retries: 1
    model_info:
      id: fallback-b
      mode: chat
      supports_tool_calling: false           # capability gap — see router_settings
      supports_vision: false
      max_input_tokens: 32000

router_settings:
  # Routing strategy chooses *among healthy deployments of the same model_name*.
  # For "primary → fallback-a → fallback-b" we use explicit ordering via
  # fallbacks and a lowest-cost / lowest-latency tie-break between healthy
  # copies within a tier (there is only one here).
  routing_strategy: simple-shuffle
  num_retries: 0                              # retries across providers go via fallbacks, not num_retries

  # Explicit ordered fallbacks: when "chat@primary" fails per the trigger
  # conditions, try "chat@fallback-a", then "chat@fallback-b".
  fallbacks:
    - chat@primary:
        - chat@fallback-a
        - chat@fallback-b

  # Context-window-aware fallback: if the primary rejects due to context
  # length, prefer fallback-a (200k) before fallback-b (32k).
  context_window_fallbacks:
    - chat@primary:
        - chat@fallback-a

  # Capability-aware routing: tool-calling requests must not spill to
  # fallback-b. LiteLLM honors `supports_tool_calling` in `model_info`
  # when `enable_pre_call_checks` is set.
  enable_pre_call_checks: true

  # Cooldown: how long a deployment stays excluded after failures.
  cooldown_time: 60                           # seconds (circuit breaker equivalent)
  allowed_fails: 20                           # fails before cooldown kicks in

  # Timeouts enforced by the router in addition to per-call timeouts.
  timeout: 30                                 # top-level request budget (seconds)
  stream_timeout: 8

litellm_settings:
  # Only retry on reliability errors; never on policy/auth/4xx-validation.
  retry_policy:
    BadRequestErrorRetries: 0
    AuthenticationErrorRetries: 0
    ContentPolicyViolationErrorRetries: 0
    RateLimitErrorRetries: 1
    TimeoutErrorRetries: 1
    InternalServerErrorRetries: 1

  # Idempotency / tracing headers that must propagate.
  forward_client_headers_to_llm_api: false
  # Our gateway adds x-idempotency-key and x-request-id before the router.

  # Observability hooks wired to the OTel collector.
  success_callback: ["otel"]
  failure_callback: ["otel"]

general_settings:
  # Allow callers to opt out of fallback per request for strictness-
  # sensitive calls (eval harness, regression runs).
  # Caller sends: {"metadata": {"no_fallback": true}}
  disable_fallbacks_for_metadata_key: no_fallback
```

Notes on shape:

- `fallbacks` / `context_window_fallbacks` / `cooldown_time` /
  `allowed_fails` / `routing_strategy` / `retry_policy` are the exact
  field names from the LiteLLM proxy config; they are the primitives
  the contract above depends on. Confirm current spelling against the
  LiteLLM docs at <https://docs.litellm.ai/> when you adopt a version
  (field names and defaults have changed across minor versions).
- The caller still sends `POST /v1/chat/completions` with
  `"model": "chat"` — the logical `model_name`. The router picks the
  deployment. That is what makes the fallback transparent and
  preserves invariant I5.

### 2.3 How the config satisfies each invariant

| Invariant | Mechanism in config |
|---|---|
| I1 Single primary | Order of `model_list` + explicit `fallbacks` ordering |
| I2 Equivalence class | `enable_pre_call_checks: true` + `supports_tool_calling` on `fallback-b` blocks non-equivalent fallback |
| I3 No partial responses | LiteLLM does not stitch streams across providers; retries re-emit full response |
| I4 Idempotency | Gateway injects `x-idempotency-key` before router; `forward_client_headers_to_llm_api: false` keeps it out of upstream requests but preserved in our logs |
| I5 User-visible contract preserved | OpenAI-shape input/output, `x-served-by` header added in gateway middleware |
| I6 Budgeted | `router_settings.timeout: 30` + per-attempt `timeout: 25`; no reset on hop |

### 2.4 Non-equivalence capability gap handling

`fallback-b` does not support tool calling. Three ways the contract
can encode this, in increasing strictness:

1. **Pre-call filter (chosen above).** `enable_pre_call_checks` + model
   capability flags skip `fallback-b` for tool-calling requests. The
   contract's I2 remains intact.
2. **Explicit per-request fallback list.** Callers that know their
   request uses tool calling pass a different logical model name
   (`chat-tools`) whose `model_list` contains only tool-capable
   deployments.
3. **Hard refuse.** Callers set `metadata.no_fallback=true` for
   tool-calling workloads; the request fails rather than silently
   degrading.

Pick one and document it. The default here is (1) because it keeps the
caller contract simple without silent degradation.

## 3. Validation steps

Validation runs in CI on every PR to `gateway/config/`.

### 3.1 Static validation

1. **Schema parse.** `yamllint` the config and parse it with the
   gateway's own validator (`litellm --config gateway/config/product-chat-router.yaml --test`
   is a dry-run flag the proxy exposes; confirm against your version).
2. **Contract-to-config lint.** A small script asserts:
   - Every `model_info.id` listed in the human contract exists exactly
     once in `model_list`.
   - `fallbacks` ordering matches the contract's provider ordering.
   - `timeout`, `stream_timeout`, `cooldown_time`, `allowed_fails`
     match the contract's §Budgets and §Circuit-breaker tables.
   - Any `supports_*` capability flag that is `false` has a
     corresponding `enable_pre_call_checks: true` setting.
3. **Secret presence.** Every `os.environ/...` reference resolves in
   the environment used by CI (use a dummy value for the schema test;
   the point is that the key name exists).

### 3.2 Behavioral validation (replay / integration)

Use a mock upstream that returns scripted failures. A compact test
matrix per the Trigger-conditions section:

| Case | Primary responds | Expectation |
|---|---|---|
| T1 Healthy | 200 OK streaming | Served by `primary`, no fallback |
| T2 503 once | 503 then no retry eligible | Retry once same provider; if still 503 → `fallback-a` serves; `served_by=fallback-a` |
| T3 First-byte timeout | hangs > 8 s | `fallback-a` serves within total budget; final latency < 30 s |
| T4 401 | 401 on first call | Terminal error surfaced; no fallback attempted |
| T5 Content policy 400 | 400 with content policy code | Terminal error surfaced; no fallback attempted |
| T6 Context length | 400 context_length_exceeded | `fallback-a` serves (context_window_fallbacks); not `fallback-b` |
| T7 Tool-calling + all-primary-fail | primary 503, tool call | `fallback-a` serves; `fallback-b` skipped by pre-call check |
| T8 All-fail | primary + fallback-a both 503 | Terminal `all_providers_failed`; body includes `attempted` array |
| T9 Cooldown | Trip primary to open circuit | Subsequent requests skip primary for `cooldown_time` |
| T10 No-fallback opt-out | caller sends `metadata.no_fallback=true`, primary 503 | Terminal error, no fallback attempted |

Each case is one unit in the integration test suite. All must pass for
the config to deploy.

### 3.3 Output-equivalence suite (I2)

A small set of fixed prompts goes to every provider in the group; the
outputs are evaluated for semantic equivalence by whatever judge the
team already uses for regression evals. Failures do not block deploy
but open a ticket against the capability gap and update the contract's
equivalence class language.

### 3.4 Live drill

Once per quarter, intentionally fail the primary at the network layer
(blackhole `api.<provider-A>` in staging) during a load test. Measure
time-to-first-byte and error rate observed by callers during the
transition. These numbers feed the SLA/SLO reconciliation in
exercise-05.

## 4. Rubric

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Contract halves** | Only prose, or only config | Both exist but out of sync | Both exist; CI lint enforces sync |
| **Invariants** | Not stated | Stated but not enforced | Each invariant has a config mechanism and a test case in §3 |
| **Trigger conditions** | Vague ("on error, retry elsewhere") | Transport, 5xx, timeout covered | All of transport/5xx/timeout/429/circuit + explicit exclusions for 401/403/policy-4xx/validation-400 |
| **Budgets** | No numbers | Numbers exist | Top-level, per-attempt, first-byte, retries, max-providers all specified with rationale |
| **Capability-aware fallback** | Ignored | Discussed in prose | Enforced in config (`supports_*` + pre-call checks) and tested |
| **Idempotency** | Not addressed | Discussed | Idempotency key path documented; retries reuse it; test covers it |
| **Error shape** | Opaque 5xx returned | Error body mentions upstream failure | Structured body with `attempted[]`, `request_id`, `retriable`, `retry_after_ms` |
| **Observability** | Metrics only, or none | Spans or metrics | Spans + metrics with explicit field names; one alert on fallback rate |
| **Validation suite** | Absent | A few cases | ≥ 8 cases from §3.2 implemented and green in CI |
| **Live drill** | Not planned | Planned | Scheduled quarterly with a named owner |

Passing: average ≥ 1.5 with no 0s on **Contract halves**, **Budgets**,
**Capability-aware fallback**, and **Validation suite**.

## 5. Common mistakes

- **Routing on cost or latency first.** Using `routing_strategy:
  lowest-cost` or `usage-based-routing` without explicit `fallbacks`
  means "fallback" happens all the time, not only on failure, and
  silently redistributes traffic. Keep reliability-driven fallback
  separate from cost routing.
- **Retrying across providers on 4xx.** 401/403/400 almost always
  indicate a bug in the caller or config, not a vendor outage.
  Retrying them wastes budget and can leak prompts to other vendors.
- **Falling back on content-policy refusals.** Treating a safety
  refusal as a reliability failure and re-sending to a provider with
  weaker filters is a safety incident, not a fallback. The contract
  must exclude policy-4xx explicitly.
- **Stitching streams across providers.** Concatenating partial output
  from `primary` with a new generation from `fallback-a` produces
  tokens that look continuous but are from a different distribution.
  Users and downstream agents are confused by this; incident forensics
  becomes nearly impossible. I3 is non-negotiable.
- **Forgetting the capability gap.** Silently falling back to a model
  that does not do tool calls and returning a chat-shaped response
  with no tool call makes agentic workflows diverge without any error.
  `enable_pre_call_checks` and the `supports_*` flags prevent this.
- **Reset budget on hop.** Starting a fresh 30-second clock on each
  attempt means a tail outage can take 90+ seconds to surface. I6
  keeps the top-level budget fixed.
- **No observability tag on `served_by`.** Without it, you cannot
  measure fallback rate, correlate quality regressions, or bill
  internally. Add the header and the metric label from day one.
- **Config without prose.** The config alone cannot tell a reviewer
  which errors must NOT be retried. Prose captures the exclusions
  that the schema cannot.

## 6. References

- **LiteLLM proxy / router documentation (practitioner implementation
  example).** BerriAI/litellm — <https://github.com/BerriAI/litellm>,
  docs at <https://docs.litellm.ai/>. Field names used in §2.2
  (`model_list`, `fallbacks`, `context_window_fallbacks`,
  `cooldown_time`, `allowed_fails`, `enable_pre_call_checks`,
  `routing_strategy`, `retry_policy`) come from the LiteLLM proxy
  configuration. Verify field names and defaults against the version
  you deploy; they have changed across minor releases.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. The
  `MANAGE` function (specifically `MANAGE 2.1`) covers residual-risk
  mitigation for identified third-party risks; the fallback contract
  is the operational artifact for that mitigation.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  `ML06 AI Supply Chain Attacks` motivates distinct inference
  substrates for `fallback-a` vs. `fallback-b`.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The `AML.T0010 ML
  Supply Chain Compromise` technique is the threat model that
  independent providers (not merely different models on one provider)
  address.
- **Upstream: exercise-01 vendor surface map.** Primary and fallback
  provider rows, including `request_path.entry_points`, `commercials`,
  and `concentration_risk`, feed the contract's §Providers and
  §Budgets.
- **Downstream: exercise-04** (fallback-rate alert consumes router
  metrics), **exercise-05** (budget math uses measured fallback
  latency), **exercise-06** (scorecard scores "fallback readiness"
  by presence and freshness of this contract).
