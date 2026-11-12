# Exercise 01 — TTFT / TPOT / ITL SLO Authoring per Workload Class — Reference Solution

## 1. Solution overview

Reference latency SLO one-pager for an **illustrative internal LLM
gateway** carried as `acme-llm-gw` across exercises 01–07. Three
distinct product surfaces are chosen so the per-surface SLI matrix
contains real contrast, not three variants of chat.

- `chat-stream` — consumer-facing chat product, streaming SSE responses.
  Workload class: **chat**.
- `code-assist` — IDE-integrated coding assistant that returns JSON
  function-call payloads (tool-calling). Workload class:
  **tool-calling**.
- `batch-summarize` — overnight pipeline that summarizes support
  tickets in bulk; non-streaming, async. Workload class: **batch**.

All numbers are illustrative and traceable to a baseline report
(dependency flagged in §6 of this document and fully derived in
chapter 06 of the module). The point of the exercise is the *shape
of the SLI surface* per workload class, not production data.

A reviewer should read sections 2 and 3 first, use section 4 to
confirm the burn thresholds are recomputed against each SLO target,
and cross-check section 5 to confirm that two live trade-offs are
defended on the four points.

> **Source policy.** The percentile targets, the valid-event filter
> set, the burn-rate multipliers, and the attribute surface draw from
> the OpenTelemetry Generative AI Semantic Conventions
> (<https://opentelemetry.io/docs/specs/semconv/gen-ai/>) and the
> SRE Workbook "Alerting on SLOs" chapter
> (<https://sre.google/workbook/alerting-on-slos/>). No facts about
> any real production service are claimed.

---

## 2. Worked answer / Implementation

### 2.1 Section 1 — The workloads

| Surface | Workload class | User-experience contract |
|---|---|---|
| `chat-stream` | chat | Users expect responses to start within one second and complete within ten seconds on 95% of turns, with no visible stall gap longer than two seconds. |
| `code-assist` | tool-calling | Users expect a syntactically-valid completion within two seconds of invocation on 95% of invocations; a malformed structured response is treated as a failure, not a slow success. |
| `batch-summarize` | batch | The nightly job completes before 06:00 UTC on 99% of weekdays; per-ticket latency is not user-visible. |

The three surfaces are chosen because their SLI matrix is
*materially different*. `batch-summarize` legitimately has no TTFT
contract. `code-assist` is dominated by structured-output
conformance, which is a quality SLI (exercise 02) rather than a
latency one. `chat-stream` is the surface where all five latency
SLIs are live.

### 2.2 Section 2 — The per-surface SLI matrix

| Surface | TTFT | TPOT | ITL | E2E | Goodput |
|---|---|---|---|---|---|
| `chat-stream` | p95 ≤ 1.2 s on cache-miss; p95 ≤ 400 ms on cache-hit | p95 ≤ 80 ms / token | p99 ≤ 2000 ms inter-token gap | p95 ≤ 10 s on turns ≤ 800 output tokens | p90 ≥ 12 tokens/sec sustained |
| `code-assist` | p95 ≤ 700 ms | N/A — response is a single structured payload, not a token stream after decode; the latency contract is E2E | N/A — same reason | p95 ≤ 2.0 s (schema-valid responses only) | N/A — ungrouped single-shot completion |
| `batch-summarize` | N/A — non-streaming async API | N/A — non-streaming | N/A — non-streaming | p99 ≤ 180 s per ticket; p99 ≤ 06:00 UTC for job completion | N/A — job-level throughput is measured separately in capacity planning (mod-106) |

**Percentile choice justifications.**

- `chat-stream` TTFT at p95, not p99: chat has frequent long-tail
  inputs (long-context sessions) where a p99 contract is a cost
  commitment to the long-context minority rather than a UX
  improvement for the median user. Cache-hit vs. cache-miss split
  because the cache-miss baseline is ~3× the cache-hit baseline
  (illustrative) and bundling them hides a cost regression.
- `chat-stream` ITL at p99: a single multi-second stall ruins a
  stream even if the average is fine. ITL is a rare-event SLI; p99
  is the right percentile.
- `code-assist` E2E at p95: downstream IDE tooling times out at
  3.5 s. The SLI target sits a margin below the timeout so a p95
  breach signals risk, not failure.
- `batch-summarize` E2E at p99: the SLA is binary — the job
  either made the 06:00 deadline or it didn't — and the business
  cost of missing the window is step-function, so the tail
  percentile is the right choice.

### 2.3 Section 3 — Valid-event filters

**`chat-stream` TTFT (cache-miss).** Server-authoritative, measured
at the gateway from the moment the request body is fully buffered
to the moment the first non-empty SSE `data:` chunk is written to
the socket. Prefix-cache hits are **excluded** from the cache-miss
denominator (they are in the companion cache-hit SLI). Chunked
prefill: "first token" is the first token written to the stream,
regardless of whether prefill chunking is in flight internally;
the engine's internal chunk boundaries are not user-visible.
Speculative decoding is server-side TPOT and does not change the
TTFT definition. Abandoned streams: if the client disconnects
before the first token is emitted, the request is **excluded** from
the TTFT denominator (there is no user-perceived first-token
experience). Retries: if the client retries on an admission-reject
(`429` / `503` before any stream opened), the retry's TTFT is the
one that counts toward the user experience; the admission-rejected
attempt is counted in the protocol-availability SLI (exercise 03),
not here.

**`chat-stream` TPOT.** Server-side; measured as the mean
inter-token interval per stream excluding the first token and the
final-token boundary. Speculative decoding complicates this —
rejected speculative tokens inflate server-side TPOT if counted,
so **only accepted tokens are in the denominator**. Chunked prefill
is irrelevant to TPOT. Abandoned streams: client-cancelled streams
contribute their observed tokens to the TPOT sample (the tokens
*were* generated at some rate before the cancel).

**`chat-stream` ITL.** Server-side; per-stream, the longest
observed inter-token gap. Prefix-cache has no effect on ITL since
cache hits are decode-side. Speculative decoding: a batch of
accepted speculative tokens emitted together produces a *near-zero*
inter-gap between the batch's tokens and a *larger* gap before it;
only the larger gap is in the sample. Abandoned streams: ITL is
undefined for streams that emit zero or one token; those are
**excluded**.

**`chat-stream` E2E.** Client-authoritative (the browser sees the
real end-of-stream), server-authoritative as a backup. The delta
between the two is tracked as a low-priority SLI (stretch goal). A
server-truncated stream is a failed E2E event, not a partial-success
event (that distinction is in exercise 03 — this SLI counts only
*complete* streams in the numerator). Retries that produced the
delivered stream are **included** in the E2E measurement from the
first client-visible attempt.

**`chat-stream` goodput.** Server-side token-rate over the sustained
middle of the stream (excluding TTFT ramp and tail flush). Prefix
cache is irrelevant because goodput is a decode-phase metric.
Speculative decoding boosts goodput (that is one of the main
reasons to enable it); only accepted tokens are in the numerator.

**`code-assist` TTFT.** Server-authoritative. "First token" here
means the first byte of the structured response, since the stream
is short and the UX is single-shot. Cache slicing: cache-hit vs.
cache-miss is tracked but the SLI is a single number; the split is
in the Grafana panel for diagnostic, not in the SLO contract.
Chunked prefill irrelevant (short prefixes). Abandoned requests
excluded.

**`code-assist` E2E.** Server-authoritative on a *schema-valid*
response. A request that returns a syntactically-invalid JSON
payload is **not a slow success** — it is counted as a failure in
the structured-output-conformance SLI (exercise 02), and the
corresponding E2E measurement is **excluded**. This decision keeps
the latency and quality SLIs cleanly separated.

**`batch-summarize` E2E (per ticket).** Server-side, start-to-end of
the summarization task. Retries within the batch pipeline (idempotent
replays on transient failure) are collapsed into a single E2E event;
the retried latency rolls into the primary.

**`batch-summarize` E2E (job completion).** Wall-clock between
scheduled start and declared-complete. Partial completions are a
failure for this SLI even if per-ticket latency is fine.

### 2.4 Section 4 — Burn-rate alert thresholds

Multi-window / multi-burn-rate thresholds per the SRE Workbook
chapter cited above. Multipliers: fast burn = 14.4× (2% of 28-day
budget in 1 h), medium burn = 6× (5% in 6 h), slow burn = 1×
(10% in 3 d). Thresholds are recomputed per SLO target because the
absolute burn fraction is `multiplier × (1 − SLO)`.

| SLI | Target | Fast burn (page) | Medium burn (page) | Slow burn (ticket) |
|---|---|---|---|---|
| `chat-stream` TTFT cache-miss | p95 ≤ 1.2 s | > 72% of valid events over the two-window `(1h, 5m)` guard — at the 95% SLO this is `14.4 × 5% = 72%` | > 30% over `(6h, 30m)` — `6 × 5% = 30%` | > 5% over `(3d, 6h)` — `1 × 5% = 5%` |
| `chat-stream` TTFT cache-hit | p95 ≤ 400 ms | > 72% over `(1h, 5m)` | > 30% over `(6h, 30m)` | > 5% over `(3d, 6h)` |
| `chat-stream` TPOT | p95 ≤ 80 ms / tok | > 72% over `(1h, 5m)` | > 30% over `(6h, 30m)` | > 5% over `(3d, 6h)` |
| `chat-stream` ITL | p99 ≤ 2 s | > 14.4% over `(1h, 5m)` — at 99% SLO this is `14.4 × 1% = 14.4%` | > 6% over `(6h, 30m)` | > 1% over `(3d, 6h)` |
| `chat-stream` E2E | p95 ≤ 10 s | > 72% over `(1h, 5m)` | > 30% over `(6h, 30m)` | > 5% over `(3d, 6h)` |
| `chat-stream` goodput | p90 ≥ 12 tok/s | > 144% over `(1h, 5m)` — at 90% SLO this is `14.4 × 10% = 144%` — i.e. always triggers; cap at "≥ 50% of events violating"; medium = 60%, slow = 10% | 60% over `(6h, 30m)` | 10% over `(3d, 6h)` |
| `code-assist` TTFT | p95 ≤ 700 ms | > 72% over `(1h, 5m)` | > 30% over `(6h, 30m)` | > 5% over `(3d, 6h)` |
| `code-assist` E2E | p95 ≤ 2 s | > 72% over `(1h, 5m)` | > 30% over `(6h, 30m)` | > 5% over `(3d, 6h)` |
| `batch-summarize` E2E (per-ticket) | p99 ≤ 180 s | > 14.4% over `(1h, 5m)` — `14.4 × 1%` | > 6% over `(6h, 30m)` | > 1% over `(3d, 6h)` |
| `batch-summarize` E2E (job) | p99 ≤ 06:00 UTC on weekdays | n/a — binary per-day signal; use a single-day miss = page, two consecutive misses = incident | — | — |

Pageable alerts: fast and medium burn for `chat-stream` TTFT
cache-miss, TPOT, ITL, E2E, and `code-assist` E2E. All others
ticket-only. Rationale in exercise 05.

The `chat-stream` goodput row illustrates a gotcha the SRE
Workbook mentions in passing: when the SLO is 90%, the mechanical
`14.4 × (1 − SLO)` fast-burn multiplier produces a threshold
greater than 100%. The multi-window guard still works (the
threshold on the *long* window stays at 14.4× budget, but is
capped against the window size), and the alert still fires on
genuine fast burns; this is a known artifact of the multi-window
pattern against high error-rate SLOs.

### 2.5 Section 5 — Threshold defence (two SLIs)

**SLI A: `chat-stream` TTFT cache-miss @ p95 ≤ 1.2 s.**

1. **What the user experiences.** The first word of the reply
   arrives inside the UX "did my request go through?" window.
   Beyond ~1.5 s, user-abandoned turns spike (illustrative baseline
   from the chat product's A/B of a prior latency regression).
2. **What the business loses when breached.** Abandoned-turn rate
   is the operational proxy for user frustration; the chat
   product's retention signal is correlated with it. A quarter-
   long breach is projected to degrade monthly-active-session by
   a measurable fraction (quantified in exercise 04's unit
   economics; stated here directionally, not numerically, to avoid
   fabricating revenue numbers).
3. **What it costs to deliver.** Moving the baseline from ~1.6 s
   to ~1.1 s on cache-miss cost a prior engineering quarter in
   prefix-cache sizing, admission-queue tuning, and KV-cache
   eviction policy work. Holding the baseline costs a continuous
   capacity-reserve margin — `mod-106` names the specific headroom
   factor.
4. **Fallback if missed.** If the SLO cannot be held under current
   capacity, the operable fallback is admission-queue trimming
   (drop long-prompt traffic to a slower queue) before any
   user-visible quality degradation. This fallback is the subject
   of the graceful-degradation SLI in exercise 03.

**SLI B: `code-assist` E2E @ p95 ≤ 2.0 s (schema-valid only).**

1. **What the user experiences.** The IDE shows the completion
   before the user's next keystroke pauses; beyond ~2.5 s, the
   user has moved on and the completion is noise rather than help.
2. **What the business loses when breached.** `code-assist`
   retention is suggestion-acceptance-rate-driven; acceptance
   collapses when latency exceeds the typing-pause window.
3. **What it costs to deliver.** The schema-valid constraint
   forces a constrained-decoding mode that is ~15% slower than
   free decoding (illustrative, consistent with vendor-published
   constrained-decoding overhead). Holding the SLO at 2.0 s with
   constrained decoding is a model-choice decision — a smaller
   model or a quantized variant may be required.
4. **Fallback if missed.** A tiered model fallback (frontier → a
   distilled smaller model) is the `code-assist` graceful-
   degradation path. Quality-vs-primary delta is tracked per
   exercise 03's fallback-path catalog.

### 2.6 Section 6 — Measurement gotchas written into the SLI spec

- **Prefix cache.** `chat-stream` TTFT is split cache-hit / cache-
  miss; the aggregate is not an SLO, only the two sub-SLOs are.
  Rationale: a cache-hit-ratio regression would hide inside an
  aggregate target.
- **Chunked prefill.** "First token" is user-visible first byte on
  the stream; engine-internal chunk boundaries are not. Spec
  explicitly says the SLI is UX-anchored, not engine-anchored.
- **Speculative decoding.** TPOT numerator counts accepted tokens
  only; rejected speculative tokens are a cost signal (exercise
  04), not a latency one. Server-side TPOT is the authoritative
  number; client-side TPOT is a secondary check.
- **Abandoned streams.** Client-cancelled streams are excluded from
  TTFT and ITL denominators; they contribute observed tokens to
  TPOT; they are a *failed* E2E event, not a partial-success one.
  The partial-success SLI (exercise 03) is where partial-success
  semantics live.
- **Retries.** Admission-reject retries contribute to the protocol-
  availability SLI (exercise 03) once; the final attempt that
  produced a stream is the one in the latency SLI. Mid-stream
  retries do not exist in the architecture — if the server drops
  mid-stream, the client sees a failure, not a transparent retry.
- **Instrumentation gap flagged — client-closed vs. server-closed
  distinction.** The gateway currently emits `stream_terminated`
  without tagging which side closed. This is an **open
  instrumentation gap**; until closed, the partial-success SLI
  in exercise 03 cannot be fully trusted. Tracked as an exercise-
  02 / 03 blocker (filed as instrumentation ticket in exercise
  02 appendix).
- **Workload-specific gotcha — `code-assist` schema-valid filter.**
  The latency SLI *excludes* schema-invalid responses. The bridge
  to exercise 02's structured-output-conformance SLI is explicit:
  one invalid response increments *that* SLI's numerator and is
  *not* counted in the latency SLI.

---

## 3. Validation steps

1. Confirm every cell in the §2.2 matrix is either a target or an
   explicit "N/A — reason." No empty cells.
2. For every non-N/A SLI in §2.2, confirm §2.3 addresses its valid-
   event filter (measurement side, cache slicing, chunked prefill,
   speculative decoding, abandoned streams, retries — those that
   apply).
3. Confirm the §2.4 thresholds were recomputed against each SLO
   target. Spot-check: the ITL row at 99% must be ~14.4%, not the
   72% used for the 95% SLOs. The goodput row at 90% hits the
   `14.4 × 10% > 100%` ceiling and requires a capped threshold.
4. Confirm §2.5 defends **two** SLIs with trade-offs *live* in the
   workload, each on all four points (user experience, business
   loss, cost, fallback).
5. Confirm §2.6 addresses at least five of the chapter 01 §5
   gotchas plus one workload-specific gotcha; check that the
   `stream_terminated` gap is flagged as an instrumentation
   dependency for exercises 02/03.
6. Word-count check against the ≤ 2000 word acceptance criterion.
7. Reviewer dry-run: hand the artifact to someone unfamiliar with
   `acme-llm-gw` and ask them to answer "what are you promising,
   how are you measuring it, what did you trade off?" in ten
   minutes.

---

## 4. Rubric or review checklist

| # | Criterion | Pass signal | Fail signal |
|---|---|---|---|
| R1 | Named workload with ≥ 3 surfaces across ≥ 2 workload classes | Three surfaces, three classes, user-experience contracts in §2.1 | "One LLM gateway" without surface breakout |
| R2 | SLI matrix populated end-to-end | All cells are a target or explicit N/A with a reason | Blank cells; N/As without a reason |
| R3 | Percentile targets justified per workload class | Each percentile has one-line defence against the workload class | "p99 because p99 is standard" |
| R4 | Valid-event filters for every non-N/A SLI | §2.3 addresses the six listed filter items where they apply | Filter section missing for one or more SLIs |
| R5 | Burn thresholds recomputed per SLI target | Multipliers applied against each SLO; 99% and 90% SLOs have different absolute thresholds | A single generic threshold table reused across all SLOs |
| R6 | Two threshold defences on all four points | §2.5 names user experience, business loss, cost, fallback for two SLIs | One-point defences; defences that pick trivial SLIs |
| R7 | ≥ 5 gotchas addressed with SLI-spec responses | §2.6 lists five chapter-01 gotchas plus one workload-specific | Gotchas listed without SLI-spec treatment |
| R8 | Reviewer can answer "what / how / trade-off" in 10 min | Dry-run with unfamiliar reviewer passes | Reviewer has to ask clarifying questions |

Partial credit: hitting R1–R5 is a passing reference artifact; R6–R8
separate competent from excellent.

---

## 5. Common mistakes

- **Treating the matrix as a density goal.** Learners try to fill
  every cell. The exercise is the opposite — the point of the
  matrix is to *expose* which SLIs are N/A per workload class.
  Dense matrices are a smell.
- **Copy-pasting one burn-rate threshold across all SLOs.** The
  absolute threshold scales with `1 − SLO`; a single number across
  a 99% SLO and a 90% SLO is wrong by an order of magnitude.
- **Bundling cache-hit and cache-miss into one TTFT SLO.**
  Cache-miss baselines are 3–5× higher; an aggregate target will
  either over-promise on cache-hit or under-promise on cache-miss
  and will hide a cache-hit-ratio regression.
- **Picking p99 everywhere.** p99 on TTFT for a chat workload
  commits the SLO to the long-context minority. p95 is usually
  the right default; p99 is right for *rare-event* SLIs like ITL.
- **Treating abandoned streams as failures by default.** A client
  that cancelled did not experience a failure in the TTFT sense;
  the partial-success SLI (exercise 03) is where that lives.
- **Hiding the baseline dependency.** Writing specific numbers
  without flagging the baseline report is a cite-your-source
  failure; the reviewer should see that the numbers trace to a
  measurement, not to intuition.
- **Forgetting the `code-assist` E2E filter is schema-valid.**
  A malformed JSON response is a quality failure, not a slow
  success. Letting it slip into the latency denominator inflates
  the latency SLI and masks the structured-output-conformance
  signal.

---

## 6. References

- OpenTelemetry Generative AI Semantic Conventions —
  <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Source for
  the attribute surface (`gen_ai.usage.*`, `gen_ai.response.*`) that
  the SLI definitions reference.
- Google SRE Workbook, *Alerting on SLOs* —
  <https://sre.google/workbook/alerting-on-slos/>. Source for the
  multi-window / multi-burn-rate pattern used in §2.4.
- Google SRE Book, *Service Level Objectives* —
  <https://sre.google/sre-book/service-level-objectives/>. Source
  for the valid-event-filter discipline used throughout.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework>. Source
  for the measurement / governance framing referenced in the
  cost-and-quality handoff paragraphs of exercises 02 and 04.
- Local context: `lessons/mod-103-slo-engineering-for-model-backed-services/exercises/exercise-01-ttft-tpot-itl-slo-authoring-per-workload-class.md`
  (the learning exercise this solution answers).
