# Exercise 01 — SLI / SLO / Error-Budget Vocabulary Drill on AI Workloads

> Reference solution. The exercise asks learners to translate the classical SRE
> SLI / SLO / error-budget vocabulary onto AI workloads — specifically an LLM
> inference service that produces streamed, non-deterministic responses. The
> deliverable is a short glossary + applied table that a new AI SRE could use on
> day one.

## 1. Solution overview

Classical SRE vocabulary (Google SRE Book, Ch. 3–4; *The Site Reliability
Workbook*, Ch. 2) was authored against mostly deterministic request/response
services where "correct" collapses to HTTP 2xx. AI workloads break that
collapse in three ways:

1. **Correctness is a distribution, not a bit.** A response can be HTTP 200
   and still hallucinate, truncate, or drift off-policy. Quality must become a
   first-class SLI alongside latency and availability.
2. **Latency is multi-phase.** For streaming LLMs, user-perceived latency is
   driven by time-to-first-token (TTFT) and inter-token latency (ITL / TPOT),
   not a single `request_duration` histogram.
3. **Capacity is bimodal.** GPU-bound inference has a steep knee: under the
   knee it looks healthy, over the knee queueing explodes. Error budgets have
   to cover saturation as a failure mode, not just explicit errors.

The worked answer below gives the vocabulary in Google's own definitions,
then re-grounds each term on an AI workload so the mapping is explicit and
testable.

## 2. Implementation — worked answer

### 2.1 Core glossary (classical definition → AI framing)

| Term | Classical definition (Google SRE Book) | AI-workload framing |
|---|---|---|
| **SLI** — Service Level Indicator | A carefully-defined quantitative measure of some aspect of the level of service that is provided (SRE Book, Ch. 4). | A measurable property of the inference request path. For LLMs the SLI set has to include a **latency SLI** (TTFT, ITL), an **availability SLI** (successful-stream rate, not just HTTP 2xx), a **quality SLI** (e.g., refusal-rate, grounded-answer rate, judge-pass rate), and a **cost SLI** ($/1M tokens, $/session) if cost is a commitment to the business. |
| **SLO** — Service Level Objective | A target value or range of values for a service level that is measured by an SLI (SRE Book, Ch. 4). | A target *over a rolling window* on an SLI. Example: "99% of chat requests have TTFT ≤ 1.0 s over 28 days." SLOs must be aspirational but achievable (Workbook, Ch. 2) — on AI workloads this usually means **per-workload-class** SLOs (chat, batch, embedding) because the engines have different operating curves. |
| **SLA** — Service Level Agreement | An explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLIs they contain (SRE Book, Ch. 4). | The contract you sign with the business or external customer. For AI workloads the SLA is usually **looser than the SLO** — e.g., SLO 99% TTFT ≤ 1.0 s; SLA 95% TTFT ≤ 2.0 s with service credits. Do not sign an SLA on a quality metric you cannot measure continuously. |
| **Error budget** | `1 − SLO`, expressed as the allowed amount of unreliability over the SLO window (SRE Book, Ch. 3; Workbook, Ch. 4). | The permitted share of failing requests over the window. On a 99.0% / 28-day SLO the budget is 1.0% of eligible requests. AI-specific: budget is consumed by **quality regressions** (bad judge-pass scores) and **latency blow-outs during saturation**, not only by error responses. |
| **Burn rate** | The rate at which the error budget is being consumed, normalized so that `1x` burn exhausts the budget exactly at the end of the window (Workbook, Ch. 5). | Same math, but on an AI service the dominant burn sources are a bad model push, a prompt-template change, a KV-cache eviction storm, or a provider degradation — all of which can push burn to 10×–50× within minutes. Multi-window / multi-burn-rate alerting (Workbook, Ch. 5) is required; single-threshold page alerts are inadequate. |
| **Toil** | Manual, repetitive, automatable, tactical, devoid of enduring value, O(service growth) work (SRE Book, Ch. 5). | For AI SRE the headline toil items are (a) restarting wedged inference servers, (b) hand-flipping traffic between regions after a provider outage, (c) re-running eval suites by hand after a weight bump, and (d) rebuilding dashboards per model variant. Automate in that order. |
| **Four Golden Signals** | Latency, Traffic, Errors, Saturation (SRE Book, Ch. 6). | Still valid, but on inference the right instantiations are: latency → TTFT + ITL; traffic → requests/s *and* tokens/s (input + output separately); errors → HTTP errors *plus* truncations, refusals-when-not-expected, judge-fail; saturation → queue depth, KV-cache utilization, GPU SM utilization, prefix-cache hit-rate. |
| **USE method** | Utilization, Saturation, Errors (Brendan Gregg, *Systems Performance*, 2nd ed., Ch. 2). | Apply at the GPU layer: Utilization = SM-active %, Saturation = queue depth / waiting batches / KV evictions, Errors = CUDA faults, OOMs, NCCL timeouts. |
| **RED method** | Rate, Errors, Duration (Tom Wilkie, 2015). | Apply at the request-path layer: Rate = requests/s per route, Errors = non-2xx + semantic failures, Duration = TTFT and end-to-end completion. |

> The framing column above is the *mapping* the exercise is really grading. If
> a learner writes "SLI = how good the service is" they have missed the point;
> SLIs are **specific measurable quantities** (SRE Book, Ch. 4).

### 2.2 AI-specific SLI catalog (the vocabulary the drill is building toward)

| SLI name | What is counted as "good" | Measurement source | Typical SLO window |
|---|---|---|---|
| Availability (stream-success) | Response stream opens AND completes without server-side termination. | Gateway + inference server logs; count on request completion, not connection open. | 28 days rolling. |
| TTFT (Time-To-First-Token) | p95 TTFT below threshold (threshold depends on workload class — chat vs. agent). | Inference engine metric (`vllm:time_to_first_token_seconds` histogram in vLLM) or client-side tracing. | 28 days rolling, per-route. |
| TPOT / ITL (Time-Per-Output-Token or Inter-Token Latency) | p95 inter-token gap below threshold. | Inference engine metric (`vllm:time_per_output_token_seconds` in vLLM). | 28 days rolling. |
| Quality — judge-pass rate | LLM-as-judge or heuristic eval marks response as acceptable. | Offline + online eval pipeline. | 7 days (quality moves fast) with 28-day secondary. |
| Quality — grounding / citation rate (RAG) | Response cites at least one retrieved source actually used. | Post-generation checker. | 28 days. |
| Cost per million tokens | $/1M output tokens stays under target. | Billing + token counters. | Calendar month. |
| Saturation safety margin | Queue depth / KV-cache utilization stay below defined knee for ≥ X% of minutes. | Engine metrics (`vllm:num_requests_waiting`, `vllm:gpu_cache_usage_perc`). | 28 days. |

Numbers above are **shapes**, not recommended thresholds — the exercise asks
learners to understand *how to pick* them, not to memorize them. Threshold
selection is covered in `mod-103`.

### 2.3 Four worked examples (drill)

For each example the learner should be able to write it down in this exact
form: **SLI + SLO + how it is measured + what burn looks like**.

**(a) Chat completion latency.**
- SLI: fraction of chat requests whose `TTFT ≤ 1.0 s`.
- SLO: 99% over 28 days.
- Measurement: `vllm:time_to_first_token_seconds_bucket` with `le="1.0"`, divided by `_count`, scoped to the chat route.
- Burn example: a KV-cache eviction storm doubles p95 TTFT for 15 minutes → short-window burn rate ~ 14× → page.

**(b) Streaming availability.**
- SLI: fraction of SSE responses that start AND complete without server-side abort.
- SLO: 99.9% over 28 days.
- Measurement: gateway counts `stream_opened` and `stream_completed_ok`; SLI is their ratio. Do **not** use HTTP status alone — a 200 followed by a mid-stream disconnect is a failure.
- Burn example: an upstream provider drops all open streams during a failover → spike in `stream_opened − stream_completed_ok` → burn rate ~ 50× for the duration.

**(c) RAG grounding quality.**
- SLI: fraction of RAG responses that cite at least one retrieved document actually surfaced in context.
- SLO: 95% over 7 days (quality SLOs run on shorter windows because the thing being measured drifts with content).
- Measurement: post-generation checker compares citations in the response to the retrieved context.
- Burn example: an index rebuild silently drops a corpus shard → grounding rate collapses by 20 pp overnight → burn rate well over 10× on the 1-hour window.

**(d) Cost SLI.**
- SLI: rolling $/1M output tokens on the premium-chat route.
- SLO: ≤ budget threshold month-over-month.
- Measurement: provider invoice line items joined to usage logs.
- Burn example: a change to the system prompt grows average prompt length by 30% → unit cost moves above budget two weeks into the month → policy must trigger (throttle, downgrade, or escalate).

### 2.4 Decision rationale (why these choices)

- **Quality must be an SLI, not just a metric.** If it only lives on a
  dashboard no one owns, it will not get defended. Giving it the same
  budget-math treatment as latency is what turns "the model got worse" into an
  actionable burn.
- **Per-workload-class SLOs.** Chat, batch, embedding, and agent workloads
  operate on different parts of the engine's throughput-vs-latency curve. A
  single SLO over all of them hides regressions in the smallest class. This is
  the Workbook's "define SLOs around user journeys" rule (Workbook, Ch. 2),
  applied to AI's differentiated workload mix.
- **Multi-window multi-burn-rate alerting from day one.** With the fast
  failure modes in AI workloads (KV-cache storms, prompt regressions, provider
  outages) a single-threshold alert is useless (Workbook, Ch. 5 explicitly
  recommends the multi-window multi-burn-rate pattern).
- **Do not make the SLA tighter than the SLO.** The SLO is your *internal*
  commitment; the SLA is the *external* contract. Keep slack between them so
  one burn event does not breach a customer contract (SRE Book, Ch. 4).
- **Don't put metrics you cannot continuously measure into an SLA.** A
  judge-pass quality signal is good enough for an internal SLO; it is almost
  never ready to be an external SLA because the judging model itself drifts.

## 3. Validation steps

A learner's submission is validated by walking through four checks. Any "no"
is a failure.

1. **Definitions match the Google SRE Book.** For each of {SLI, SLO, SLA,
   error budget, burn rate}, the learner's one-line definition is
   substantively consistent with Google SRE Book, Ch. 3–4 and Workbook, Ch. 2.
2. **The AI framing column exists and is specific.** The learner cannot just
   restate the classical definition — they have to say *what changes* for AI
   workloads. Minimum bar: they call out that correctness is distributional,
   latency is multi-phase (TTFT / ITL), and saturation on GPU-bound inference
   is bimodal.
3. **At least four SLIs instantiated.** Of these at least one must be
   latency-based, one availability-based, one quality-based, and one
   saturation- or cost-based. For each, the learner says explicitly how it is
   measured and what SLO window they would pick.
4. **Burn-rate thinking is present.** Somewhere the learner describes a
   realistic failure that produces a high short-window burn rate (model push
   regression, KV-cache storm, provider degradation, prompt-template change,
   etc.).

## 4. Rubric / review checklist

Score each row independently; passing bar is **12/16** with no row at 0.

| Row | 0 (missing) | 2 (partial) | 4 (solid) |
|---|---|---|---|
| **Definitions** | One or more classical terms wrong or missing. | All five terms present but AI-framing column missing. | Five terms defined correctly AND re-framed for AI workloads. |
| **SLI catalog** | ≤ 2 SLIs, or no measurement sources named. | 3 SLIs with measurement sources. | ≥ 4 SLIs, each with measurement source and SLO window. |
| **Quality SLI treated as first-class** | Not mentioned, or only "quality should be good." | Mentioned but no measurement method. | Concrete quality SLI (judge-pass, grounding, refusal-rate) with a plausible source and window. |
| **Burn rate / policy linkage** | Not mentioned. | Burn rate defined but no example. | Burn-rate math *and* ≥ 1 realistic AI failure example that produces high short-window burn. |

## 5. Common mistakes

- **Treating HTTP 200 as the availability SLI for a streaming endpoint.** A
  200 can be followed by a mid-stream abort. Use *stream completed* on the
  denominator-eligible population.
- **Collapsing "quality" to a single eval score.** Quality is a set of SLIs
  (groundedness, refusal correctness, safety-pass), not one. Pick the ones
  that reflect the user journey.
- **Using one global latency SLO.** p95 TTFT on chat is not the same system as
  p95 TTFT on batch summarization. Per-workload-class, or the SLO is meaningless.
- **Picking SLO numbers out of the air.** The exercise does not require the
  "right" numbers; it requires the learner to explain the *process* for
  choosing them (user expectation, achievable headroom, SLA slack).
- **Mixing up SLO and SLA.** The SLA is the external contract. If the learner
  writes "our SLA is 99.0% TTFT ≤ 1 s" without any external contract context,
  they have conflated the two.
- **Treating cost as a non-reliability concern.** If the business commits to a
  $/token number, it is an SLI like any other; a cost burn event has the same
  shape as a latency burn event.
- **Forgetting that quality is non-deterministic.** A quality SLI measured on
  a sampled population has statistical noise; the SLO window has to be long
  enough that noise doesn't trip the budget.

## 6. References

- Google SRE Book, Ch. 3 "Embracing Risk", Ch. 4 "Service Level Objectives",
  Ch. 5 "Eliminating Toil", Ch. 6 "Monitoring Distributed Systems" —
  <https://sre.google/sre-book/table-of-contents/>.
- *The Site Reliability Workbook*, Ch. 2 "Implementing SLOs", Ch. 4 "Service
  Level Objectives", Ch. 5 "Alerting on SLOs" —
  <https://sre.google/workbook/table-of-contents/>.
- vLLM production metrics reference (histogram names for TTFT / TPOT, queue
  depth, KV-cache usage) — <https://docs.vllm.ai/en/latest/serving/metrics.html>.
- Brendan Gregg, *Systems Performance*, 2nd ed., for the USE method.
- Tom Wilkie, "The RED method" (Weaveworks, 2015) —
  <https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/>.
- NIST AI Risk Management Framework —
  <https://www.nist.gov/itl/ai-risk-management-framework> — used only to
  motivate why quality / safety belong in the SLI set.
