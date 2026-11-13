# Exercise 01 — Queueing-Theory Base Model for Prefill-Bound and Decode-Bound Workloads — Solution

## 1. Solution overview

This exercise asks the learner to build a *base* capacity model for an LLM
serving tier by separating the two workloads that live on the same GPUs:

- **Prefill** — the compute-bound pass that processes the input prompt. Latency
  scales roughly linearly with prompt length; throughput is bounded by GEMM
  throughput on the device.
- **Decode** — the memory-bandwidth-bound autoregressive loop that emits tokens
  one at a time. Latency per token (TPOT / ITL) is bounded by KV-cache reads
  and HBM bandwidth; throughput is bounded by how many concurrent sequences
  you can batch without blowing the KV-cache budget.

A learner who turns in a single global "requests per second" number has not
done the exercise. The whole point is to model the two regimes separately,
compose them, and show where the fleet saturates first.

The reference answer below delivers four artifacts:

1. A worked arithmetic example that sizes a fleet from a traffic forecast,
   showing each intermediate step so a reviewer can re-run the math.
2. A short written rationale (the "why" each term is in the formula), tied
   to Little's Law and to the M/M/c approximation used as a base model.
3. The decision rule a reviewer should use when deciding whether to add
   prefill capacity, decode capacity, or both.
4. A grading rubric so a reviewer can score a learner submission in roughly
   ten minutes.

The model is deliberately simple. It is a *base* — not a replacement for
measured load tests (that is Exercise 06). Its job is to make the learner
articulate the ratio between prefill cost and decode cost before they touch
k6 or an HPA.

## 2. Worked implementation — the base model

### 2.1 Variables and units (fix these first)

| Symbol | Meaning | Units |
|---|---|---|
| `λ` | Arrival rate of requests | req/s |
| `L_in` | Mean prompt length | input tokens/req |
| `L_out` | Mean generated length | output tokens/req |
| `T_pf` | Prefill service time per token | s / input-token on one replica |
| `T_dec` | Decode service time per token | s / output-token on one replica |
| `B` | Max concurrent sequences per replica (decode batch cap) | sequences |
| `U_target` | Target steady-state utilisation | unitless, e.g. 0.65 |
| `N` | Replicas to provision | integer |

Prefill and decode use **different** service-rate numbers because they are
different kernels on the same hardware. Do not blur them into one.

### 2.2 Per-replica service rates

- Prefill tokens served per second per replica: `μ_pf = 1 / T_pf`
- Decode tokens served per second per replica, saturated batch:
  `μ_dec = B / T_dec`

The batch factor `B` appears only on the decode side because decode is where
continuous batching compounds throughput; prefill is already dense GEMM and
usually processes one (or a very small micro-batch of) prompt at a time.

### 2.3 Offered load on each stage

Offered prefill rate (in tokens/s across the fleet):

```
Λ_pf = λ · L_in
```

Offered decode rate (in tokens/s across the fleet):

```
Λ_dec = λ · L_out
```

### 2.4 Required replicas — take the max of the two stages

```
N_pf  = ceil( Λ_pf  / ( μ_pf  · U_target ) )
N_dec = ceil( Λ_dec / ( μ_dec · U_target ) )
N     = max( N_pf, N_dec )
```

The fleet size is set by **whichever stage saturates first**. If `N_pf > N_dec`,
the workload is *prefill-bound* and you should scale by prompt length or by
input-TPS. If `N_dec > N_pf`, the workload is *decode-bound* and you should
scale by output tokens or concurrent sequences. Reporting a single `N` without
disclosing which term dominated is the failure mode this exercise is designed
to catch.

### 2.5 Queue-wait sanity check (M/M/c approximation)

A base model also needs a *latency* sanity check so the learner notices when
the fleet is sized for average throughput but will still miss its TTFT SLO.
Treat the prefill stage as an M/M/c queue with `c = N_pf` servers and service
rate `μ_pf` per server. With offered load `ρ = Λ_pf / (c · μ_pf)`, the
Erlang-C formula gives the probability of queueing (`P_wait`), and the mean
wait before service is approximately `P_wait / (c · μ_pf · (1 − ρ))`.

M/M/c is wrong in the details (arrivals are bursty, service times are
heavy-tailed, decode is batched), but it is good enough to answer the
question *"how fast does my queue-wait blow up as `ρ` approaches 1?"*. The
intended takeaway is the knee: at `ρ = 0.9` mean wait is already several
times service time, which is why the rubric below marks `U_target ≥ 0.85`
as a defect.

Reference Little's Law for the composition: the number of in-flight requests
`L = λ · W` (where `W` is end-to-end time). A learner who derives `W` from
TTFT (prefill wait + first-token service) plus TPOT · `L_out` and compares it
against the SLO has done the exercise correctly.

### 2.6 Worked numeric example

Assume the exercise fixture gives these numbers (or the learner measures them):

- `λ = 40` req/s
- `L_in = 800` input tokens/req, `L_out = 200` output tokens/req
- `T_pf = 1.0` ms per input token on one replica → `μ_pf = 1000` tok/s
- `T_dec = 25` ms per output token on one replica → single-stream decode
  throughput = 40 tok/s; with `B = 16` concurrent sequences, `μ_dec = 640` tok/s
- `U_target = 0.65`

Then:

```
Λ_pf  = 40 · 800 = 32,000  input-tok/s
Λ_dec = 40 · 200 =  8,000  output-tok/s

N_pf  = ceil( 32,000 / (1000 · 0.65) ) = ceil(49.23)  = 50
N_dec = ceil(  8,000 / ( 640 · 0.65) ) = ceil(19.23)  = 20

N     = max(50, 20)                                   = 50  (prefill-bound)
```

Decision: this fleet is prefill-bound. Headroom actions in priority order:

1. Shorten prompts (prompt-template review, context compression, retrieval
   trimming) — biggest lever on `L_in`.
2. Move prefill to a hardware tier with higher GEMM throughput, or to a
   disaggregated prefill pool so decode replicas are not starved by prefill.
3. Increase `N_pf` by adding replicas.

Changing the decode batch `B` is *not* the lever here — the stage that is
saturating has no decode in it.

### 2.7 Reverse case — decode-bound

Flip the example: `L_in = 100`, `L_out = 2,000` tokens/req. Now:

```
Λ_pf  = 40 ·   100 =    4,000   input-tok/s
Λ_dec = 40 · 2,000 =   80,000   output-tok/s

N_pf  = ceil(   4,000 / (1000 · 0.65) ) =   7
N_dec = ceil(  80,000 / ( 640 · 0.65) ) = 193

N     = max(7, 193)                     = 193  (decode-bound)
```

Headroom actions here are the mirror image: increase decode batch `B`, move
to a hardware tier with higher HBM bandwidth, or shed load on output tokens
(`max_tokens`, streaming cutoffs). Shortening prompts barely helps.

### 2.8 Decision rule (how to use the model)

Use this three-step check every time a demand forecast changes:

1. Recompute `N_pf` and `N_dec` from the new `λ`, `L_in`, `L_out`.
2. Whichever is larger names the regime (prefill-bound vs decode-bound).
3. Apply the headroom levers for that regime first. Only add replicas when
   the per-token-cost levers are exhausted.

The model does not replace a real load test. It picks the right *shape* of
intervention so the load test is not wasted on the wrong lever.

## 3. Validation steps

A reviewer should confirm the submission passes each of these checks:

1. **Dimensional analysis** — every term in `N_pf` and `N_dec` carries units
   that cancel to a dimensionless replica count. If the learner's formula
   produces `tokens / seconds² · replicas`, the units reveal a dropped
   factor.
2. **Degenerate cases** — set `L_out = 0`; `N_dec` should go to zero (pure
   embedding workload). Set `L_in ≈ 0`; `N_pf` should go to zero (pure
   continuation workload). If the model does not produce these answers the
   two stages are entangled incorrectly.
3. **Linearity in `λ`** — doubling `λ` with everything else fixed should
   double both `Λ_pf` and `Λ_dec`, and therefore (modulo ceiling) double
   `N_pf` and `N_dec`. A submission where `N` grows faster-than-linearly in
   `λ` at low utilisation has a bug.
4. **Utilisation reality check** — the answer must call out that `U_target`
   should be in the 0.5–0.75 range for interactive workloads. Any submission
   with `U_target ≥ 0.85` must justify it against the wait-time blow-up at
   `ρ → 1`; most will not be able to.
5. **Prefill/decode naming** — the written rationale must use the words
   "prefill-bound" or "decode-bound" and state which one the numeric example
   produced. A submission that only reports total replicas has failed the
   core learning objective.
6. **Sanity vs. load test** — the submission must explicitly note that this
   is a *base* model and that Exercise 06 (k6/Locust) is the empirical
   verification step.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | Variables table with units, prefill and decode kept separate | 2 |
| 2 | Correct per-replica service rates `μ_pf` and `μ_dec = B / T_dec` | 2 |
| 3 | `N = max(N_pf, N_dec)` with `ceil()` and `U_target` factor | 2 |
| 4 | Worked numeric example with intermediate values shown | 1 |
| 5 | Regime named (prefill-bound vs decode-bound) with the right levers | 1 |
| 6 | Queue-wait / Little's Law sanity check on TTFT | 1 |
| 7 | Honest statement that this is a base model, not a replacement for
      load tests | 1 |

Deductions (any one knocks a submission below pass):

- Treats prefill and decode as a single combined stage: −3.
- No units on any variable: −2.
- `U_target` silently set to 1.0 (or no `U_target` at all): −2.
- Claims the model is sufficient without load testing: −2.

## 5. Common mistakes

- **Collapsing the two stages into one `requests/s` number.** Produces a
  plausible-looking but unactionable answer; the learner cannot tell which
  lever to pull when the fleet saturates.
- **Using throughput at `U = 1.0`.** Steady-state queueing is unstable near
  full utilisation; the base model must bake in a utilisation target so
  TTFT does not fall off a cliff when traffic bursts.
- **Forgetting the decode batch factor `B`.** Decode throughput per replica
  is not `1 / T_dec`; it is `B / T_dec` under continuous batching. Missing
  this makes every fleet look decode-bound.
- **Reasoning in requests per second when the hardware ships in tokens per
  second.** Convert through `L_in` and `L_out` at the start; keep requests
  for arrival rate only.
- **Claiming the base model replaces load testing.** It does not. It sizes
  the search space the load test will exercise (Exercise 06).
- **Changing decode `B` to fix a prefill-bound fleet.** It will not help;
  the levers must match the regime.
- **Ignoring KV-cache memory as a cap on `B`.** `B` is not free — it is
  bounded by HBM. A submission that treats `B` as a tunable with no ceiling
  is engineering in the wrong direction. (The exercise's local fixture
  states a `B` cap; use it.)

## 6. References

- Google SRE Workbook — *Managing Load* and *Non-Abstract Large System
  Design*: utilisation targets, queueing intuition, and the "measure, then
  model" posture this exercise sits on top of —
  <https://sre.google/workbook/>.
- NIST AI Risk Management Framework (AI RMF 1.0) — Manage function, MG-2.3:
  resource planning and allocation are part of trustworthy AI operations —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- Little's Law and M/M/c / Erlang-C are standard results from classical
  queueing theory; any introductory text (Kleinrock; Allen; Gross & Harris)
  covers the derivations invoked above.
- Local exercise context: prefill vs decode terminology, continuous
  batching, and TTFT / TPOT as the latency decomposition the fleet is sized
  against — see the mod-104 observability exercises in this repo for the
  dashboards that produce the measured `T_pf`, `T_dec`, and `B` values the
  base model consumes.
