# Exercise 03 — Availability SLO for Streaming Responses — Reference Solution

## 1. Solution overview

Reference streaming-availability artifact for the **`acme-llm-gw`**
workload introduced in exercise 01. The protocol-level 2xx SLI is
retained for continuity; three streaming-specific SLIs are added —
partial-success, graceful-degradation, streaming-stall — each with
a specific valid-event filter, a defended target, and a burn-rate
alert config. A five-scenario failure walkthrough verifies the
SLI surface diagnoses each failure by a *distinct* burn pattern.

The load-bearing instrumentation gap from exercise 01 — that the
gateway does not currently tag `stream_terminated` with client-vs-
server closer — is called out again here as a blocker for the
partial-success SLI being fully operable (section 4).

---

## 2. Worked answer / Implementation

### 2.1 Section 1 — The streaming-availability block

| SLI | Definition | Valid-event filter | Target | 28-day shortfall | Burn thresholds (fast/med/slow) |
|---|---|---|---|---|---|
| Protocol 2xx | Fraction of requests the gateway returned a `2xx` HTTP status (or `3xx` redirect, rare) | Includes admission-reject retries as separate request-events (retries do *not* collapse at this layer); excludes health-check traffic tagged `probe=true` | ≥ 99.9% | 0.1% | 1.44% / 0.6% / 0.1% |
| Partial-success | Fraction of streams the server emitted the declared-complete `[DONE]` sentinel for, among streams that the server *opened* | Numerator: server-side observed `stream.terminated.by=server_done`. Denominator: streams that passed the TTFT boundary. Client-cancelled streams **excluded** — this SLI measures server-side completion, not user patience. Instrumentation gap: `stream.terminated.by` tag required; until it ships, SLI is in warn-only mode | ≥ 99% | 1% | 14.4% / 6% / 1% |
| Graceful-degradation | Fraction of responses *tagged with a fallback path* that met a per-path quality / latency gate | Numerator: responses where `fallback.path` attribute is non-empty AND the per-path sub-SLI passed. Denominator: responses with non-empty `fallback.path`. Primary-path responses are **not** in the denominator | ≥ 99.5% aggregate | 0.5% | 7.2% / 3% / 0.5% |
| Streaming-stall | Fraction of active streams where the longest inter-token gap stayed under the 2-second per-stream threshold | Numerator: streams where `max_itl_gap_ms < 2000`. Denominator: streams emitting ≥ 2 tokens. Aliased to the `chat-stream` ITL SLI from exercise 01; carried here for diagnostic visibility only — the single source of truth for the shortfall budget is the ITL SLI | ≥ 99% (aliased) | 1% (aliased, not double-counted) | 14.4% / 6% / 1% |

### 2.2 Section 2 — The fallback-path catalog

| Fallback name | Trigger condition | Fail-open vs. fail-closed | Quality-vs-primary delta (qualitative) | Per-path sub-SLI target |
|---|---|---|---|---|
| `provider-secondary` (LiteLLM-routed secondary vendor) | Primary provider emits ≥ 5% 5xx over a 30 s window, or latency SLO ITL burning on primary | Fail-open — stream proceeds on secondary | Modest — secondary is a peer-tier model; hallucination signal may shift by ~0.5–1 point | ≥ 99% of routed streams complete; sub-SLI tracked separately |
| `model-smaller` (distilled smaller model) | Admission-queue depth > 85% for > 60 s | Fail-open — stream proceeds on smaller model | Noticeable — smaller model may regress on long-reasoning turns; retention proxy is tracked | ≥ 98% of routed streams complete |
| `retrieval-unavailable` (RAG pipeline failure fallback) | Retrieval index returns 5xx on primary call and secondary replica | Fail-open with a *degraded-notice* banner in the UI; the model is prompted to answer without retrieved context and to disclose the lack of grounding | High — hallucination SLI is expected to burn on this path; the fallback is announced to the user | ≥ 95% of routed streams complete (looser target because the UX is explicitly degraded) |
| `guardrail-fail-open` (safety sidecar timeout) | Safety sidecar misses its 200 ms budget | **Fail-closed** — request is rejected with a user-facing "please try again" message | N/A — no response served | ≥ 99.9% of safety-sidecar calls meet the budget (the SLI is really a dependency-health SLI on the sidecar) |
| `cached-response` (last-good cached response on prefix match) | Backend returns 5xx *and* a prefix cache hit is available for the request | Fail-open — cached response served with a `fallback.reason=cached` tag | Low — the cached response is known good; the risk is staleness of the content itself | ≥ 99% of cache-served streams complete |

The guardrail-fail-closed row is deliberately in the catalog even
though it is the opposite of "graceful degradation" — exercise 05
section 6 requires the policy name the fail-closed paths explicitly.
Safety fallbacks also route a notification to `ai-infra-security`
(not just SRE on-call).

### 2.3 Section 3 — Streaming-stall threshold calibration

The 2 s inter-token-gap threshold is defended on two axes.

**UI axis.**  The `chat-stream` surface renders tokens in-place as
they arrive. Empirical UX research (illustrative, consistent with
well-established reading-pace / perception-pause thresholds) shows
that a stall gap beyond ~1.5–2 s reads to the user as "the model
is stuck." A 1 s threshold would be more sensitive but would page
on routine scheduler jitter; a 5 s threshold would miss stall
events that abandon user turns. 2 s is the operating point. Voice
UIs — out of scope for `chat-stream` — would need a ~200 ms
threshold; batch pipelines would tolerate ~10 s.

**Engine-event axis.**  Gaps of ~2 s are *rare* under a healthy
decode and are produced by: (a) scheduler preemption when a noisy
co-tenant lands on the same shard, (b) KV-cache eviction forcing a
prefix re-prefill under memory pressure, (c) upstream network
microbursts at the provider edge if the gateway is proxying a
remote model, (d) GC-pause-equivalent in the inference engine's
batching loop. The threshold is chosen to be *outside* the natural
operating variance — so a burn signals a real engine event, not
healthy scheduler noise.

**Trade-off commentary.**  Lower threshold = more pages on scheduler
interference that is self-recovering; higher threshold = missed
abandons. The chosen 2 s is also the threshold the streaming-stall
sub-SLI *aliases* from latency ITL (exercise 01) — the same number
serves both the latency surface and the availability surface, which
is intentional to avoid managing two threshold knobs.

### 2.4 Section 4 — Measurement decisions codified

1. **Server-side is authoritative; client-side is secondary.**  All
   four availability-block SLIs read from server-side spans. The
   client-side delta (server-emitted-completion-time vs. client-
   observed-completion-time) is tracked as a low-priority SLI
   `availability.client_server_delta` — a growing delta is a
   gateway or egress incident. Instrumentation lives in the browser
   SDK (`gen_ai.client.*`) and the gateway edge span
   (`gen_ai.server.*`).
2. **Client cancellation vs. server truncation.**  The gateway
   emits `stream.terminated.by` with values `server_done`,
   `server_error`, `client_cancelled`. The partial-success SLI's
   valid-event filter *requires* this tag to be reliable.
   **Instrumentation gap**: the current build conflates
   `server_error` and `client_cancelled` under a single
   `stream_terminated` event without the `by` tag. Until that
   lands, the partial-success SLI is warn-only and does *not*
   drive an error budget. Tracked as a release blocker.
3. **Fallback-path tag on every response.**  Every response
   (streaming or not) carries a `fallback.path` attribute. Empty
   or `primary` means the request served the primary path; a
   non-empty value names the specific fallback (see §2.2). The
   graceful-degradation SLI's denominator *is* the set of responses
   with a non-empty `fallback.path` — if the tag is not reliably
   emitted, the SLI silently under-counts fallback events.
   Attribute emitted at the gateway span root so it is visible to
   both metrics and traces.

### 2.5 Section 5 — Failure walkthrough

Each failure case walks all four availability-block SLIs *and*
latency SLIs where relevant. The point is that each failure burns
a *distinct* combination.

| Failure | Protocol 2xx | Partial-success | Graceful-degradation | Streaming-stall | First burn | Runbook |
|---|---|---|---|---|---|---|
| A — 5xx storm (30% 5xx for 15 min) | **Burns fast** (0.1% budget; 30% error rate is ~300× multiplier) | Depressed slightly (streams that failed before TTFT excluded) | Does not burn (no fallback tagged) | Does not burn | Protocol 2xx | `runbooks/protocol-availability-burn.md` → `mod-102` chapter 03 escalation ladder |
| B — Mid-stream server crash (preemption, 400 of 1000 tokens) | Clean (response had 2xx headers) | **Burns** (streams opened but did not reach `server_done`) | Does not burn (no fallback) | May burn depending on pre-crash cadence | Partial-success | `runbooks/partial-success-burn.md` |
| C — Primary provider 5xx, LiteLLM fallback (20% for 10 min) | Clean (gateway absorbed the failures) | Clean (streams completed on secondary) | **Burns** aggregate signal (20% of traffic is now `fallback.path=provider-secondary`); per-path sub-SLI may stay green | Clean | Graceful-degradation (aggregate signal rate, not sub-SLI target) | `runbooks/fallback-rate-burn.md` |
| D — Scheduler-induced stall (4 s inter-token gap) | Clean | Clean (streams did complete) | Clean | **Burns** (ITL p99 > 2 s) | Streaming-stall | `runbooks/stall-burn.md` → capacity escalation |
| E — Retrieval index outage (`retrieval-unavailable` fallback engaged for 20 min) | Clean | Clean | **Burns** aggregate signal AND the `retrieval-unavailable` sub-SLI (hallucination SLI also burns — correlated-burn) | Clean | Graceful-degradation (sub-SLI) correlated with hallucination | `runbooks/retrieval-outage.md` + eval-peer notification |

Each row burns a distinct SLI combination:

- A: protocol only.
- B: partial-success only.
- C: graceful-degradation aggregate only.
- D: streaming-stall only (correlated with ITL latency).
- E: graceful-degradation **sub-SLI** correlated with a *quality*
  SLI (hallucination) — the only case that produces a cross-
  dimension correlated-burn signal. The on-call primary routes
  this through the IC path from exercise 05 section 5.

The walkthrough acceptance test: a hypothetical sixth failure —
"sidecar guardrail timing out on 2% of safety-checked prompts for
30 minutes" — should burn the `guardrail-fail-open` sub-SLI (zero
responses served while the fallback is fail-closed) and *not* the
other SLIs. A new on-call with the SLI surface open should be
able to resolve the diagnosis using this pattern without
reading a runbook first.

---

## 3. Validation steps

1. Confirm §2.1 has four rows (protocol + three streaming) with
   every cell populated and valid-event filters explicit.
2. Confirm §2.2 has at least three fallback paths (we have five),
   each with fail-open/fail-closed decision and per-path target.
3. Confirm §2.3 defends the 2 s streaming-stall threshold on both
   UI and engine-event axes.
4. Confirm §2.4 codifies the three measurement decisions and
   flags the `stream.terminated.by` instrumentation gap.
5. Confirm §2.5 walks five failures and each burns a *distinct*
   SLI combination.
6. Confirm the aliasing note on streaming-stall (ITL in exercise
   01) prevents double-counting in the composed stack from
   exercise 02.
7. Word-count check against the ≤ 1500 word acceptance criterion.
8. Reviewer dry-run: a sixth hypothetical failure should be
   diagnosable from the SLI surface pattern without reading a
   runbook.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Four-row availability block populated | Protocol + 3 streaming SLIs with filters, targets, thresholds | Missing an SLI; thresholds absent |
| R2 | ≥ 3 fallback paths cataloged with fail-open/closed explicit | Five paths listed; fail-closed path visible | Only reliability fallbacks; no safety fallback |
| R3 | Stall threshold defended on UI and engine axes | Both axes present with concrete content | Hand-wave; "1 second is standard" |
| R4 | Three measurement decisions codified | Server/client, cancellation vs. truncation (gap flagged), fallback tag | Any decision missing or gap not flagged |
| R5 | Five failures walked | Each burns a distinct SLI combination | Two failures burn the same SLIs (duplicative) |
| R6 | Streaming-stall aliased, not double-counted | Alias note visible; shortfall not duplicated in composite | SLI duplicated in exercise-02 stack |
| R7 | Reviewer diagnoses sixth hypothetical without runbook | SLI surface pattern is self-explanatory | Reviewer asks for the runbook |

---

## 5. Common mistakes

- **Trusting the protocol 2xx SLI alone.** Streaming failures
  frequently return 2xx headers before they go sideways. Teams
  that skip the partial-success SLI miss mid-stream crashes.
- **Collapsing graceful-degradation into a single SLO.** The
  aggregate rate is a *symptom*; the per-path sub-SLIs are the
  diagnostic. A review that sees only the aggregate cannot
  distinguish provider-fallback from retrieval-unavailable.
- **Omitting fail-closed paths from the fallback catalog.** The
  guardrail-fail-closed path is *also* a fallback decision; it
  belongs in the catalog so the policy names its ownership.
- **Picking a generic 1 s stall threshold.** The threshold is UI-
  anchored; a chat UI's threshold is not a voice UI's threshold.
  Defend it in UX terms.
- **Not tagging `stream.terminated.by`.** The partial-success SLI
  depends on distinguishing client cancellation from server
  truncation. An SLI built on a conflated signal rewards the
  wrong behavior.
- **Double-counting streaming-stall.** If streaming-stall is a
  separate SLI with its own shortfall, the composite stack
  double-counts the ITL signal. Alias it instead.
- **Assuming the aggregate graceful-degradation SLI is the
  user contract.** A healthy aggregate with a hot sub-SLI is a
  silent incident; the review must open the sub-SLIs.

---

## 6. References

- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source
  for `gen_ai.client.*`, `gen_ai.server.*`, and `fallback.*`
  attribute surface conventions.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for
  the burn-rate thresholds and multi-window guard used in §2.1.
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the valid-event-filter discipline applied throughout.
- OWASP Machine Learning Security Top 10 —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Source for the safety-fallback framing in §2.2 and the
  `guardrail-fail-closed` row.
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-03-availability-slo-for-streaming-responses.md`
  (learning exercise this solution answers) plus exercise-01 and
  exercise-02 SOLUTION.md (surfaces this document extends).
