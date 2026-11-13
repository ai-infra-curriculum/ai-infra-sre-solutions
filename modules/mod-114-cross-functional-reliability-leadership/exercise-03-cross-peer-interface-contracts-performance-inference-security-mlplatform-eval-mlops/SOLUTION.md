# Exercise 03 — Cross-Peer Interface Contracts (Performance, Inference, Security, MLPlatform, Eval, MLOps) — Reference Solution

Paired learning file: `lessons/mod-114-cross-functional-reliability-leadership/exercises/exercise-03-cross-peer-interface-contracts-performance-inference-security-mlplatform-eval-mlops.md`.

> **Scope note.** The paired exercise asks the learner to author the written contracts that define *how AI-SRE works with its peer disciplines*. "Peer" here is used in the sense of the AI-infrastructure track — adjacent engineering roles that share an interface with AI-SRE: Inference-Performance Engineering, ML-Platform Engineering, Eval / Applied-AI Engineering, MLOps, and Security / AI Safety. The deliverable is a *set* of contracts: one per peer, plus a peer-contract index and a quarterly review ritual. The reference solution uses the illustrative `chatcat` product used throughout this module.

## 1. Solution overview

Any one of these peers can block, unblock, or derail a release of a model-backed service. In the absence of written contracts, three failure modes dominate:

1. **Ownership gaps.** A failure that is 55 % model-quality and 45 % latency falls between the eval owner and the inference-performance owner; neither picks it up until the SLO is already blown.
2. **Ownership overlaps.** A security review and a reliability gate both block on the same issue, with different thresholds, and the author of the rollout has no way to resolve the conflict without a senior tiebreaker.
3. **Interface drift.** The peer that owns a surface changes it (a new eval metric, a new tenant isolation boundary, a new feature-store schema) without a written contract, and the SRE team discovers the change through a post-incident post-mortem.

The reference solution is a set of **peer interface contracts**, one per peer, each with the same shape: what the peer owns, what the peer produces, what the peer consumes from SRE, SLO and error-budget interaction, veto and appeal rules, change-management rules, and the escalation path. The contracts live in-repo (`peer-contracts/`) and are reviewed quarterly (ritual defined in exercise 01).

### 1.1 Design principles

- **Every surface has exactly one owner.** No surface has two owners; no surface has zero owners. The matrix in §2.1 is the normalising artefact.
- **Signals cross the boundary; implementations do not.** The contract names *what* a peer produces (a metric, a verdict, an artefact) and *where it is emitted*, not *how* the peer produces it. This keeps contracts durable under internal peer re-orgs.
- **Freshness is part of every contract.** A signal older than its `max_staleness` is a `fail`, not a `pass`. This mirrors the release-gate contract in `mod-111` exercise 05.
- **Veto is scoped, not global.** Each peer may veto within its surface class. A contract names which signal classes the peer may veto and which may be overridden by appeal.
- **Appeals are written, not spoken.** Every appeal is logged; no appeal overturns a contract silently.
- **Change management is bilateral.** A peer may not change its produced-signal schema without a written PR against this contract. An SRE-side threshold tightening that would newly fail a peer's signal requires a similarly written change.

### 1.2 Surface ownership matrix

| Surface | Owner | SRE role |
|---|---|---|
| Base model serving binary (runtime, sampling params, batching) | Inference-Performance | Consumer; SRE owns the SLO on latency and error-rate downstream |
| GPU / accelerator reservation, quota, bin-packing | ML Platform | Consumer; SRE owns utilisation and burst policy |
| Feature store, retrieval index | ML Platform / MLOps | Consumer; SRE owns freshness SLO on the index |
| Eval harness, offline eval signals | Eval | Consumer; SRE owns freshness and drift SLO on eval |
| Shadow-traffic pipeline, canary analysis | Eval + SRE (co-owned) | Co-owner; split defined in `mod-111` exercise 03 |
| Prompt-template registry, version pinning | MLOps + Eval | Consumer; SRE owns release-ledger linkage |
| Safety classifier / guardrail sidecar | Security / AI Safety | Consumer; SRE owns the latency and error SLO on the sidecar |
| Data-residency boundary, PII handling in logs | Security | Consumer; SRE owns the incident-response path when this is violated |
| SLO definition, error-budget policy | SRE | Owner |
| Release ledger, incident ledger, SLO ledger | SRE | Owner |
| Capacity plan and procurement ask | SRE + ML Platform (co-owned) | Co-owner; capacity model in `mod-106` |

Every peer contract below instantiates one row (or several related rows) of this matrix.

## 2. Implementation — worked answers

Each peer contract lives at `peer-contracts/<peer>.md`. All five contracts share a fixed section order; the contracts below are complete worked answers for two of them (Inference-Performance and Eval), followed by summaries for the remaining three. A learner may author any subset; a passing submission covers all five.

### 2.1 Contract — Inference-Performance Engineering (`peer-contracts/inference-performance.md`)

```markdown
# Contract — Inference-Performance Engineering

**Owner (peer side).** Inference-Performance Engineering lead.
**Owner (SRE side).** AI-SRE function lead.
**Last reviewed.** YYYY-MM-DD.
**Next review.** Quarterly, per exercise 01 cadence.

## Scope

Inference-Performance owns the model serving binary and its performance
characteristics: runtime choice (vLLM, TGI, SGLang, TensorRT-LLM), batching
strategy (continuous, static, speculative), kv-cache strategy, sampling
parameter ranges, and the per-request compute envelope. They *do not* own
the SLO on the service, the alerting, or the release gate.

## Signals produced by Inference-Performance

Posted to a well-known Prometheus namespace; schemas are versioned in
`peer-contracts/signals/inference-performance.schema.json`.

| Signal | Unit | max_staleness | Description |
|---|---|---|---|
| `infperf_ttft_seconds_p95_5m` | seconds | 180s | p95 time-to-first-token |
| `infperf_tpot_seconds_p95_5m` | seconds | 180s | p95 time-per-output-token |
| `infperf_queue_depth_mean_5m` | count | 180s | Mean per-replica queue depth |
| `infperf_kv_cache_util_mean_5m` | ratio | 300s | Mean KV cache utilisation |
| `infperf_batch_size_mean_5m` | count | 300s | Mean effective batch size |
| `infperf_evict_rate_5m` | /sec | 300s | Request eviction rate due to kv-cache pressure |

Each signal has a documented formula and the Prometheus query that produces
it. Signal removal or rename requires a 30-day deprecation PR against this
contract.

## Signals consumed from SRE

| Signal | Why Inference-Performance needs it |
|---|---|
| `slo_error_rate_5m` by route | To know whether a batching change caused upstream errors |
| `slo_ttft_p95_28d` by route | To size the acceptable delta on their own experiments |
| `release_ledger entries by service` | To correlate a performance regression with a specific release |

## Interface with the release gate (`mod-111` exercise 05)

Inference-Performance signals back the SLO-side of the gate. A proposed
runtime or batching change that is expected to improve TTFT ≥ 10 % is
promoted to canary via the standard rollout flow. The gate uses
`infperf_ttft_seconds_p95_5m` as one of its SLO-owned signals per the
chatcat gate contract.

## Veto authority

- Inference-Performance may *veto* a release that would require a runtime
  version it has not certified (e.g. a vLLM major-version bump). The veto
  is per release, not per route.
- Inference-Performance may *not* veto an eval-driven release (a prompt
  change, a model-version change) merely because it would change latency.
  They may raise a documented concern, and the SLO owner decides whether to
  tighten the gate before the release.
- SRE may *veto* a performance change that would consume > 50 % of an
  active error budget. The veto is in writing; it cannot be overridden
  except by appeal to the SRE function lead.

## Appeal path

Written appeal to the SRE function lead; three business days to decide;
gate remains in its pre-appeal state during the appeal.

## Change management

- New Inference-Performance signals: PR against this contract. Must
  include unit, formula, Prometheus query, max_staleness, and the SLO (if
  any) that will consume it.
- Signal rename or removal: 30-day deprecation PR.
- Threshold changes on consumed SRE signals: no change on SRE's side; the
  SLO owner can tighten or loosen without peer consent, but changes are
  logged and reviewed at the quarterly peer-contract review.

## Oncall interaction

- The on-call for the week carries Inference-Performance on the escalation
  tree. For a p95 TTFT burn that is suspected runtime-caused, Inference-
  Performance is paged within 15 minutes.
- Inference-Performance maintains their own on-call rotation for GPU
  kernel / runtime issues. SRE does not page their on-call for incidents
  that are not runtime-caused.

## SLO interaction

- Inference-Performance signals *are* used in SLOs owned by SRE (e.g. the
  TTFT SLO). SRE owns the SLO definition; Inference-Performance owns the
  signal.
- Changing the SLO target requires joint sign-off (SLO owner + Inference-
  Performance lead). The default cadence for this is the quarterly review.

## Escalation path

SRE function lead → Inference-Performance lead → engineering director.
No surface in this contract is escalated past engineering director; any
dispute that reaches director is a sign the contract needs revision.

## Related contracts and exercises

- `peer-contracts/ml-platform.md` — GPU reservations feed Inference-
  Performance's runtime.
- `peer-contracts/eval.md` — Eval's acceptance of a runtime change.
- `mod-104` observability contract — Prometheus namespace ownership.
- `mod-111` exercise 05 — release-gate contract.
```

### 2.2 Contract — Eval / Applied-AI Engineering (`peer-contracts/eval.md`)

```markdown
# Contract — Eval / Applied-AI Engineering

**Owner (peer side).** Eval lead.
**Owner (SRE side).** AI-SRE function lead.

## Scope

Eval owns the correctness, groundedness, refusal, safety, and structured-
output signals for `chatcat`. They own the offline eval harness and the
online sampling / instrumentation that produces eval signals in production.
They *do not* own SLO targets, incident response, or the release pipeline.

## Signals produced by Eval

Schema in `peer-contracts/signals/eval.schema.json`.

| Signal | Unit | max_staleness | Description |
|---|---|---|---|
| `eval_groundedness_mean_1h` | score | 1h | Hourly groundedness, sampled set |
| `eval_groundedness_delta` | delta | 1h | Canary – baseline groundedness |
| `eval_refusal_rate_1h` | ratio | 1h | Refusal rate on sampled prompts |
| `eval_refusal_delta` | delta | 1h | Canary – baseline refusal rate |
| `eval_structured_valid_rate_1h` | ratio | 1h | JSON / tool-call validity |
| `eval_structured_exact_match_rate_1h` | ratio | 1h | Exact match on structured routes |
| `eval_policy_safety_regressions_24h` | count | 2h | Safety regressions in last 24h |

## Signals consumed from SRE

| Signal | Why Eval needs it |
|---|---|
| `slo_error_rate_5m` by route | To know whether the model under eval was actually serving |
| `release_ledger entries by service` | To tag eval runs to a specific release |
| `shadow_traffic_sample_rate` | Operational signal; eval owns the analysis, SRE owns the pipe |

## Interface with the release gate

Eval signals are the eval-side of the gate contract in `mod-111` exercise
05. Threshold authorship for every `owner: eval` signal is Eval's; SRE
cannot unilaterally change those thresholds. The release gate treats stale
eval signals as `fail` (not `pass`); Eval is accountable for freshness of
every signal they produce.

## Veto authority

- Eval may veto a release on any `owner: eval` signal regression per the
  gate contract.
- Eval may NOT veto a release on an operational (SLO) concern. They may
  raise one; the SLO owner decides.
- SRE may *not* veto an eval-only decision (choice of benchmark, label
  policy, sampling rate), but may require the signal to meet the
  freshness and schema requirements above. If an eval signal cannot meet
  the freshness requirement, it is removed from the gate.

## Appeal path

Written appeal to the SRE function lead for a gate veto; three business
days to decide; gate remains in its pre-appeal state during the appeal.

## Change management

- New eval signals: PR against this contract and the gate schema, with
  thresholds proposed per route class. Must include the sampling plan,
  label source, and freshness strategy.
- Signal rename or removal: 30-day deprecation PR.
- Threshold tightening: requires an incident link, a documented
  regression, or an explicit product decision (per the gate contract).
- Sampling-plan changes that reduce freshness: require SRE sign-off,
  because they weaken the gate.

## Oncall interaction

- Eval is paged for safety regressions (`eval_policy_safety_regressions_24h
  > 0`). The page escalates to the Eval lead within 30 minutes.
- SRE on-call is not paged for eval-only regressions (groundedness drift
  with no SLO impact). Those route to the eval team's slower-cadence
  review.

## SLO interaction

- "Eval SLOs" (groundedness, refusal, structured-valid) exist in the SLO
  ledger. Their numeric targets are co-owned: Eval sets the floor; SRE
  sets the burn-rate response.
- An eval-SLO breach that is sustained for more than two weekly reviews
  triggers a freeze on the affected route class per the error-budget
  policy.

## Escalation path

SRE function lead → Eval lead → engineering director.

## Related contracts and exercises

- `peer-contracts/mlops.md` — prompt and model-version registry.
- `mod-111` exercises 03 (shadow traffic), 05 (gate contract).
- `mod-104` observability — eval signal emission conventions.
```

### 2.3 Contract summaries — ML Platform, MLOps, Security / AI Safety

A full passing submission includes one file per peer. The three remaining contracts share the same template and sections; the key per-peer differences:

- **ML Platform (`peer-contracts/ml-platform.md`).** Owns GPU reservation, accelerator quota, feature store, retrieval index. Produces: `platform_gpu_reservation_utilisation`, `platform_gpu_available_minutes`, `platform_feature_store_freshness_seconds`, `platform_index_build_last_success_unix`. Veto: may veto any release that would require GPU reservations not pre-allocated; may veto any index-consuming change that would push index freshness past its SLO. Appeal path: SRE function lead. Change management: schema changes to the feature store require 30-day deprecation PR and the SRE-owned freshness SLO updated in the same PR. Interacts with capacity ritual (exercise 01 monthly) as a primary input.

- **MLOps (`peer-contracts/mlops.md`).** Owns prompt / model-version registry, training-pipeline artefacts, prompt-template deployment. Produces: `mlops_prompt_registry_last_mutation_unix`, `mlops_model_version_in_use`, `mlops_prompt_rollout_in_progress`. Veto: may veto a release whose prompt-template version is not pinned; may veto a release whose model-version pin conflicts with an active training-pipeline artifact. Appeal: SRE function lead. Change management: prompt-store schema changes are a 30-day deprecation PR; model-version pinning policy changes require joint sign-off with Eval. Interacts with release ledger via a mandatory `change_anchor` entry (see `mod-111` exercise 06).

- **Security / AI Safety (`peer-contracts/security.md`).** Owns safety classifier / guardrail sidecar, data-residency boundary, PII-in-logs policy, and the adversarial-ML threat register. Produces: `security_guardrail_decision_latency_seconds_p95`, `security_guardrail_error_rate_5m`, `security_classifier_regression_count_24h`, `security_pii_leak_alert_count_24h`. Veto: may veto any release that would newly admit PII into a non-compliant surface; may veto a release that lowers the guardrail decision rate; may *not* veto a routine release on a surface where guardrails are already green. Appeal path: SRE function lead + security lead jointly (security has a stronger veto than other peers because of regulatory stakes; this is called out explicitly in the contract). Change management: new guardrail classifier thresholds require a documented back-test and a 30-day shadow period before enforcing in the release gate. Interacts with `mod-110` exercises 01 (sidecar latency), 04 (reliability-gate vs safety-gate reconciliation), 05 (incident sensitive-data handling).

### 2.4 Peer-contract index — `peer-contracts/README.md`

```markdown
# Peer contracts — index

Each peer has one contract in this directory. The contracts share a fixed
section order (Scope / Signals produced / Signals consumed / Interface
with the release gate / Veto authority / Appeal path / Change management /
Oncall interaction / SLO interaction / Escalation path / Related contracts).
All contracts are reviewed quarterly at the Cross-Peer Contract Review
ritual (exercise 01).

| Peer | File | Last reviewed | Open issues |
|---|---|---|---|
| Inference-Performance | inference-performance.md | YYYY-MM-DD | — |
| ML Platform | ml-platform.md | YYYY-MM-DD | — |
| Eval / Applied-AI | eval.md | YYYY-MM-DD | — |
| MLOps | mlops.md | YYYY-MM-DD | — |
| Security / AI Safety | security.md | YYYY-MM-DD | — |

Contract version is the git SHA of the file. Any contract change is a PR
with the peer-side owner and the SRE-side owner both in the reviewers list.
```

## 3. Validation steps

1. **Ownership matrix coverage.** Every row in §1.2 has exactly one owner (or exactly one co-ownership edge with a split defined in the contract). A linter walks the matrix and fails on any row with zero or two owners without a defined split.
2. **Contract-file coverage.** One file per peer; each file instantiates the fixed template. A linter checks for section presence.
3. **Signal schema consistency.** Signals named in a peer contract must appear in `peer-contracts/signals/<peer>.schema.json`. Missing or extra signals produce a linter warning.
4. **Gate consistency.** Every peer signal consumed by the release gate (`mod-111` exercise 05 schema) must be named in the corresponding peer contract. The gate schema and the peer contracts share one source of truth.
5. **Veto-conflict detection.** For any surface in the ownership matrix, no two peers may claim veto authority without a written tiebreaker. A linter walks every peer contract's veto section and flags conflicts.
6. **Appeal-path consistency.** Every veto in every contract has an appeal path with a named decider, a bounded-time window, and a stated during-appeal state. Missing fields fail validation.
7. **Change-management tracing.** Simulate a signal rename from Inference-Performance. Confirm the 30-day deprecation PR process triggers, that the gate schema PR is linked, and that the Inference-Performance on-call is a required reviewer.
8. **Quarterly-review timestamping.** Every contract has a `Last reviewed` field; the quarterly peer-contract review (exercise 01) increments it. Contracts whose `Last reviewed` is older than one quarter trigger a program-retro note.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 18 / 24 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Ownership matrix *(hard gate)* | Missing or has gaps | Full coverage but no co-ownership splits | Full coverage + explicit co-ownership splits |
| 2 | Contract per peer *(hard gate)* | < 5 | 5, with gaps | 5, each with the fixed template |
| 3 | Signals produced schemas | Named in prose only | JSON Schema per peer | Schema + versioning + deprecation rule |
| 4 | Freshness per signal *(hard gate)* | Missing | Mentioned | `max_staleness` on every signal, enforced by gate |
| 5 | Veto authority scoped | Global vetoes | Scoped per peer | Scoped per peer + conflict-check linter |
| 6 | Appeal path | Missing | Prose "contact X" | Named decider + bounded window + state during appeal |
| 7 | Change management | None | Prose "do a PR" | 30-day deprecation + bilateral sign-off |
| 8 | Oncall interaction | Vague | Named for one peer | Named for every peer, with escalation time |
| 9 | SLO interaction | Missing | Prose only | Which signals feed which SLOs + who sets the target |
| 10 | Quarterly review ritual | Not referenced | Named | Named + default-off rule if contracts drift |
| 11 | Related contracts cross-linked | Not linked | Linked once | Linked across all five contracts + module exercises |
| 12 | Security peer has stronger veto, explicitly | Treated as identical | Named stronger | Named, with regulatory reasoning |

## 5. Common mistakes

- **One big contract.** Bundling all five peers into one document hides conflicts. One file per peer forces the differences to be written.
- **"Veto by whoever yells loudest."** Without a scoped-veto rule and a written appeal path, peer disputes default to seniority. The scoped-veto rule exists because it stops this.
- **No freshness on peer signals.** A peer signal that is 24 h old does not reflect the running service. The gate must treat stale peer signals as `fail`.
- **Security treated as a peer identical to Inference-Performance.** Security carries regulatory stakes that no other peer carries (incident-notification SLAs, data-residency boundaries, PII policies). The contract must say so explicitly; failing to do so leads to the first regulated-incident post-mortem writing it the hard way.
- **Co-ownership by handshake.** "Both teams own it" without a written split is one team owning it. The matrix forces the split.
- **Contract changes without 30-day deprecation.** A signal rename shipped the same week causes downstream alerts to go silent. 30 days is a minimum; some enterprises use 90.
- **Appeal path with no "during-appeal state".** If the contract does not say what the gate does during the appeal window, the appeal process becomes a de-facto approval. The contract names it.
- **Quarterly review that never happens.** Contracts drift fast. A review that is scheduled but never run is worse than none; the retro (exercise 01) must enforce it.

## 6. References

- Google SRE workbook, *On-Call* — [sre.google/workbook/on-call/](https://sre.google/workbook/on-call/) — the on-call interaction section of each peer contract adopts the paging-tree and primary/secondary split it recommends.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `GOVERN-2.1` (roles and responsibilities) and `GOVERN-5.1` (external actors and third-party relationships) frame the explicit ownership and veto matrix; `MANAGE-4.1` (ongoing monitoring) frames the freshness rule on every peer signal.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML05: Model Theft`, `ML06: ML Supply Chain Attacks`, and `ML08: Model Skewing` justify the Security contract's stronger veto on the guardrail and PII surfaces.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-ML threat catalogue that motivates security's regulatory-stakes framing and the mandatory back-test + 30-day shadow on new guardrail thresholds.
