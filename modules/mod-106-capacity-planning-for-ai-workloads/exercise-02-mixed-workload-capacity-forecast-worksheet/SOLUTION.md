# Exercise 02 — Mixed-Workload Capacity Forecast Worksheet — Solution

## 1. Solution overview

Exercise 01 fit a base model to a *single* workload shape. Real serving tiers
host several workload classes at once — a chatbot stream, a batch
summarisation job, and a RAG-with-retrieval path — each with different
prompt/output distributions and different latency SLOs. A single average
hides the workload that is about to break first.

This exercise asks the learner to produce a **mixed-workload capacity
forecast worksheet**: a table that decomposes offered load by class,
computes per-class prefill/decode demand, rolls them up into a fleet-wide
replica count, and shows the headroom each class retains at the forecast
level.

The reference solution below provides:

1. A canonical worksheet schema (what columns must be present and why).
2. A worked example with three workload classes, numbers filled in end to
   end so a reviewer can recompute each cell.
3. The rollup rule that composes per-class demand into a single replica
   count.
4. A rubric and common-mistakes list so the worksheet can be graded in a
   fixed-length review.

The worksheet is the artifact the learner carries into capacity reviews,
budget meetings, and the HPA tuning conversation (Exercise 05). It is
*not* a forecast *tool*; it is the forecast *shape* that any tool or
spreadsheet must produce.

## 2. Worked implementation — the worksheet

### 2.1 Column schema (and why each column is there)

| Column | Meaning | Why it is required |
|---|---|---|
| `class` | Workload name (e.g. `chat_stream`, `batch_summary`, `rag_qa`) | So SLOs and levers can be attributed to a traffic source rather than averaged away |
| `λ_c` (req/s) | Forecast arrival rate at the horizon (e.g. peak + 30 d) | Capacity is sized to the forecast, not today's average |
| `L_in_c` (tok/req) | Mean (and p95) input tokens for this class | Drives prefill demand; p95 is required because prefill is compute-bound |
| `L_out_c` (tok/req) | Mean (and p95) output tokens | Drives decode demand; p95 drives tail-latency sizing |
| `priority_c` | `interactive` / `batch` / `background` | Determines which SLO applies and whether this class is eligible for admission control |
| `SLO_c` | TTFT and TPOT budget for this class | A `batch` class with a 60 s TTFT budget cannot be sized to the same target as a chat stream |
| `μ_pf`, `μ_dec` (tok/s per replica) | From Exercise 01 measurements | Base model inputs |
| `U_c` | Per-class utilisation target | Interactive classes get lower targets (0.5–0.65); batch gets higher (0.75–0.85) |
| `Λ_pf_c`, `Λ_dec_c` | `λ_c · L_in_c`, `λ_c · L_out_c` | Fleet-wide offered load per stage per class |
| `N_pf_c`, `N_dec_c` | Per-class replica demand | Shows which class drives which stage |
| `headroom_c` | `(N_provisioned − N_demand) / N_provisioned` at the forecast | The number the capacity review actually looks at |
| `regime_c` | `prefill-bound` or `decode-bound` | So the lever conversation is per-class |

A worksheet that omits any of these columns fails the "handoff" test — it
cannot be used as the capacity review artifact.

### 2.2 Rollup rule

For a shared fleet with pooled replicas, the mixed-workload replica count
at each stage is the *sum* over classes of per-class token demand divided by
per-replica service rate, discounted by the utilisation target of the
least-tolerant class sharing that stage:

```
U_shared = min over c of U_c                       # the tightest SLO wins

N_pf  = ceil( Σ_c Λ_pf_c  / ( μ_pf  · U_shared ) )
N_dec = ceil( Σ_c Λ_dec_c / ( μ_dec · U_shared ) )
N     = max( N_pf, N_dec )
```

If interactive and batch classes share the same replicas, they share the
same utilisation — and therefore the interactive SLO dominates the whole
fleet's capacity cost. This is why many teams physically segregate the
batch class onto its own pool: it lets `U_shared` rise to the batch target
on the batch pool and stay at the interactive target on the interactive
pool.

When classes are segregated onto separate pools, apply the rollup inside
each pool independently and sum the pools for the total replica count.

### 2.3 Worked example — three classes on one pool

Fixture (treat these as the forecast inputs for the next quarter):

| class | `λ_c` | `L_in_c` | `L_out_c` | priority | SLO (TTFT p95 / TPOT p95) | `U_c` |
|---|---|---|---|---|---|---|
| `chat_stream` | 20 req/s | 400 | 300 | interactive | 800 ms / 50 ms | 0.60 |
| `rag_qa` | 10 req/s | 1,600 | 150 | interactive | 1,500 ms / 50 ms | 0.60 |
| `batch_summary` | 5 req/s | 3,000 | 500 | batch | 60 s / 200 ms | 0.80 |

Measured service rates from the base model (Exercise 01):

- `μ_pf = 1,000` input-tok/s per replica
- `μ_dec = 640` output-tok/s per replica (continuous batch `B = 16`)

Step 1 — per-class demand (tokens/s across the fleet):

```
chat_stream:     Λ_pf =    8,000,  Λ_dec =  6,000
rag_qa:          Λ_pf =   16,000,  Λ_dec =  1,500
batch_summary:   Λ_pf =   15,000,  Λ_dec =  2,500
Σ Λ:             Λ_pf =   39,000,  Λ_dec = 10,000
```

Step 2 — shared pool, dominated by interactive utilisation target
`U_shared = min(0.60, 0.60, 0.80) = 0.60`:

```
N_pf  = ceil( 39,000 / (1,000 · 0.60) ) = ceil(65.0)   = 65
N_dec = ceil( 10,000 / (  640 · 0.60) ) = ceil(26.04)  = 27
N     = max(65, 27)                                     = 65   (prefill-bound)
```

Step 3 — segregated-pool counterfactual (what segregation would save):

- Interactive pool (`chat_stream` + `rag_qa`), `U = 0.60`:
  - `Σ Λ_pf = 24,000`, `Σ Λ_dec = 7,500`
  - `N_pf = ceil(40.0) = 40`, `N_dec = ceil(19.53) = 20` → `N_int = 40`
- Batch pool (`batch_summary`), `U = 0.80`:
  - `Λ_pf = 15,000`, `Λ_dec = 2,500`
  - `N_pf = ceil(18.75) = 19`, `N_dec = ceil(4.88) = 5` → `N_batch = 19`
- Total segregated: `40 + 19 = 59` replicas vs 65 shared.

Segregation saves ~9% replicas at this forecast because batch can run at
higher utilisation on its own pool. Whether that saving beats the
operational cost of maintaining two pools is the reviewer's call — the
worksheet exposes the tradeoff; it does not decide it.

Step 4 — per-class headroom at provisioned `N = 65` (shared):

```
chat_stream  prefill share =  8,000 / 39,000 = 20.5% →  N demand ≈ 13.3 →  headroom vs 65: ~80%
rag_qa       prefill share = 16,000 / 39,000 = 41.0% →  N demand ≈ 26.7 →  headroom vs 65: ~59%
batch_summary prefill share= 15,000 / 39,000 = 38.5% →  N demand ≈ 25.0 →  headroom vs 65: ~62%
```

`rag_qa` is the top consumer. If the forecast moves, `rag_qa` is the class
whose `L_in` and `λ` the reviewer should question first.

### 2.4 Decision rule — reading the worksheet

The worksheet is finished when all three questions below have a one-line
answer written underneath it:

1. **Which class drives the fleet?** — the class with the largest
   per-stage `Λ` in the regime that `N` resolved to.
2. **Which class is at the lowest headroom?** — the class whose demand is
   the largest fraction of provisioned replicas; if two forecasts miss,
   this is the one that breaks first.
3. **Should this pool be split?** — segregate when a class's utilisation
   target differs from the pool's shared target by enough to justify the
   operational cost (typically `U_batch − U_interactive ≥ 0.2`).

### 2.5 Peak vs. average and the forecast horizon

Use the *peak* of the forecast, not the average. A fleet sized to the
average 2 h window will miss the TTFT SLO during the peak 15 min.
Standard practice is to size for `peak = p99 of hourly arrival over the
forecast horizon`, and to add a floor so a sub-hourly burst does not
cause a scale-out lag. The HPA (Exercise 05) covers the sub-hourly
response; the worksheet only has to be right at the forecast horizon.

## 3. Validation steps

1. **Every class row balances.** `Λ_pf_c = λ_c · L_in_c` and
   `Λ_dec_c = λ_c · L_out_c` must recompute to the number in the cell.
   A reviewer should spot-check one cell per class.
2. **Rollup uses the tightest `U_c` on shared pools.** If any interactive
   class shares a pool with a batch class, `U_shared` must be the
   interactive target. Submissions that average `U_c` across classes are
   wrong — the SLO is set by the tightest consumer.
3. **Regime named per class.** Each row carries a `regime_c` cell. A
   pool-level regime is derived from the stage that produced `N`.
4. **Peak-not-average.** The worksheet header must declare which quantile
   of the forecast it is sized against. "Monthly average" is a failure.
5. **Segregation counterfactual.** The worksheet must either (a) show the
   segregated-pool replica count alongside the shared count, or (b) carry
   a one-line note saying segregation was considered and rejected, with
   the reason. "We did not consider it" is a defect.
6. **Linkage to the base model.** `μ_pf` and `μ_dec` must come from the
   Exercise 01 base model with a citation to the measurement source;
   invented service rates fail.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | Column schema covers class, `λ`, `L_in`, `L_out`, priority, SLO, `U_c`, `Λ_pf`, `Λ_dec`, `N_pf_c`, `N_dec_c`, headroom, regime | 2 |
| 2 | Per-class numbers compute correctly end-to-end on the fixture | 2 |
| 3 | Shared-pool rollup uses `U_shared = min(U_c)` for mixed priorities | 1 |
| 4 | Segregated-pool counterfactual present or explicitly rejected | 1 |
| 5 | Peak (not average) forecast used, horizon stated | 1 |
| 6 | Top-consumer class and lowest-headroom class called out in prose | 1 |
| 7 | Regime named per class and at pool level | 1 |
| 8 | `μ_pf`, `μ_dec` traced back to the Exercise 01 measurement | 1 |

Deductions:

- Single global `requests/s` with no per-class decomposition: −4.
- Averaged `U_c` across priorities on a shared pool: −2.
- No forecast horizon stated: −1.
- Unresolved research placeholder left in a production artifact: −1.

## 5. Common mistakes

- **Sizing to "monthly average".** The whole point of a peak-forecast
  worksheet is to survive the hour that matters. Monthly averages mask
  the worst 1% of windows.
- **Averaging utilisation targets across priorities.** The SLO is set by
  the tightest consumer; averaging is wishful.
- **Omitting `L_in` p95 (or `L_out` p95).** Prefill is compute-bound and
  cares about tails, not just means; a mean-only worksheet under-sizes
  the fleet.
- **Ignoring segregation as an option.** Even when the answer is "no,
  stay shared," the worksheet must show that the option was weighed.
- **Reusing `μ_pf`, `μ_dec` from a different model or a different quant.**
  The base-model rates are per-model and per-quant; copying them across
  model families is a failure.
- **Reporting per-class replica counts without a shared rollup.** The
  pool is shared; the per-class demands must add up, not be read
  independently.
- **Confusing HPA-level burst sizing with forecast-level sizing.** The
  worksheet is the forecast; the HPA (Exercise 05) handles sub-hourly
  bursts. Do not try to solve both at once.

## 6. References

- Google SRE Workbook — *Managing Load* (workload prioritisation,
  admission control, utilisation as the primary dial) —
  <https://sre.google/workbook/>.
- NIST AI Risk Management Framework (AI RMF 1.0) — Manage function, MG-2
  sub-categories around resource allocation and risk-tolerated
  deployment —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- Little's Law (`L = λ · W`) provides the sanity check on in-flight
  counts implied by the forecast; see Kleinrock *Queueing Systems* for
  the standard derivation.
- Local exercise context: this worksheet consumes `μ_pf`, `μ_dec`, and
  `B` produced by Exercise 01's base model and is itself consumed by
  Exercise 05 (HPA tuning) and Exercise 06 (load-test verification).
