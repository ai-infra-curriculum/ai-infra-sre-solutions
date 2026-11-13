# Exercise 05 — Non-Abstract Large-System Design (NALSD) Review of an AI Stack

> Reference solution. The learner is asked to perform a NALSD review on a
> mid-sized AI inference stack: start from requirements, derive capacity
> from first principles, draw a diagram, identify failure modes, and arrive
> at a defensible operating plan. This is the *methodology* exercise for
> mod-102 — the output is a review report, not a tuning recipe.

## 1. Solution overview

Non-Abstract Large-System Design, from *The Site Reliability Workbook*, Ch. 12,
is the discipline of reasoning about a system end-to-end in **concrete
numbers**: how many requests, how many servers, how much memory, how much
tail latency, how many failure domains. The classic NALSD loop is:

1. State the problem and the SLOs.
2. Design a non-distributed version that could satisfy them (if only one
   machine were allowed).
3. Scale out, show your math at every step.
4. Identify failure modes and the failure domains that contain them.
5. Report the operating plan (capacity, scaling levers, fallbacks).

Applied to an AI inference stack the loop is unchanged; three substitutions
make it AI-shaped:

- **"QPS" becomes "requests/s AND tokens/s".** Capacity math on GPU is
  token-throughput-bound, not request-count-bound.
- **"Latency budget" splits into TTFT and TPOT.** You cannot compute
  capacity from end-to-end latency alone.
- **"Failure modes" include quality and vendor degradation.** The design
  has to explicitly budget for these — they are not extraordinary events.

The worked answer below is a full NALSD review of a representative LLM
inference platform. Numbers are **illustrative** — the deliverable is the
*structure* of the math and the review.

## 2. Implementation — a NALSD review

### 2.1 Problem statement and SLOs

- **Workload:** externally-facing chatbot backend.
- **Workload class mix (steady state):**
  - `/chat` interactive: 60% of traffic, median prompt 500 tokens, median
    output 300 tokens, streamed.
  - `/agent` tool-using: 10%, median prompt 1500 tokens (plus tool results),
    median output 400 tokens.
  - `/embed` retrieval support: 30%, average 200 tokens in, no output tokens
    (embedding vector only).
- **SLOs** (from `mod-103`, instantiated here for the review):
  - `/chat`: TTFT p99 ≤ 1.0 s; TPOT p95 ≤ 50 ms/token; stream-complete
    99.9%.
  - `/agent`: end-to-end p95 ≤ 15 s.
  - `/embed`: p99 ≤ 100 ms.
  - Quality (judge-pass) ≥ 97% on `/chat`.
- **Traffic (peak, from projection):** 400 chat requests/s, 70 agent
  requests/s, 200 embed requests/s. **Express these as tokens/s** too:
  roughly 240k input tokens/s and 144k output tokens/s on `/chat` at peak;
  more on `/agent`; `/embed` roughly 40k input tokens/s.

> Pick numbers appropriate to the lab. The grading is on the *method*, not
> the exact figures.

### 2.2 Step 1 — Non-distributed sanity (one GPU)

The purpose of this step is to show the learner understands the limiting
resource.

- Pick a representative model (say an 8B-parameter chat model on an 80 GB
  GPU). From the engine's advertised throughput (vLLM benchmarks, or
  measured) we estimate a single-GPU steady-state throughput of **O(1000–3000
  output tokens/s)** at the operating latency targets. The order-of-magnitude
  figure is a placeholder; the lab should substitute an exact benchmark
  measurement taken on the model and hardware actually in use.
- Required output tokens/s at peak on `/chat` alone ≈ 144k.
- Therefore the minimum number of equivalent GPUs needed for `/chat` output
  throughput is on the order of **50–100 GPUs**, before any headroom.
- This is the *lower bound* — you cannot do this with a single machine, and
  you cannot do it with fewer than ~50 GPUs in the configuration assumed.

The point of this calculation is not the exact answer. It is to establish
that the bottleneck is **GPU output-token throughput**, and all design
decisions after this will be driven by that constraint.

### 2.3 Step 2 — Scale out (first distributed version)

Apply standard SRE multipliers to the lower bound:

- **Headroom for the SLO knee (×1.5–2.0).** Because the engine's
  throughput-vs-latency curve bends sharply, running close to the knee
  violates the TPOT SLO. Target steady-state utilization at 50–60%.
- **Headroom for traffic variance (×1.2–1.3).** Diurnal + weekly swings
  observed.
- **Redundancy for 1 zone failure (×1.5 for 3-zone, ×2.0 for 2-zone).**
  Use N+1 at the zone level; the target is "lose one zone, still inside
  SLO."
- **Separate pool for `/embed`**, because embed workloads run on a smaller
  model with different batching characteristics — pooling with chat
  reduces efficiency for both.

Combining: 50–100 GPUs × 1.75 × 1.25 × 1.5 ≈ **170–330 GPUs** for `/chat`
alone, spread across 3 zones. Agent pool and embed pool computed similarly
and added.

Document the math row-by-row. A learner who writes "we need 200 GPUs
because the model is big" has not shown their work.

### 2.4 Step 3 — Diagram

```
             ┌──────────────┐          ┌───────────────┐
Client ────▶ │  API Gateway │ ──────▶  │  LLM Router   │
             │ (auth / rl)  │          │ (model / ab)  │
             └──────┬───────┘          └──────┬────────┘
                    │                         │
                    │                         ▼
                    │             ┌───────────────────────┐
                    │             │ Inference pools       │
                    │             │                       │
                    │             │   chat pool (3 zones) │
                    │             │   agent pool (3 zones)│
                    │             │   embed pool (3 zones)│
                    │             └──────────┬────────────┘
                    │                        │
                    │                        ▼
                    │             ┌───────────────────────┐
                    ▼             │  Tool / Retrieval     │
             ┌──────────────┐     │   - vector store      │
             │ Eval & Guard │ ◀── │   - web / DB tools    │
             │  (sync +     │     └───────────────────────┘
             │   async)     │
             └──────┬───────┘
                    │
                    ▼
           ┌─────────────────┐
           │ Observability   │
           │  Prom + traces  │
           └─────────────────┘
```

Named components (each is a failure domain):

- **API Gateway.** Auth, rate-limiting, admission control. Stateless.
- **LLM Router.** Chooses model / provider / A-B assignment per request.
  Stateless. **Holds the policy knobs** from Exercise 02 (degrade, failover).
- **Inference pools.** vLLM / SGLang / TRT-LLM fleets. 3 zones, pool per
  workload class. (See Exercise 04 for knobs.)
- **Tool / Retrieval plane.** Vector store, databases, tool endpoints.
  Separate SLOs of their own.
- **Eval & Guard.** Synchronous safety / policy checks on the request path
  (fast); asynchronous quality sampling off the path (slower, for SLIs Q1 /
  Q2 from Exercise 02).
- **Observability stack.** Prometheus for engine + infra metrics;
  tracing for request paths; cost / token accounting for cost SLIs.

### 2.5 Step 4 — Failure-mode table

| Mode | Blast radius | SLO affected | Containment mechanism |
|---|---|---|---|
| Single GPU / node loss | 1 replica. | Latency only if saturated. | N+1 within zone + autoscaler. |
| Single-zone loss (power / network) | 1 zone. | Latency, availability. | 3-zone deployment, LB health-checks, autoscaler across zones. |
| Model-weight regression (new push) | 100% of affected workload class. | Quality, possibly latency. | Canary promotion; error-budget policy D.2 pin; D.3 rollback. |
| Prompt-template regression | 100% of affected route. | Quality. | Template versioning; D.2 pin. |
| Retriever / index regression | 100% of RAG traffic. | Quality (grounding SLI). | Index pinning; D.6 quality-incident flow. |
| KV-cache eviction storm | Whole inference pool. | Latency (TTFT). | Pre-provisioned headroom; admission control → fast-503. |
| Vendor / provider degradation | All traffic routed to that provider. | Latency, availability. | D.5 vendor failover to pre-tested secondary. |
| Safety / unsafe-output report | 1 user → class of users. | Trust, compliance. | Supersedes error-budget policy; safety-incident runbook (NIST AI RMF `MANAGE`). |
| Tool / retrieval outage | Agent + RAG. | Latency + quality. | Circuit-breakers; defined degraded responses. |
| Eval pipeline silent failure | Quality signal disappears. | Monitoring of Quality SLO. | Dead-man's switch on judge-pass sample volume. |
| Observability outage | All SLIs stale. | Monitoring. | Metrics redundancy (prom federation / HA); dead-man's switch alert. |

Every row has (a) a named blast radius, (b) an SLO it affects, and (c) a
containment mechanism that already exists in the design or the error-budget
policy from Exercise 02.

### 2.6 Step 5 — Operating plan

1. **Baseline fleet:** 3-zone, per-workload-class pools sized by the Step 2
   math, with the headroom multipliers written down.
2. **Scaling levers (in order):**
   - Autoscaler on queue depth / KV utilization per pool.
   - Dynamic batching knobs (`max-num-batched-tokens`, `max-num-seqs` —
     see Exercise 04) — scheduled, not panic-tuned.
   - Admission control: fast-503 with `Retry-After` beyond knee threshold.
   - Model-tier downgrade for tier-2 customers under sustained saturation.
3. **Degradation levers (from the error-budget policy):** downgrade
   `max_tokens`, downgrade model variant, prompt-pin, index-pin, vendor
   failover.
4. **Drills:** monthly chaos (Exercise 03) covering: zone loss, vendor
   failover, model rollback, index swap. Each drill leaves a written
   result in the on-call handoff.
5. **Capacity review cadence:** monthly, with the Step 2 math re-run
   against the previous month's actual traffic.
6. **Launch gating:** any change to model, prompt, or retriever must have
   its canary plan (see `mod-111`) referenced; launches are governed by the
   error-budget policy's launch-freeze trigger.

### 2.7 Decision rationale

- **Capacity is a token-throughput problem.** Learners who compute capacity
  in requests/s miss the dominant cost. The exercise is explicitly checking
  that they convert to tokens/s.
- **Per-workload-class pools.** Pooling `/chat` and `/embed` hurts both
  (batches of mixed shape hurt throughput; different batching policies help
  each individually). This is also why Exercise 01 insists on per-class
  SLOs.
- **Headroom is non-negotiable on GPU workloads.** Running close to the knee
  on an engine is the fastest way to turn a hot morning into a page storm.
  Half-utilization is not waste, it is the SLO.
- **Vendor failover is a first-class failure mode.** Design *to* vendor
  failover, don't bolt it on: dual-provider adapters at the router, tested
  monthly. (The Vendor-risk module `mod-107` goes deeper; this review just
  pays the required tax.)
- **Eval pipeline needs a dead-man's switch.** The most dangerous failure
  mode for an AI system is *silent quality failure*: the metric itself
  stops being produced, and the dashboard shows "we're fine." This is why
  the failure table has a row for it.
- **Keep the diagram small.** NALSD rewards *fewer components you
  understand well* over more components you don't. A review that shows 25
  boxes nobody can draw the blast radius of is worse than 7 boxes that
  each have a named failure mode.

## 3. Validation steps

1. **Numbers are present, not placeholder-only.** The learner has shown
   at least one tokens/s calculation for at least one route.
2. **Headroom multipliers are named.** Knee-headroom, variance,
   redundancy. Each has a numerical multiplier and a rationale.
3. **Per-workload-class pools.** The design does not pool `/chat` with
   `/embed`. If it does, the learner must defend it.
4. **Diagram enumerates failure domains.** Every box in the diagram
   reappears in the failure-mode table with a blast radius.
5. **Failure modes include quality, vendor, and observability.** Not just
   infra.
6. **Operating plan references the error-budget policy from Exercise 02.**
   The levers in the operating plan are the levers already pre-authorized.
   No new improvised controls.
7. **Capacity review cadence is explicit.** Monthly or similar, with the
   math re-run.

## 4. Rubric / review checklist

Score each row independently; passing bar is **16/24** with no row at 0.

| Row | 0 | 2 | 4 |
|---|---|---|---|
| **Problem statement** | SLOs absent. | SLOs present but no traffic shape. | Full SLO + traffic shape (req/s AND tok/s). |
| **Non-distributed sanity** | Skipped. | Mentioned without math. | Math shown with the limiting resource identified. |
| **Scale-out math** | Hand-waved. | One multiplier only. | Headroom + variance + redundancy multipliers, each justified. |
| **Diagram + failure domains** | Missing diagram. | Diagram without blast radius. | Diagram where every component shows up in the failure table. |
| **Failure modes breadth** | Infra only. | Infra + model regression. | Infra + model + vendor + quality + observability. |
| **Operating plan** | Capacity only. | Capacity + scaling levers. | Capacity + scaling levers + degradation + drill cadence + review cadence, tied to Exercise 02 policy. |

## 5. Common mistakes

- **Doing the capacity math in requests/s only.** Misses the dominant
  variable; the resulting plan under-sizes dramatically.
- **Running the pool at 90% utilization.** The engine's knee punishes this
  as a tail-latency explosion.
- **Pooling embed with chat.** Looks efficient on paper; destroys TTFT on
  chat because embed prefills clog batches.
- **No zone redundancy.** One zone loss ⇒ SLO breach ⇒ error-budget wiped
  in minutes.
- **Omitting the eval / safety plane.** If quality is an SLI (and it is),
  the eval pipeline is a *production* system with its own reliability
  requirements.
- **Observability as a footer.** Observability is on the critical path:
  no SLIs, no SLO policy, no on-call. Treat it as a first-class pool.
- **Unlinked to the policy.** The operating plan invents levers that aren't
  in the error-budget policy. The on-call cannot use them.
- **No dead-man's switch on the quality SLI.** Silent eval failure is the
  worst class of AI outage because nobody notices.
- **Over-drawing.** 25 boxes with no blast-radius story is not a design,
  it is an org chart.

## 6. References

- *The Site Reliability Workbook*, Ch. 12 "Introducing Non-Abstract Large
  System Design" — <https://sre.google/workbook/non-abstract-design/>.
- Google SRE Book, Ch. 6 "Monitoring Distributed Systems", Ch. 22
  "Addressing Cascading Failures" (feeds the admission-control / fast-503
  lever) — <https://sre.google/sre-book/table-of-contents/>.
- NIST AI Risk Management Framework (`MAP`, `MEASURE`, `MANAGE` functions)
  — grounds the inclusion of safety and quality in the failure-mode table
  — <https://www.nist.gov/itl/ai-risk-management-framework>.
- OWASP Machine Learning Security Top 10 (motivates the safety-incident
  row of the failure table) —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
- MITRE ATLAS (threat-informed failure modes for the design review) —
  <https://atlas.mitre.org/>.
- vLLM, SGLang, TensorRT-LLM docs — see Exercise 04.
