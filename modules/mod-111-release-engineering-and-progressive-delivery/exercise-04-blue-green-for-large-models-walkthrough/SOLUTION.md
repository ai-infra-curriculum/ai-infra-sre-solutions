# Exercise 04 — Blue / Green For Large Models Walkthrough — Reference Solution

Paired learning file: `lessons/mod-111-release-engineering-and-progressive-delivery/exercises/exercise-04-blue-green-for-large-models-walkthrough.md`.

> **Scope note.** The paired exercise asks the learner to walk through a blue/green deployment for a large self-hosted model (tens-to-hundreds of GB of weights, multi-GPU serving replicas), reasoning about capacity, warm-up, cache, and rollback. The reference solution works a plausible illustrative shape and names the places the classical stateless-service blue/green playbook breaks.

## 1. Solution overview

A classical web-service blue/green has a cheap, symmetric colour switch: both colours are already running, the LB atomically shifts traffic, and rollback is a re-flip. Large-model blue/green breaks at least four of those assumptions:

1. **Both colours are not cheap to stand up.** 70 B-class weights take minutes-to-tens-of-minutes to load into GPU memory per replica; a 20-replica green fleet may need 30 min before it can take production traffic. The "ready" signal is not pod-ready, it is "weights loaded, warm-up prompts flushed, KV-cache primed."
2. **Capacity doubles while both colours run.** Running blue and green side-by-side at full capacity doubles GPU spend for the overlap window. Fleet sizing, quota, and cost budget must all accommodate this.
3. **State moves with the request.** Prompt cache, KV cache, retrieval index, and per-user memory are per-replica or per-cluster. A naïve LB flip discards all the warm cache on blue and ships green with a cold cache — producing a user-visible latency regression during what should be the smoothest moment of the release.
4. **Rollback is not cheap either.** If blue has been scaled down after the flip, "flip back" is actually "spin up blue, wait 20 min, flip back". The rollback plan has to pre-commit to keeping blue warm for the agreed hold window.

The reference solution walks a blue/green release of `chatcat-serving` from a Llama-3-70B-class model (blue) to a hypothetical Llama-3-70B-finetune (green), on H100 ×8 pods, behind a shared gateway and a shared prompt-cache / retrieval stack. It shows the manifests, the ordering, the warm-up protocol, the metrics that gate the flip, and the rollback plan.

### 1.1 When blue/green beats canary

- When the model change is **not safely incrementalable**: a tokenizer change, a cross-architecture model swap (dense → MoE), a context-window change, or a prompt-cache-breaking template change. A canary at 5 % blends two cache populations and hurts both.
- When **shadow coverage is already strong** (exercise 03). Blue/green is a "flip after shadow" shape for changes that *cannot* run gradually.
- When **regulatory or contractual requirements** demand an atomic cutover with an attestation (e.g. a named model version appears in a SOC-2 attestation; the switch must be identifiable to a single timestamp).

Otherwise, prefer a canary (exercise 01). Blue/green pays the doubled-capacity premium; canary does not.

## 2. Implementation — worked answer

### 2.1 Pre-flight ledger

Before the flip, the following must all be true. The release is blocked otherwise.

| # | Pre-flight check | Source | Pass condition |
|---|------------------|--------|----------------|
| 1 | Green fleet sized identically to blue | `kubectl get sts chatcat-serving-{blue,green}` | Replica counts equal; GPU resource requests equal |
| 2 | Green image pullable on all target nodes | DaemonSet pre-pull run on green node pool | Pre-pull job reports success on every node |
| 3 | Weights pre-staged | Local cache dir populated on every green node by the weight-shipper sidecar | `kubectl exec green-<n> -- ls /weights/` shows expected SHAs on every replica |
| 4 | Warm-up prompts available | `warmup/` ConfigMap exists in the green namespace | At least 50 prompts across all cache-relevant prefixes |
| 5 | Shadow verdict is green | Exercise-03 shadow summary last 24 h | `safety_reg = 0`, per-route `quality_reg/total` within route budget |
| 6 | Capacity reservation | Cloud quota check; cluster autoscaler headroom | Headroom ≥ (blue size + green size) GPUs for the planned overlap window |
| 7 | Rollback SLA budget | Cost budget check | Enough GPU-hours in budget to hold blue warm for at least the hold window |
| 8 | On-call confirmation | PagerDuty override check | `ai-infra-inference-engineer-oncall` and `ai-engineering-oncall` both acknowledged |
| 9 | Communication plan | Release doc approved | Internal customers notified in #chatcat-releases; user-communication template ready |
| 10 | Rollback runbook linked | Release doc field `rollback_runbook_id` populated | Field matches an entry in the runbook library |

### 2.2 Weight staging

A `DaemonSet` runs on the green node pool to pre-pull the weights into a hostPath cache *before* any green pod is scheduled. Without this, the first green pod on every node pays the full model-weight download on start.

`k8s/green/weights-prepull.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: chatcat-weights-prepull-green
  namespace: chatcat
spec:
  selector:
    matchLabels:
      app: chatcat-weights-prepull-green
  template:
    metadata:
      labels:
        app: chatcat-weights-prepull-green
    spec:
      nodeSelector:
        chatcat.io/pool: green
      containers:
        - name: prepull
          image: ghcr.io/example/chatcat-weight-shipper:1.3.0
          args:
            - --target-dir=/weights
            - --model-sha=REPLACED-BY-CI
            - --from=s3://chatcat-weights/
          volumeMounts:
            - name: weights
              mountPath: /weights
      volumes:
        - name: weights
          hostPath:
            path: /mnt/data/chatcat-weights
            type: DirectoryOrCreate
```

A follow-up `Job` asserts every green node reports success; the pre-flight ledger reads that job's status.

### 2.3 Green StatefulSet

`k8s/green/chatcat-serving-green.yaml` (abridged):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: chatcat-serving-green
  namespace: chatcat
spec:
  replicas: 20
  serviceName: chatcat-serving-green
  podManagementPolicy: Parallel
  selector:
    matchLabels:
      app: chatcat-serving
      colour: green
  template:
    metadata:
      labels:
        app: chatcat-serving
        colour: green
    spec:
      nodeSelector:
        chatcat.io/pool: green
      containers:
        - name: vllm
          image: ghcr.io/example/chatcat-serving:REPLACED-BY-CI
          args:
            - --model=/weights
            - --tokenizer=/weights
            - --max-model-len=16384
            - --tensor-parallel-size=8
          readinessProbe:
            httpGet:
              path: /health/warm
              port: 8080
            periodSeconds: 10
            failureThreshold: 60   # 10 min window to become warm
          startupProbe:
            httpGet:
              path: /health/loaded
              port: 8080
            periodSeconds: 10
            failureThreshold: 180  # 30 min weight-load ceiling
          resources:
            limits:
              nvidia.com/gpu: 8
          volumeMounts:
            - name: weights
              mountPath: /weights
              readOnly: true
      volumes:
        - name: weights
          hostPath:
            path: /mnt/data/chatcat-weights
            type: Directory
```

Two readiness signals:

- `/health/loaded` — the model weights are on GPU, the KV allocator is initialised.
- `/health/warm` — warm-up prompts have been flushed; the prompt cache has been primed; the groundedness canary has been run through once.

Only `/health/warm` counts as "ready for traffic". Adding the second signal is the single-biggest difference between this manifest and a classical web-service StatefulSet.

### 2.4 Warm-up protocol

`warmup/` is a ConfigMap containing prompts that exercise every cacheable prefix in production: system prompts, tool-stub prompts, retrieval-template prompts, and locale variants. A `Job` runs against each green pod on startup, issuing the warm-up prompts and confirming responses before the pod flips `/health/warm` to 200.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: chatcat-serving-green-warmup
  namespace: chatcat
spec:
  parallelism: 20
  completions: 20
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: warmup
          image: ghcr.io/example/chatcat-warmup:1.0.0
          env:
            - name: TARGET
              value: "http://chatcat-serving-green:8080"
            - name: PROMPTS
              value: "/warmup/prompts.jsonl"
          volumeMounts:
            - name: warmup
              mountPath: /warmup
      volumes:
        - name: warmup
          configMap:
            name: chatcat-warmup-prompts
```

The Job is parallelised to the replica count so no pod waits for another; wall clock ≈ max(single-pod warm-up).

### 2.5 Gateway flip

The gateway is an Istio `VirtualService` with two subsets. The flip is a single-field edit; applied via `kubectl apply` or an Argo CD sync of a new `VirtualService` revision.

`k8s/istio/chatcat-serving-flip.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: chatcat-serving
  namespace: chatcat
spec:
  hosts:
    - chatcat-serving
  http:
    - route:
        - destination:
            host: chatcat-serving
            subset: green
          weight: 100
        - destination:
            host: chatcat-serving
            subset: blue
          weight: 0
```

The operator applies the manifest and watches:

- `chatcat_requests_total{colour="green"}` ramps to 100 % within one minute (`envoy` round-trip for routing config).
- `chatcat_ttft_seconds` p95 stays within the per-route SLO for the first five minutes.
- `chatcat_prompt_cache_hit_ratio` on green warms up to its steady-state within 15 minutes (warm-up pre-primed it; real traffic pushes it to steady state).
- `chatcat_provider_error_rate` stays at baseline.

### 2.6 Hold window

Blue is held at `replicas=20` and `weight=0` for a published hold window — typically one hour for a quality-only change and four hours for a tokenizer- or context-window change. Rollback during the hold is a single-field flip back. After the hold, blue scales to zero and is torn down.

Hold-window decisions:

- **Short hold (1 h).** Change is reversible without data regeneration: prompt change, fine-tuned weights with same tokenizer.
- **Extended hold (4 h).** Change touches tokenizer, context window, or prompt-cache keying. These are the changes whose quality regressions surface slowly in rare cohorts.
- **Extended hold (24 h).** Regulated route: hold until the next-business-day quality review.

### 2.7 Rollback plan

Rollback during hold:

1. Operator applies the inverse manifest (`weight: 100 → blue, weight: 0 → green`).
2. Verify traffic lands on blue within one minute.
3. Keep green warm (not terminated) until the root cause is identified; the problem fleet is sometimes the diagnostic artefact.
4. Open an incident and page `ai-infra-inference-engineer-oncall`.

Rollback after hold:

1. Scale blue StatefulSet back to the pre-release replica count; wait for `/health/warm` on every pod.
2. Apply the inverse manifest.
3. Expect **extended user-visible latency** during the re-warm; publish a maintenance notice before the flip.

### 2.8 Attestation record

On a successful cutover, the release pipeline writes to the release ledger:

```json
{
  "release_id": "rel-2026-11-13-chatcat-serving-g42a3",
  "strategy": "blue-green",
  "from_model_sha": "...",
  "to_model_sha": "...",
  "cutover_ts": "2026-11-13T14:08:00Z",
  "hold_window_end_ts": "2026-11-13T15:08:00Z",
  "blue_torn_down_ts": null,
  "rollback_executed": false,
  "pre_flight_ledger_ref": "s3://chatcat-releases/rel-2026-11-13-chatcat-serving-g42a3/preflight.json",
  "shadow_summary_ref": "s3://chatcat-shadow/.../2026-11-12.json",
  "approver_roles": ["ai-infra-inference-engineer-oncall", "ai-engineering-oncall"]
}
```

This is the attestation artefact that compliance uses.

## 3. Validation steps

A reviewer runs the walkthrough in the staging environment against a smaller model and verifies each stage.

1. **Pre-flight ledger blocks on any failure.** Manually fail check 3 (weight staging) and verify the release pipeline refuses to proceed.
2. **Weights are present before pods start.** Confirm that first-pod cold-start time with pre-pull is within the budget; disable pre-pull and confirm the budget is missed. The pre-pull is the single intervention that keeps the release in budget.
3. **`/health/warm` holds traffic.** Confirm that scaling green up but *not* running warm-up leaves `/health/warm` 503 even while `/health/loaded` is 200. The gateway sees green as not-ready and routes all traffic to blue.
4. **Warm-up primes the prompt cache.** `chatcat_prompt_cache_hit_ratio{colour="green"}` is within 5 pp of blue's steady-state value before the flip. If not, warm-up prompts are incomplete and the release is blocked.
5. **Flip is fast and reversible.** Flip to green; measure the time from `kubectl apply` to 95 % of requests on green (target < 60 s). Flip back; verify same.
6. **Hold-window rollback does not regress latency.** During hold, rollback the flip. Blue is still warm; latency on blue equals pre-release latency.
7. **Post-hold rollback is slow and announced.** After hold, scale blue to zero. Attempt a rollback: the pipeline should refuse without an explicit `--urgent` flag, which triggers the 20-minute re-warm and a user-facing maintenance notice.
8. **Attestation record.** Confirm the release ledger contains the attestation JSON. Confirm compliance can reconstruct who approved, what shadow verdict backed it, and when the cutover happened.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | When to pick blue/green over canary | Not justified | Justified in one sentence | Justified against ≥ 2 of (irreducibility, shadow backing, compliance) |
| 2 | Capacity planning *(hard gate)* | Assumes free capacity | Notes doubled footprint | Doubled footprint + explicit cost / quota pre-check |
| 3 | Pre-flight ledger *(hard gate)* | Missing | ≤ 5 checks | ≥ 9 checks including rollback SLA budget |
| 4 | Weight staging | Weights downloaded at pod start | Pre-pull without verification | Pre-pull Job + per-node verification |
| 5 | Dual readiness signal *(hard gate)* | One probe | `/health/loaded` only | `/health/loaded` + `/health/warm` with warm-up gating `/health/warm` |
| 6 | Warm-up protocol | Not addressed | Ad hoc scripted | ConfigMap + parallel Job + prompt-cache-hit metric gate |
| 7 | Flip mechanism | kubectl rolling update | Manual LB edit | Declarative VirtualService (or equivalent) with atomic weight swap |
| 8 | Hold window *(hard gate)* | Not defined | Fixed value with no rationale | Change-class-dependent with explicit categories |
| 9 | Rollback plan | "Flip back" only | Distinguishes during-hold vs. post-hold | During-hold + post-hold + urgent-with-warning path |
| 10 | Attestation / ledger | Missing | Present but minimal | Present with pre-flight, shadow, approvers, rollback flag |

## 5. Common mistakes

- **Treating "pod ready" as "take traffic".** The Kubernetes readiness probe is designed for stateless HTTP apps. A large-model pod is ready to serve *correct* responses only after warm-up; `/health/warm` is the production-grade signal.
- **Forgetting the doubled capacity bill.** A 20-replica blue/green overlap costs twice the normal serving bill for the overlap window; teams discover this after the finance team notices, which is the wrong time.
- **Pre-pulling only one copy of the weights.** Weight storage is per-node, not per-cluster, for large models. Pre-pulling into a shared PVC gives you one network race and a lot of GPUs waiting.
- **Warming up with the wrong prompts.** The warm-up set must match the real prompt-cache keys in production. Warm-up that exercises only "hello world" primes nothing useful.
- **Scaling blue down during the hold window.** The entire point of a hold window is a cheap rollback. Scaling blue down defeats it; the rollback plan now carries the 20-minute re-warm as unavoidable.
- **Flipping without shadow coverage.** Blue/green has no "ramp" to learn on; the shadow summary (exercise 03) is the only pre-release signal for quality. A green-colour release without a `shadow-green` tag is untested on real traffic.
- **No attestation.** Compliance reviews eventually ask "when did this model switch". A ledger record is cheap on the day of; reconstructing one from logs after the fact is not.

## 6. References

- Kubernetes documentation, *Configure Liveness, Readiness and Startup Probes* — [kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — the probe primitive this solution leans on for the dual `/health/loaded` + `/health/warm` signal.
- Istio, *Traffic Management: Weighted Routing* — [istio.io/latest/docs/tasks/traffic-management/traffic-shifting/](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/) — the single-field flip in §2.5 is the documented primitive.
- vLLM, *Serving — architecture* — [docs.vllm.ai/en/latest/serving/distributed_serving.html](https://docs.vllm.ai/en/latest/serving/distributed_serving.html) — tensor-parallel and weight-loading semantics referenced in the StatefulSet.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML Supply Chain` motivates attestation for the weight SHA used in the attestation record.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "ML Model Tamper" technique motivates the weight-SHA attestation path.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `GOVERN` function's "release decision accountability" is the attestation record's purpose.
