# Exercise 01 — Top-Twenty AI Runbook Authoring — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-01-top-twenty-ai-runbook-authoring.md`.

## 1. Solution overview

The exercise asks for five production-ready runbooks drawn from the twenty described in chapter 01, plus a one-page cover note. The reference solution below authors the balanced default set — `rb-ttft-regression`, `rb-cache-hit-cliff`, `rb-hallucination-rate-spike`, `rb-provider-degraded-fallback`, and `rb-gpu-driver-ecc-xid` — against an **illustrative** product (`chatcat`, a RAG-chat surface backed by a hosted model with embeddings and moderation). The product is fictional; the signal names reference an OpenTelemetry + Prometheus + Grafana + Loki stack that is standard enough to be portable.

Each runbook uses the seven-section shape from chapter 01 §1 and a machine-readable YAML header from §4. Every detection signal is either plausibly present on the stack or marked `(proposed)` with a note in the cover-note gap list. The mitigation menu is ranked by blast radius and reversibility; the handoff is by role, not name.

Reviewers of a learner submission should use the rubric in §4 below. A rubric score ≥ 15/20 with no "0" categories corresponds to a usable, environment-grounded fragment of the twenty-runbook library.

## 2. Implementation — worked answer

### 2.1 Product and observability context (one-page cover note)

```markdown
# Cover note — chatcat runbook library fragment

**Product.** `chatcat`, a RAG-chat surface for a mid-sized consumer product. Traffic is served
by a hosted model (primary: `gpt-4o-mini-2024-07-18`; fallback: `claude-3-5-haiku-20241022`),
with an in-house embedding pipeline (BGE-base-en-v1.5) feeding a Pinecone index, and a safety
filter chain in front of the model output.

**Observability stack.**
- Metrics: Prometheus scraped from vLLM 0.6.x, DCGM exporter, and a custom `chatcat-telemetry`
  sidecar that emits `chatcat_*` prefixed signals.
- Traces: OpenTelemetry with `genai.*` semantic conventions, forwarded to Honeycomb.
- Logs: Loki with structured JSON; sample traces pinned to the incident channel by trace ID.
- Dashboards: Grafana, with a canonical "chatcat / serving-health" board and a parallel
  "chatcat / quality-health" board.
- Judge pipeline: nightly against `evals/chatcat-canary` plus hourly groundedness checks on a
  sampled stream.

**Why these five cover the three categories.**
- Serving-path: `rb-ttft-regression` and `rb-cache-hit-cliff`. TTFT is the most-paged class on
  this surface, and cache-hit cliffs are the most common *hidden* cause of TTFT pages.
- Quality-path: `rb-hallucination-rate-spike`. The judge pipeline is the only source of signal
  for this class; authoring the runbook first forced us to notice that the pipeline runs
  hourly, not continuously, and lives in the cover note as a gap.
- Infrastructure-path: `rb-provider-degraded-fallback` (vendor surface) and
  `rb-gpu-driver-ecc-xid` (hardware surface) — the two most distinct infra-path classes.

**Gaps and next runbooks.** We would write `rb-tpot-regression`, `rb-tool-call-cascade`,
`rb-structured-output-break`, `rb-weight-corruption`, and `rb-cost-blow-up` next. The observability
gaps that block them today are: per-session cost accounting, a structured-output validator error
counter, and a model-weight checksum attestation pipeline. All three are in the SLO-stack
backlog.

**Introduction plan.** The five runbooks are introduced as a *floor*, not a replacement for
the existing on-call pages. Existing runbooks for infra classes (`host-down`, `disk-full`) stay
untouched. The runbook index gains an "AI-specific" section with a one-line symptom per entry,
and the alert YAML is updated so every AI-specific alert carries the matching `runbook_id`.
```

### 2.2 `rb-ttft-regression` — Time-to-first-token regression

```yaml
---
runbook_id: rb-ttft-regression
title: "TTFT regression on chatcat serving"
symptom: "chatcat_ttft_seconds p95 is >= 2x the 7-day baseline on a route for >= 10m"
default_severity: SEV-3
severity_ladder:
  SEV-3: "single-route, no SLO burn"
  SEV-2: "1h SLO burn-rate alert fires on the chatcat-latency SLO"
  SEV-1: "route is a top-5 revenue surface AND the 5m SLO burn-rate alert fires"
peer_owner: ai-infra-inference-engineer-oncall
related_runbooks: [rb-cache-hit-cliff, rb-tpot-regression, rb-provider-degraded-fallback]
mod_105_class: serving-latency
fixture_backfeed_hints: [alert-threshold, canary-prompt, load-shape]
last_reviewed: 2026-11-01
---
```

**Detection signals.**

- `histogram_quantile(0.95, sum by (le,route,model) (rate(chatcat_ttft_seconds_bucket[5m])))`
  on dashboard `chatcat/serving-health` panel `TTFT p95 per route`. Expected lag from root
  cause to signal: 1–3 min (metric is 15s scrape, 5m window).
- `chatcat_prompt_cache_hit_ratio` on the same route (`chatcat/serving-health` panel
  `Prompt cache hit ratio`). Expected lag: 1 min.
- `vllm:prefill_queue_depth{route="<route>"}`. Expected lag: 30s.
- `DCGM_FI_DEV_GPU_UTIL{pool="chatcat-serving"}`, breaking down prefill vs. decode via
  `vllm:gpu_phase`. Expected lag: 1 min.
- `genai.request.model` span attribute change in the last 30 min (Honeycomb saved query
  `chatcat / model-version-switches`).

**Confirm checklist.**

1. **Scope** — is the regression on one route, one model, one replica group, one zone, or
   fleet-wide? Consult Grafana `chatcat/serving-health` and Honeycomb `genai.request.route`
   breakdown.
2. **Correlated change** — did anything deploy in the last 60 min? Check ArgoCD `chatcat-*`
   apps, LaunchDarkly flag audit, and Honeycomb `genai.request.model` version counter.
3. **Prompt-cache** — is `chatcat_prompt_cache_hit_ratio` within ±5 pp of its 7-day baseline on
   this route? If not, swap to `rb-cache-hit-cliff`.
4. **Arrival shape** — did request rate or prompt-length distribution shift? Consult the
   `chatcat / request-arrival` Honeycomb board.
5. **Upstream** — status pages for the hosted model and embedding providers (via the
   `vendor-status` ingestor).
6. **Downstream** — support-ticket volume on "slow responses" tag in Zendesk (dashboard panel
   `chatcat / support-correlation`).

**Mitigation menu (ranked).**

| # | Mitigation | Blast radius | Reversibility | Pre-conditions | Time-to-effect |
|---|------------|--------------|---------------|----------------|-----------------|
| 1 | Open `chatcat-serving` router breaker; fall through to `claude-3-5-haiku` via the fallback contract | User-facing (the fallback has a slightly different voice) | One-command (`chatcat-ctl breaker open`) | Fallback canary eval green in the last 24h | 30s |
| 2 | Lower prefill-batch cap from 128 → 64 via vLLM runtime flag | Dev-facing (reduces throughput by ~15%) | One-command | None | 1–2 min |
| 3 | Shed background-chat traffic (`chatcat-batch-*` jobs) via admission-policy flag | Dev-facing | One-command | None | 30s |
| 4 | Roll back the most recent `chatcat-serving` deploy | Fleet-wide | Coordinated unwind (requires eval sign-off on prior version) | Prior version's canary eval score still green | 5–8 min |
| 5 | Pin requests to the healthy replica group / zone via routing weight | User-facing (regional scope) | One-command | At least one healthy group identified in step 1 | 1 min |

**Handoff.** Page `ai-infra-inference-engineer-oncall` (Pagerduty escalation `chatcat-inference`).
Structured payload:

```json
{
  "incident_id": "<channel-id>",
  "runbook": "rb-ttft-regression",
  "window": {"start": "<utc>", "end": "<utc-or-ongoing>"},
  "scope": {"route": "<route>", "model": "<model-id>", "replicas": ["<rg-a>", "<rg-b>"]},
  "confirm_checklist_state": {"1": "...", "2": "...", "3": "...", "4": "...", "5": "...", "6": "..."},
  "mitigations_applied": [{"name": "...", "at": "<utc>", "effect": "..."}],
  "next_decision": "..."
}
```

Escalate to SEV-2 if the 1h SLO burn-rate alert fires; SEV-1 if the route is a top-5 revenue
surface and the 5m burn-rate alert fires.

**User communication template.** "Some chat responses may begin more slowly than usual. We have
reduced the impact and are investigating. — [status: investigating | monitoring | resolved]."
No vendor name until the vendor has acknowledged.

**Fixture back-feed hint.** Candidates: (a) a threshold alert on `chatcat_prompt_cache_hit_ratio`
if cache was the root cause; (b) a load-shape in the load-test harness that reproduces the
batch-size regression; (c) a canary prompt added to the hourly set that exercises the slow path.
Decide per chapter 05 §8's decision tree.

### 2.3 `rb-cache-hit-cliff` — Prompt-cache hit-rate cliff

```yaml
---
runbook_id: rb-cache-hit-cliff
title: "Prompt-cache hit-rate cliff on chatcat"
symptom: "chatcat_prompt_cache_hit_ratio on a route dropped >= 20pp over <= 30m"
default_severity: SEV-3
severity_ladder:
  SEV-3: "single route, no adjacent SLO burn"
  SEV-2: "shared cache: multiple routes regressed OR adjacent TTFT regression"
  SEV-1: "cliff is co-incident with a cost blow-up on the same route"
peer_owner: ai-infra-inference-engineer-oncall
secondary_peer_owner: ai-engineering-oncall  # for prompt-template-related cliffs
related_runbooks: [rb-ttft-regression, rb-cost-blow-up]
mod_105_class: serving-cache
fixture_backfeed_hints: [canary-stable-prefix, ci-check-template-hash]
last_reviewed: 2026-11-01
---
```

**Detection signals.**

- `chatcat_prompt_cache_hit_ratio{route="<route>", model="<model>"}` on
  `chatcat/serving-health` panel `Prompt cache hit ratio`. Expected lag: 1 min.
- `chatcat_prompt_template_version{route="<route>"}` — a counter that increments on every
  template deploy. Expected lag: instant.
- `chatcat_tokenizer_version{model="<model>"}` — expected lag: instant.
- `chatcat_prompt_prefix_length_bytes` histogram. Expected lag: 1 min.
- `chatcat_prompt_cache_evictions_total` — expected lag: 1 min.

**Confirm checklist.**

1. **Template change** — did the system prompt or scaffolding change in the window? Consult
   ArgoCD `chatcat-app` revisions and the prompt-store git log.
2. **Model / tokenizer change** — did the model ID or tokenizer version move? Consult
   `genai.request.model` and `chatcat_tokenizer_version` counters.
3. **Request shape** — new leading field (user locale, A/B flag) injected into the prompt?
   Diff a sample trace against a pre-cliff trace in Honeycomb.
4. **Cache pressure** — is `chatcat_prompt_cache_evictions_total` elevated (cache memory
   budget full)?
5. **New tenant** — did a new workload start sharing the cache (check tenant-ID distribution
   on `chatcat_prompt_cache_queries_total`)?

**Mitigation menu (ranked).**

| # | Mitigation | Blast radius | Reversibility | Pre-conditions | Time-to-effect |
|---|------------|--------------|---------------|----------------|-----------------|
| 1 | Roll back the template change | User-facing (quality may regress to prior) | Coordinated (template deploy via `chatcat-prompts` repo PR revert) | Prior template green on last nightly eval | 3–5 min |
| 2 | Pin template version on the affected route | User-facing (prior behaviour on this route) | One-command | Known-good version id | 1 min |
| 3 | Raise cache memory budget (vLLM flag) or shard cache per route | Dev-facing | Coordinated (requires serving restart on affected pool) | Capacity headroom on the serving pool | 5–10 min |
| 4 | Accept the cost / latency; shift route to the fallback provider's cache | User-facing (tone drift) | One-command (`chatcat-ctl breaker open`) | Fallback canary eval green | 30s |

**Handoff.** `ai-infra-inference-engineer-oncall` and `ai-engineering-oncall` jointly. Payload
shape mirrors `rb-ttft-regression`; add the suspected template commit SHA.

**User communication template.** Usually silent (symptom surfaces as TTFT regression covered
by `rb-ttft-regression`'s template).

**Fixture back-feed hint.** Candidates: (a) a canary that tests the stable-prefix hash against
the cache metric; (b) a CI check in the `chatcat-prompts` repo that blocks template PRs that
move the prefix hash without a corresponding cache-warmup plan.

### 2.4 `rb-hallucination-rate-spike` — Groundedness regression

```yaml
---
runbook_id: rb-hallucination-rate-spike
title: "Hallucination / groundedness regression on chatcat"
symptom: "Judge pipeline groundedness score on a route moved >= 2 sigma worse from the 7-day baseline for >= 1h with no corresponding product-change event"
default_severity: SEV-2
severity_ladder:
  SEV-2: "default; eval signal confirmed on sampled stream"
  SEV-1: "route touches a regulated domain OR customer-facing content is auto-published OR a safety-adjacent single-sample is reported"
peer_owner: ai-eval-engineer-oncall
secondary_peer_owner: ai-engineering-oncall
related_runbooks: [rb-judge-model-drift, rb-provider-degraded-fallback, rb-retrieval-index-staleness]
mod_105_class: quality-hallucination
fixture_backfeed_hints: [canary-eval-case, judge-drift-alert, publish-gate]
last_reviewed: 2026-11-01
---
```

**Detection signals.**

- `chatcat_groundedness_score{route="<route>"}` — judge-pipeline nightly batch plus
  hourly-sampled stream. Dashboard `chatcat/quality-health` panel `Groundedness per route`.
  Expected lag: up to 1h for the stream signal, up to 24h for the batch signal (this lag
  is a gap called out in the cover note).
- `chatcat_retrieval_recall_at_k{route="<route>", k="5"}` on the same board. Expected lag:
  5 min.
- `chatcat_refusal_rate{route="<route>"}` — inversely correlated with groundedness in several
  past incidents. Expected lag: 5 min.
- Support-ticket volume on the "incorrect answer" tag in Zendesk. Expected lag: 1–4h.
- `genai.request.model` Honeycomb breakdown — silent model-version changes on the route.
  Expected lag: instant.

**Confirm checklist.**

1. **Model change** — did the provider route or model-version change (`genai.request.model`
   diff)? If yes, consider `rb-provider-degraded-fallback` as the real runbook.
2. **Retrieval change** — did the index refresh, embedding model, chunking, or metadata
   filter change? Consult the `chatcat-pipeline` deploy audit.
3. **Prompt change** — did the system prompt or retrieval-prompt template change?
4. **Judge sanity** — is the judge model itself behaving normally? Run the judge-sanity
   canary; if it fails, swap to `rb-judge-model-drift` — the regression may not be real.
5. **Input distribution** — did a new cohort, locale, or workload start on this route
   (trace-attribute breakdown by `user.cohort`)?

**Mitigation menu (ranked).**

| # | Mitigation | Blast radius | Reversibility | Pre-conditions | Time-to-effect |
|---|------------|--------------|---------------|----------------|-----------------|
| 1 | Route off the offending model to the fallback | User-facing (voice drift) | One-command | Fallback canary green; fallback groundedness >= primary | 30s |
| 2 | Roll back the retrieval change (index version pin, chunking revert) | User-facing | Coordinated (index re-pin via `chatcat-pipeline` tool) | Prior index snapshot retained | 10–20 min |
| 3 | Roll back the prompt template | User-facing | Coordinated (PR revert) | Prior template green on last eval | 3–5 min |
| 4 | Raise the refusal / uncertainty threshold on the route | User-facing (reduced coverage) | One-command | Communicate to product that coverage is reduced | 1 min |
| 5 | Suspend auto-publish workflows that consume this route unmoderated | Dev-facing | One-command (feature flag) | Downstream auto-publish exists | 30s |

**Handoff.** Page `ai-eval-engineer-oncall` for diagnosis and `ai-engineering-oncall` for prompt
/ retrieval ownership. If the regression touches a regulated domain or an auto-publish path,
also page `ai-risk-engineer-oncall` and legal per chapter 06 §3.1.

**User communication template.** "We have observed a quality regression in [feature]. The
feature is temporarily reduced in capability while we investigate." Avoid vendor naming; avoid
committing to a quality outcome.

**Fixture back-feed hint.** Candidates: (a) new canary eval case that reproduces the failing
query shape; (b) a threshold alert on `chatcat_groundedness_score` if one did not exist; (c) a
publish-gate feature flag that defaults to off when the eval signal is stale.

### 2.5 `rb-provider-degraded-fallback` — Hosted-model provider degraded

```yaml
---
runbook_id: rb-provider-degraded-fallback
title: "Hosted model provider degraded; fallback engagement"
symptom: "Provider error rate on chatcat routes exceeds the fallback-contract threshold (5xx + 429 + timeout above 2% on a 5m window) OR the vendor status ingestor publishes a degraded event on a route in the dependency map"
default_severity: SEV-2
severity_ladder:
  SEV-3: "single route; <5% error rate; < 10 min duration"
  SEV-2: "default"
  SEV-1: "fallback also degrading; multiple routes impacted; event exceeds the SLA credit threshold"
peer_owner: ai-infra-inference-engineer-oncall
secondary_peer_owner: ai-risk-engineer-oncall
related_runbooks: [rb-ttft-regression, rb-cost-blow-up]
mod_105_class: provider-degradation
fixture_backfeed_hints: [chaos-breaker-drill, status-ingestor-tightening]
last_reviewed: 2026-11-01
---
```

**Detection signals.**

- `chatcat_provider_error_rate{provider="<id>", route="<route>"}` (sum of 5xx + 429 + timeout
  as a fraction of requests) on `chatcat/serving-health` panel `Provider error rate per route`.
  Expected lag: 1 min.
- `chatcat_provider_latency_seconds{provider="<id>"}` p95. Expected lag: 1 min.
- `vendor_status_ingested_events_total{provider="<id>", severity=~"degraded|major"}` from the
  status-page ingestor (`mod-107` chapter 04). Expected lag: 1–5 min depending on provider.
- `chatcat_fallback_engagement_ratio{route="<route>"}` — the fraction of traffic the fallback is
  currently absorbing. Expected lag: 30s.

**Confirm checklist.**

1. **Provider confirmation** — does the vendor status page corroborate? (If not, we may be
   open-circuiting on a local signal; call it out.)
2. **Scope** — one route, one region, or fleet-wide on this provider?
3. **Fallback health** — is the fallback provider's own error rate within normal range? If the
   fallback is also degraded, this is a SEV-1.
4. **Contract coverage** — is this route covered by the fallback contract, or are we
   open-circuiting on an uncovered route? Consult the dependency map (`mod-107` chapter 01).
5. **Billing / SLA** — is the event accumulating toward the SLA credit threshold? (Paged
   explicitly, not debugged in-incident — flag for `ai-risk-engineer`.)

**Mitigation menu (ranked).**

| # | Mitigation | Blast radius | Reversibility | Pre-conditions | Time-to-effect |
|---|------------|--------------|---------------|----------------|-----------------|
| 1 | Open the `chatcat-provider-<id>` router breaker; fall through to the contracted fallback | User-facing (tone drift) | One-command | Fallback canary green; route is in the fallback contract | 30s |
| 2 | Pin requests to the healthy region if the event is regional | User-facing (latency hit for cross-region users) | One-command | Healthy region identified | 1 min |
| 3 | Apply a per-provider rate cap to prevent burn through the 429 wall | Dev-facing | One-command | Cap level configured | 30s |
| 4 | Shed non-critical traffic classes on this provider | Dev-facing | One-command | Admission policy classes present | 30s |
| 5 | Open a vendor support ticket with the incident ID and the error-rate graph | None (external) | n/a (not a mitigation, a notification) | Vendor account manager contact available | n/a |

**Handoff.** Page `ai-infra-inference-engineer-oncall` for routing control; page
`ai-risk-engineer-oncall` if the SLA credit threshold is crossed or if the event duration will
make a Chapter 06 handoff likely.

**User communication template.** "We are seeing degraded performance from an upstream provider
and have routed around it. Some responses may be slower or slightly different in style while we
monitor."  Do NOT name the provider until they have publicly acknowledged (chapter 02 §5.3).

**Fixture back-feed hint.** Candidates: (a) a chaos experiment that opens the breaker on a
staging canary and verifies the fallback engagement ratio reaches 100%; (b) a tightening of the
status-ingestor's event-threshold mapping if the ingestor lagged the real event.

### 2.6 `rb-gpu-driver-ecc-xid` — GPU driver / ECC / XID incident

```yaml
---
runbook_id: rb-gpu-driver-ecc-xid
title: "GPU driver, ECC, or XID incident on chatcat-serving"
symptom: "DCGM XID error class in the recently-fatal set (XID 13/31/43/48/63/64/74/79/92/94) on a serving node, OR an ECC double-bit error, OR a sudden spike in DCGM_FI_DEV_FB_USED errors coincident with pod restarts"
default_severity: SEV-3
severity_ladder:
  SEV-3: "single node, pod drained successfully, no user-visible impact"
  SEV-2: "multiple nodes OR cordon exhausted the healthy pool OR availability SLO burning"
  SEV-1: "cluster-wide pattern suggesting a driver upgrade regression"
peer_owner: ai-infra-performance-oncall
secondary_peer_owner: ai-infra-inference-engineer-oncall
related_runbooks: [rb-preemption-spike, rb-ttft-regression]
mod_105_class: hardware-gpu
fixture_backfeed_hints: [chaos-xid-injection, driver-pin-ci, pdb-audit]
last_reviewed: 2026-11-01
---
```

**Detection signals.**

- `DCGM_FI_DEV_XID_ERRORS` with label `xid` ∈ the recently-fatal set. Expected lag: 10–30s.
- `DCGM_FI_DEV_ECC_DBE_AGG_TOTAL` increase. Expected lag: 30s.
- Node-level Kubernetes event `NodeHasInsufficientGPU` or `UnhealthyGPU` taint applied.
  Expected lag: 1–3 min (kubelet event loop).
- `chatcat_serving_pod_restarts_total{reason="gpu-unhealthy"}` counter. Expected lag: 1 min.
- `nvidia-smi --query-gpu=driver_version,gpu_bus_id` snapshot — pin the current driver version
  per node pool; a recent change is the primary correlate.

**Confirm checklist.**

1. **Scope** — one node, one node pool, one zone, or fleet-wide?
2. **Driver version** — did the GPU driver, CUDA, or NCCL version change in the last 72h on
   this node pool? Consult the DaemonSet audit.
3. **XID class** — is the XID in the "benign" set (31 "GPU memory page fault, workload crash")
   or the "recently fatal" set (63/64 "ECC uncorrectable error; retire GPU")? The ladder is
   different.
4. **Pool capacity** — after cordoning the affected node(s), does the remaining pool have the
   capacity to hold the serving fleet? Consult `DCGM_FI_DEV_GPU_UTIL` summed across the
   `chatcat-serving` pool.
5. **Vendor RMA / cloud replacement path** — if nodes must be replaced, is the replacement
   cadence known (minutes for cloud GPUs, hours-to-days for on-prem)?

**Mitigation menu (ranked).**

| # | Mitigation | Blast radius | Reversibility | Pre-conditions | Time-to-effect |
|---|------------|--------------|---------------|----------------|-----------------|
| 1 | Cordon and drain the affected node(s) via `kubectl cordon` + `drain --ignore-daemonsets` | Dev-facing (pods reschedule) | One-command (`kubectl uncordon`) | PDB permits the drain; headroom in the pool | 2–5 min |
| 2 | Scale the serving StatefulSet by +1 replicas to preserve capacity during drain | Dev-facing | One-command | Cluster has GPU capacity | 3–8 min |
| 3 | Pin the node pool's driver DaemonSet to the last-known-good version | Fleet-wide | Coordinated unwind (DaemonSet rollout) | Last-known-good version identified | 10–30 min |
| 4 | Shift traffic to an unaffected region via the router if the event is regional | User-facing (latency hit cross-region) | One-command | Fallback contract covers region shift | 1 min |
| 5 | Open a cloud / vendor ticket with the DCGM XID dump and `nvidia-bug-report.sh` output | None | n/a | Vendor support path available | n/a |

**Handoff.** Page `ai-infra-performance-oncall` for the cluster / driver decisions; page
`ai-infra-inference-engineer-oncall` if serving configuration changes are required.

**User communication template.** Usually silent (handled by capacity). If the event causes
sustained impact, use `rb-ttft-regression`'s template.

**Fixture back-feed hint.** Candidates: (a) a chaos experiment that injects an XID 31 on a
staging node and verifies the drain-and-reschedule flow; (b) a CI gate that blocks driver
DaemonSet promotions without a staging canary period; (c) an audit on PDB and priority-class
values across the `chatcat-serving` pool.

## 3. Validation steps

A learner (or reviewer) runs the following validation steps against their own submission.

1. **YAML header parse.** Extract every YAML block with `yq eval '.runbook_id' *.md` (or
   equivalent). Every file must emit a `runbook_id`; every `runbook_id` must be unique and
   prefixed `rb-`.

   ```bash
   # from the runbooks/ directory
   for f in rb-*.md; do
     yq eval --front-matter=extract '.runbook_id' "$f" \
       || { echo "FAIL: $f has invalid YAML header"; exit 1; }
   done
   ```

2. **Section presence.** Grep each file for every one of the seven section headings. A missing
   section fails the runbook.

   ```bash
   REQUIRED=('## Detection signals' '## Confirm checklist' '## Mitigation menu' \
             '## Handoff' '## User communication' '## Fixture back-feed' )
   # header is the YAML block, so the structural check runs on the remaining six
   for f in rb-*.md; do
     for h in "${REQUIRED[@]}"; do
       grep -qF -- "$h" "$f" || { echo "FAIL: $f missing $h"; exit 1; }
     done
   done
   ```

3. **No placeholders.** Reject any file containing `TBD`, `TODO`, `FIXME`, `the metric that`,
   or `an alert for`.

4. **Alert ↔ runbook coverage (stretch).** For every Prometheus alert rule in
   `alerts/chatcat/*.yaml`, assert `annotations.runbook_id` exists and resolves to a file in
   `runbooks/`.

5. **Peer-dry-run.** Hand one runbook to an engineer who is not the author and read the
   confirm checklist against a fabricated alert payload. If they ask "what does X mean" more
   than twice, the runbook is under-specified.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Submission passes at ≥ 15 / 20 with no zero in dimensions
marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Category coverage *(hard gate)* | Fewer than three categories | Three categories, unbalanced | Three categories, balanced |
| 2 | Section completeness *(hard gate)* | One or more sections missing in any runbook | Every runbook has all seven, some thin | All seven sections present and populated |
| 3 | Header parseability *(hard gate)* | Missing or non-parseable YAML | Parses but missing required keys | Parses and complete |
| 4 | Detection signal specificity | "The metric for X" | Signal named but not grounded to a dashboard | Exact metric name, dashboard panel, and expected lag |
| 5 | Mitigation ranking | Unranked list | Ranked but no blast-radius tags | Ranked with blast-radius, reversibility, pre-conditions, time-to-effect |
| 6 | Handoff by role *(hard gate)* | Named persons appear | Role + named persons | Role-only, with structured payload |
| 7 | Communication template | Missing or generic placeholder | Present but vague | Specific, named placeholders, no vendor naming |
| 8 | Fixture back-feed hint | Missing | Present but vague | Specific fixture candidates mapped to chapter 05 §1 |
| 9 | Cover note | Missing or generic | Covers product + stack | Covers product, stack, gaps, and introduction plan |
| 10 | No placeholders *(hard gate)* | TBD / TODO / placeholder present | Rare placeholder | None |

## 5. Common mistakes

- **Generic "the metric that would be relevant here."** Specificity is the entire point. If the
  learner does not have a real metric, they must propose one and mark it `(proposed)`, not
  gesture at it.
- **Rollback as mitigation #1.** AI state (weights, caches, indices) does not roll back the way
  a container image does. Chapter 01 §1.4 is explicit. Open the breaker first.
- **Handoff by name.** Names change rotations; roles do not. `ai-infra-inference-engineer-oncall`
  is the right handle.
- **Vendor-naming in the user-communication template.** Naming a vendor before they publicly
  acknowledge is both a correctness risk and a relationship risk (chapter 02 §5.3).
- **All five runbooks on the serving path.** Three-category coverage is a hard gate; a submission
  with five serving-path runbooks has not demonstrated the library's shape.
- **Fixture back-feed section populated with "add an alert."** The hint should name the signal,
  the system, and the back-feed class from chapter 05 §1.
- **Confirm checklist too long.** More than six items is unreadable at 3 AM. Chapter 01 §1.3 is
  explicit on the four-to-six bound.

## 6. References

- Module chapter 01, *The Top-Twenty AI-Specific Runbook Library* — the seven-section shape,
  the twenty runbooks, and the authoring discipline this exercise implements.
- Module chapter 04, *Pattern-Matching Against Public Post-Mortems* — the rationale for
  grounding user-communication templates in what vendors have (and have not) been able to say.
- Module chapter 05, *Regression-Fixture Back-Feed Discipline* — the six fixture classes the
  back-feed hint references.
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — the industry-recognised vocabulary for ML-specific failure modes, useful when naming the
  class a runbook mitigates (e.g., ML02 data poisoning aligns with `rb-eval-set-poisoning`;
  ML06 ML supply chain aligns with `rb-provider-degraded-fallback`).
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-techniques
  framework; useful for safety-adjacent runbooks like `rb-toxic-output-blast` that must map to
  threat categories legal and security recognise.
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the risk-management vocabulary that aligns the runbook library with the function-level
  reliability program (`mod-105` chapter 02 cross-references).
