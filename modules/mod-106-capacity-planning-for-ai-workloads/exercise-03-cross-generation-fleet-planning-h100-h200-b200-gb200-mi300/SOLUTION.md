# Exercise 03 — Cross-Generation Fleet Planning (H100 / H200 / B200 / GB200 / MI300) — Solution

## 1. Solution overview

A production GPU fleet is almost never one generation. A team acquires
H100-class capacity in year one, lands H200 refreshes alongside existing
H100 inventory, is allocated a B200/GB200 block as a new generation ships,
and may hold an AMD MI300X pool for a second-source or price/perf reason.
These generations differ along axes that matter for capacity planning —
peak compute, HBM capacity, HBM bandwidth, interconnect bandwidth, and
software stack maturity — and a planner who treats them as interchangeable
units will over- or under-provision.

This exercise asks the learner to produce a **cross-generation fleet plan**
that:

1. Normalises each generation into a common capacity-planning currency so
   demand can be routed to the right pool.
2. Defines a routing policy that assigns workload classes to generations
   by the constraint that actually matters for that class (compute,
   memory, bandwidth, or cost).
3. Plans the refresh / retirement path so the fleet can absorb a new
   generation without a capacity cliff on the old one.

The reference solution below delivers an artifact triplet: a normalisation
table, a routing policy table, and a refresh plan. The specific per-SKU
numbers inside the normalisation table are **learner-supplied from the
exercise's local fixture or from the vendor spec sheet at the time the
plan is written**; this solution gives the *shape* of the artifact and
the *decision rules* a reviewer should apply.

## 2. Worked implementation — the three artifacts

### 2.1 Normalisation table (one row per SKU in the fleet)

The purpose of this table is to give a common language so a demand
forecast expressed in tokens/s can be mapped onto *any* generation. It is
**not** a benchmarking exercise and should not pretend to be — any single
tokens/s/GPU number is model-dependent and quant-dependent. Capture the
relative measurements from the team's own base-model runs (Exercise 01)
and annotate with the public spec that explains why the number is what it
is.

Required columns:

| Column | Meaning | Source |
|---|---|---|
| `sku` | e.g. `H100-SXM-80GB`, `H200-SXM-141GB`, `B200`, `GB200-NVL72`, `MI300X` | Inventory |
| `hbm_gb` | HBM capacity per device | Vendor spec |
| `hbm_bw_tbps` | HBM bandwidth per device | Vendor spec |
| `interconnect` | NVLink / Infinity-Fabric / PCIe bandwidth and topology | Vendor spec |
| `software_stack` | CUDA / ROCm version, kernel coverage for target model | Local measurement |
| `model_support` | List of models with validated inference recipes on this SKU | Local measurement |
| `μ_pf_rel` | Prefill tokens/s **relative to H100 = 1.0** on the team's reference model | Measured; Exercise 01 |
| `μ_dec_rel` | Decode tokens/s **relative to H100 = 1.0** on the team's reference model, at the team's max safe `B` | Measured; Exercise 01 |
| `max_B` | Max continuous-batch `B` that fits in HBM at the team's quant | Measured |
| `$/h` | Cost per device-hour to the team (reserved / on-demand / internal cross-charge) | Finance input |
| `lead_time_weeks` | How long it takes to add one more unit of this SKU | Supply chain |
| `power_kw` | Device power draw (plus accelerator carrier where relevant) | Vendor spec + facilities |

Two rules the table must enforce:

1. **Relative, not absolute, throughput.** `μ_pf_rel` and `μ_dec_rel`
   anchor to H100 = 1.0. This keeps the plan honest when vendor marketing
   quotes peak FLOPs that the team's actual kernels do not realise, and it
   lets the plan survive a model change because only the anchor row needs
   to be re-measured.
2. **Model-conditional support column.** Not every model has a tuned
   inference recipe on every SKU on day one. A new generation with
   `μ_dec_rel = 3.0` is worth nothing to a model with no supported
   runtime image on that SKU. The `model_support` column makes that
   explicit so the routing policy can't accidentally route to an
   unsupported pool.

### 2.2 Routing policy — which workload class goes to which generation

Capacity planning across generations is only useful if the demand
forecast can be *steered*. The routing policy is a table from workload
class to a preferred-SKU list, driven by the stage that constrains the
class (see Exercise 01).

| Workload class | Constraint | Preferred SKU rank | Rationale |
|---|---|---|---|
| Long-context RAG (prefill-bound) | Compute | newest generation first (e.g. B200/GB200 > H200 > H100 > MI300X) — subject to model-support column | Prefill wins most from raw compute uplift |
| Very-long-output agent (decode-bound) | HBM bandwidth | HBM-bandwidth-leading SKUs first (e.g. GB200 > B200 > H200 > H100), MI300X placed by measured bandwidth relative to H100 | Decode is bandwidth-bound, not FLOPs-bound |
| Very-long-context single-stream | HBM capacity | Highest `hbm_gb` first (e.g. H200, MI300X, GB200) | KV cache fits on-device at higher batch |
| Batch / background | Cost | Lowest `$/h` per effective tok/s first (depends on local cost table) | Latency budget is loose; price wins |
| Second-source for availability | Diversity | Hold ≥X% of total capacity on MI300X (or a non-Nvidia alternative) regardless of price | Resilience to a vendor-side shortage or roadmap slip |

The policy is written as a *preference order*, not a hard binding, so the
scheduler (or admission controller) can fall back when the preferred pool
is saturated. The reviewer should check that the policy admits
fall-through and does not hard-pin a class to one generation only.

### 2.3 Refresh plan — rolling a new generation in without a cliff

The failure mode a cross-generation plan must prevent is:

- Older generation is being retired.
- Newer generation lands later than quoted (`lead_time_weeks` was
  optimistic) and with gaps in `model_support`.
- Demand forecast keeps rising.
- No contingency.

Mitigation is a three-stage refresh contract:

1. **Overlap window.** Keep the older generation live for a minimum of
   `max(retirement_lead_time, model_support_validation_time)` *past* the
   first production traffic on the new generation. Never schedule the
   retirement before validation completes.
2. **Shadow + gradient roll.** Route a small percentage of production
   traffic to the new generation (shadow eval first, then gradient from
   1% → 10% → 50% → 100%). The gating metric at each step is the
   workload class's SLO (TTFT, TPOT, error rate), not a synthetic
   benchmark.
3. **Rollback contract.** Capacity on the old generation stays
   reservable for the full overlap window so a failed gradient step can
   reverse without a scale-out delay.

The refresh plan carries dates in weeks relative to the first new-SKU
install (W+0). A canonical example:

| Milestone | Week | Action |
|---|---|---|
| W+0 | 0 | First new-SKU rack online; software stack frozen for the plan |
| W+2 | 2 | Shadow eval on new SKU complete; `μ_pf_rel`, `μ_dec_rel`, `max_B` measured and added to normalisation table |
| W+4 | 4 | 1% production traffic on the new SKU behind the SLO gate |
| W+8 | 8 | 10% production traffic; parity validated against the old-SKU SLOs |
| W+12 | 12 | 50% production traffic; begin ramping down old-SKU reservations |
| W+16 | 16 | 100% production traffic; old SKU in standby for rollback |
| W+20 | 20 | Earliest safe retirement of old SKU (contingent on no open rollback incidents) |

The dates compress or stretch based on `model_support`, `lead_time_weeks`,
and incident history. The *shape* — shadow, gradient, overlap, retirement
gate — is what the plan must contain.

### 2.4 Example rollup (how the three artifacts compose)

Assume a forecast from Exercise 02 gives `N = 65` H100-equivalent replicas
at the quarter horizon, decomposed as:

- `chat_stream` (decode-bound, interactive): 15 H100-equivalent replicas
- `rag_qa` (prefill-bound, interactive): 30 H100-equivalent replicas
- `batch_summary` (either, batch): 20 H100-equivalent replicas

Suppose the normalisation table carries (measured by the team's base
model) `μ_pf_rel = 1.0 / 1.4 / 2.2 / 2.6 / 0.9` and
`μ_dec_rel = 1.0 / 1.3 / 1.8 / 2.4 / 1.0` across
`H100 / H200 / B200 / GB200 / MI300X`.

Routing + normalisation:

- `rag_qa` → prefer B200 (or GB200). 30 / 2.2 ≈ 14 B200 devices.
- `chat_stream` → prefer GB200 (highest `μ_dec_rel`). 15 / 2.4 ≈ 7 GB200.
- `batch_summary` → prefer lowest-cost per effective tok/s. If local
  finance says MI300X is cheapest per decode-tok/s, 20 / 1.0 ≈ 20 MI300X.

The plan now has three concrete acquisition line items with lead times
and a refresh milestone table for the H100 pool being drawn down.

## 3. Validation steps

1. **Every SKU row is traceable.** `μ_pf_rel` and `μ_dec_rel` must cite
   a measurement artifact (Exercise 01 run, internal benchmark report,
   dated kernel build). No cell in the normalisation table is allowed to
   be an unsourced number.
2. **Model-support column is populated and honoured.** A routing policy
   that sends a model to an SKU listed as unsupported for that model is
   wrong; a reviewer should check the policy against the support column.
3. **Routing policy admits fall-through.** Every workload class has at
   least two SKUs in its preference list. A policy with a single-SKU
   binding fails resilience; cite a credible failure mode where the
   single-SKU binding breaks (vendor supply, kernel regression, HBM
   cooling fault).
4. **Overlap window is longer than lead time.** The retirement date of
   the old SKU is at least `lead_time_weeks` of the new SKU past the
   100%-rollout date, with explicit rollback capacity reserved.
5. **Second-source diversity floor.** At least one workload class has a
   non-primary-vendor SKU as a preferred option. The plan states the
   floor as a percentage of total capacity, not "best effort".
6. **Power / cooling cross-check.** Power per rack and inlet cooling
   capacity are checked against facilities limits before an acquisition
   line item is committed. A plan that lists GB200 on a rack whose power
   budget cannot support it is a facilities failure in disguise.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | Normalisation table with the required columns, numbers anchored
      relative to H100 | 2 |
| 2 | `μ_pf_rel`, `μ_dec_rel`, `max_B` cite a measurement, not a spec
      sheet | 1 |
| 3 | Routing policy maps each workload class to the stage it is
      constrained by, with ≥2 SKUs per class | 2 |
| 4 | `model_support` column is populated and honoured in the routing
      policy | 1 |
| 5 | Refresh plan uses shadow → gradient → overlap → retirement with
      rollback capacity reserved | 2 |
| 6 | Second-source / vendor-diversity floor stated as a percentage of
      total capacity | 1 |
| 7 | Facilities constraints (power, cooling, rack space) cross-checked | 1 |

Deductions:

- Any cell in the normalisation table that quotes a vendor peak-FLOPs
  number as `μ_pf_rel`: −2 (peak FLOPs is not kernel throughput).
- Hard-binding a workload class to a single SKU: −2.
- Retirement scheduled before 100%-rollout on the new SKU: −3.
- No rollback capacity reserved: −2.

## 5. Common mistakes

- **Using vendor peak FLOPs as the normalisation currency.** Kernel
  realisation varies by model, quantisation, and context length. Measure
  it on the team's own model or the plan is a brochure.
- **Routing a model to a generation that lacks a supported runtime
  image.** The `model_support` column exists specifically to catch this.
- **Over-fitting to the newest generation.** Newest generation often has
  the thinnest supply and the least-mature kernels. Keeping some capacity
  on a prior generation is hedging, not waste.
- **Ignoring HBM bandwidth as its own axis.** Decode is bandwidth-bound;
  a generation with more compute but similar HBM bandwidth per device is
  not a big decode uplift.
- **Treating MI300X / AMD as a drop-in replacement.** ROCm kernel coverage
  and tooling are not identical to CUDA; the plan must reflect what the
  team has actually validated.
- **No facilities step.** A rack-level power budget or cooling inlet can
  make an on-paper acquisition impossible. Check before you commit.
- **Retiring the old SKU on the same date the new SKU reaches 100%.**
  The overlap window is the whole point; without it a rollback fails.
- **Ignoring the finance line.** `$/h` and `lead_time_weeks` live in the
  same table as throughput for a reason: the right generation for a
  workload is not always the fastest one.

## 6. References

- NIST AI Risk Management Framework (AI RMF 1.0) — Manage function
  (resource allocation, supplier risk, deployment governance) —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- Google SRE book — *Capacity Planning* chapter: lead time, provisioning
  signals, and the risk of single-source dependencies —
  <https://sre.google/sre-book/>.
- OWASP ML Security Top 10 and MITRE ATLAS inform the second-source /
  supplier-diversity dimension of the plan (supply-chain and
  availability threats that justify vendor diversity as a reliability
  floor, not a cost-saving option) —
  <https://owasp.org/www-project-machine-learning-security-top-10/> and
  <https://atlas.mitre.org/>.
- Vendor specifications (Nvidia H100, H200, B200, GB200 product briefs;
  AMD Instinct MI300X product brief) are the correct source for the
  `hbm_gb`, `hbm_bw_tbps`, `interconnect`, and `power_kw` columns. They
  are **not** the correct source for `μ_pf_rel` or `μ_dec_rel`, which
  must be measured on the team's own model.
- Local exercise context: this plan consumes `μ_pf`, `μ_dec`, and `B`
  from Exercise 01 and feeds the demand forecast from Exercise 02 into
  acquisition line items. It is validated empirically by the load-test
  harness in Exercise 06.
