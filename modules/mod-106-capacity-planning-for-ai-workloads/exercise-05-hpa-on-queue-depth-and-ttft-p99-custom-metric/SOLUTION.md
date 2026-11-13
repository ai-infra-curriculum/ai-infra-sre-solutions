# Exercise 05 — HPA on Queue Depth and TTFT p99 Custom Metric — Solution

## 1. Solution overview

CPU utilisation is the wrong HPA signal for an LLM inference tier. GPUs
are the saturating resource, and GPU utilisation is itself a poor proxy
for serving health because a replica can look busy at the kernel level
while its *queue* is minutes deep. The right signals are:

- **Queue depth** (pending requests the server has accepted but not yet
  started). This is what the user feels before TTFT.
- **TTFT p99** (time-to-first-token at the 99th percentile). This is the
  SLO for an interactive workload.

This exercise asks the learner to wire a Kubernetes Horizontal Pod
Autoscaler to these two custom metrics, set sensible targets, and
operate safely at the boundary where scale-out is slower than demand
growth.

The reference solution below delivers:

1. A working HPA manifest (`autoscaling/v2`) that scales on both
   signals with `selectPolicy: Max`.
2. The Prometheus-adapter / external-metrics wiring that exposes
   `queue_depth` and `ttft_p99_ms` to the HPA.
3. The reasoning for the target values and the stabilisation windows.
4. An operating guide: pre-warming, scale-up/scale-down asymmetry, and
   interaction with the cluster autoscaler.
5. A rubric for review.

The specific Kubernetes API versions used here are the v2 autoscaling
group and the external-metrics / custom-metrics API; the HPA fields
referenced are those documented for `autoscaling/v2` in current
Kubernetes releases.

## 2. Worked implementation — HPA and wiring

### 2.1 HPA manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-serve
  namespace: inference
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-serve
  minReplicas: 4
  maxReplicas: 120
  metrics:
    # Signal 1: queue depth per replica. Averaged across pods so the HPA
    # scales to keep each replica's queue within budget rather than
    # scaling on a stuck single pod.
    - type: Pods
      pods:
        metric:
          name: queue_depth
        target:
          type: AverageValue
          averageValue: "8"
    # Signal 2: TTFT p99 as an external metric. This is an SLO-anchored
    # signal; crossing it scales out even if queue depth looks healthy
    # (e.g. decode-bound regression).
    - type: External
      external:
        metric:
          name: ttft_p99_ms
          selector:
            matchLabels:
              service: inference-serve
        target:
          type: Value
          value: "800"   # ms; aligned to the interactive TTFT SLO budget
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 10
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
      selectPolicy: Min
```

Why these specific choices:

- **`autoscaling/v2`** is required for multiple metric sources and the
  `behavior` block; `v1` does not support this.
- **Pods-type metric for `queue_depth`** so Kubernetes averages the
  per-pod value. A stuck replica with a deep queue does not drag the
  whole fleet into a scale-out — the average is bounded by the healthy
  replicas, and the stuck pod should be evicted by liveness, not papered
  over by HPA.
- **External-type metric for `ttft_p99_ms`** because it is a
  service-level aggregate, not a per-pod quantity. The external metric
  provider computes it from Prometheus histograms server-side.
- **`selectPolicy: Max` for scale-up.** When either signal wants a
  scale-out, do the larger one. A deep queue and a crossing TTFT p99
  usually fire together; using `Max` matches the demand envelope.
- **Scale-up stabilisation = 30 s, scale-down = 600 s.** Scale-up is
  cheap when wrong (a few extra replicas for a minute); scale-down is
  expensive when wrong (the queue blows back up and TTFT breaches).
  Asymmetry is the SRE pattern.
- **Scale-up policies capped at +100%/30 s and +10 pods/30 s with
  `Max`.** The percent policy wins at low fleet size (double from 4 to
  8 in 30 s). The absolute-pods policy wins at high fleet size (add 10
  at a time so a traffic spike does not try to double a 60-pod fleet
  in one tick and overflow node capacity).
- **Scale-down policy = 10%/60 s with `Min`.** Drains a factor-of-two
  spike over roughly seven minutes; still faster than a full
  demand-cycle but slow enough that the queue signal can argue back if
  the demand returns.

### 2.2 Prometheus-adapter / external-metrics wiring

The signals need to be exposed to the Kubernetes metrics API. The
standard path is:

1. Have the inference server emit Prometheus metrics:
   - A `queue_depth` gauge per pod.
   - A histogram for TTFT (e.g. `ttft_seconds_bucket`) per service, so
     p99 can be computed server-side with `histogram_quantile`.
2. Install `prometheus-adapter` with a config that maps a Prometheus
   expression onto each metric name the HPA references.

Example adapter config fragment:

```yaml
rules:
  # Signal 1: queue_depth per pod — exposed as a Pods-type metric.
  - seriesQuery: 'queue_depth{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: namespace}
        pod: {resource: pod}
    name:
      matches: "^(.*)$"
      as: "${1}"
    metricsQuery: 'avg_over_time(<<.Series>>{<<.LabelMatchers>>}[1m])'

externalRules:
  # Signal 2: ttft_p99_ms — service-level, exposed as an External metric.
  - seriesQuery: 'ttft_seconds_bucket{service="inference-serve"}'
    name:
      matches: ".*"
      as: "ttft_p99_ms"
    metricsQuery: |
      1000 * histogram_quantile(
        0.99,
        sum by (le, service) (
          rate(ttft_seconds_bucket{<<.LabelMatchers>>}[2m])
        )
      )
```

Two correctness notes:

- Use `avg_over_time(... [1m])` on the gauge so a single 15 s scrape
  blip does not jerk the HPA.
- Use a 2 min `rate(... [2m])` window on the histogram so p99 is not
  dominated by a single bucket crossing in one scrape interval. The
  stabilisation window on the HPA is a second line of defence, not the
  only one.

### 2.3 Target rationale

- **`queue_depth` target = 8.** Chosen as the point where mean queue
  wait with the measured service rate stays within the TTFT budget for
  the interactive class (see Exercise 01: with `μ_pf` and `U_target`
  from the base model, average wait ≈ `queue / μ_effective`). Teams
  should recompute this from their own numbers; 8 is a plausible
  starting value, not a universal constant.
- **`ttft_p99_ms` target = 800.** The interactive TTFT p99 SLO. This
  is an **SLO-anchored**, not throughput-anchored, signal: when p99
  crosses it the HPA scales out regardless of what queue depth says,
  because p99 catches decode-bound regressions and partial-failure
  degradations that queue depth misses.

### 2.4 Pre-warming and cluster-autoscaler interaction

HPA alone does not add nodes. If the fleet scales out beyond current
cluster capacity, pods go `Pending` and the cluster autoscaler must
provision nodes — which for GPU nodes often takes minutes. Mitigations:

1. **Overprovisioning pods.** Keep a small `PriorityClass`-low
   deployment running sentinel pods (requesting the same GPU) so the
   cluster autoscaler is already ahead of the fleet. Real HPA scale-ups
   evict the sentinels and get GPUs instantly; the autoscaler races to
   restore the sentinel.
2. **Scheduled baselines.** For predictable diurnal patterns, set
   `minReplicas` from a scheduled job so the fleet is already large
   before the morning peak.
3. **Model-weights warm start.** Bake weights into the container image
   or pre-fetch to node local SSDs so the first-ready time for a scaled
   pod is dominated by CUDA context, not by model download.
4. **Readiness gate honours warm-up.** The pod must not take traffic
   before the engine warm-up and a minimum-load smoke pass; otherwise
   p99 will spike on the newly-scaled pod and (ironically) trigger more
   scale-out.

### 2.5 Scale-down asymmetry and SLO safety

Scale-down is not symmetric to scale-up. The costs:

- Scale-up wrong → a few extra replica-minutes of cost.
- Scale-down wrong → queue backs up, TTFT p99 breaches SLO, incident.

The HPA's 600 s stabilisation window and 10%/60 s policy enforce this
asymmetry. A reviewer should look specifically for symmetric behavior
blocks (scale-up and scale-down with identical parameters) and reject
them — they are a signal the learner copied a default without thinking
about SLO cost.

## 3. Validation steps

1. **API version is `autoscaling/v2`.** `v1` cannot carry multiple
   metrics or the behavior block and must not be accepted.
2. **Both signals present.** A single-signal HPA (queue depth only, or
   TTFT only) fails the exercise. Queue catches throughput; TTFT catches
   SLO degradation.
3. **`selectPolicy: Max` on scale-up.** Otherwise the fleet scales to
   the smaller of the two recommendations and under-provisions during
   demand bursts.
4. **Scale-up / scale-down asymmetry.** Scale-down stabilisation ≥ 10×
   scale-up stabilisation, scale-down policy conservative compared to
   scale-up policy.
5. **Readiness gate covers warm-up.** New pods do not receive traffic
   until the serving engine passes a warm-up probe.
6. **Metric pipeline sanity.** The Prometheus adapter rules apply a
   smoothing window (avg_over_time / rate) consistent with the HPA
   sync period (15 s default) so no single scrape can jerk the HPA.
7. **Pre-warm strategy named.** Cluster-autoscaler interaction is
   addressed — sentinel pods, scheduled baselines, or both. A plan
   that assumes GPU nodes appear instantly is wrong.
8. **minReplicas floor is non-zero.** A zero floor guarantees a first
   TTFT breach after any idle window.
9. **maxReplicas ceiling is justified.** It should be at or above the
   Exercise-02 forecast peak × a safety factor, and it should not
   exceed the pool's provider quota (Exercise 04) when the service
   fronts a hosted model.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | `autoscaling/v2` HPA with both `queue_depth` (Pods) and
      `ttft_p99_ms` (External) | 2 |
| 2 | `selectPolicy: Max` on scale-up | 1 |
| 3 | Scale-up and scale-down asymmetry in stabilisation and policy | 2 |
| 4 | Prometheus-adapter / external-metrics config that smooths the
      source series | 1 |
| 5 | Readiness gate covers engine warm-up | 1 |
| 6 | Pre-warm / cluster-autoscaler interaction addressed (sentinels or
      scheduled baselines) | 1 |
| 7 | Target values tied to measured base-model numbers and SLO budgets | 1 |
| 8 | `minReplicas ≥ 1` with justification; `maxReplicas` bounded by
      provider quota or fleet peak | 1 |

Deductions:

- CPU-utilisation metric present in the HPA: −2 (CPU is not the
  saturating resource).
- Symmetric scale-up / scale-down behavior: −2.
- No readiness gate / warm-up: −2.
- No cluster-autoscaler interaction addressed: −1.
- Metric smoothing missing and HPA sync period not accounted for: −1.

## 5. Common mistakes

- **Scaling on CPU utilisation.** CPU does not reflect GPU-side
  saturation. A fleet at 30% CPU can be at 100% GPU — or sitting on an
  idle GPU but with its queue backed up.
- **Scaling on GPU utilisation only.** GPU-busy includes idle kernel
  launches and warm-up noise; it is a poor SLO proxy by itself.
- **Scaling on a single signal.** Queue depth misses decode-bound
  regressions where decode-token service time lengthens without the
  queue growing; TTFT alone misses sustained saturation where p99 is
  slow to move but queue is already deep.
- **Using `autoscaling/v1`.** No multi-metric, no behavior block.
- **Copying a web-service HPA.** The stabilisation windows,
  warm-up, and scale-down discipline are wrong by two or three orders
  of magnitude for LLM inference.
- **Serving traffic before the engine is warm.** Readiness becomes
  "pod has started" rather than "engine passes smoke test", and p99
  spikes on every scale-out.
- **Ignoring cluster autoscaler latency.** HPA requests pods; pods go
  `Pending`; the cluster autoscaler adds nodes a few minutes later. By
  then TTFT has breached. Sentinel pods fix this.
- **Setting `maxReplicas` above the provider quota.** The HPA cheerfully
  scales into a wall of 429s because the model gateway cannot keep up.
  The HPA ceiling must respect the Exercise 04 pool ceiling.
- **Metric series not smoothed.** The HPA controller sync period is
  short; raw per-scrape jitter will produce flapping. Smooth in the
  adapter rule.

## 6. References

- Kubernetes documentation — *Horizontal Pod Autoscaler*
  (`autoscaling/v2` API, behavior block, selectPolicy, custom and
  external metrics) —
  <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>.
- Kubernetes documentation — *Horizontal Pod Autoscaler Walkthrough* and
  the custom-metrics API design —
  <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/>.
- `kubernetes-sigs/prometheus-adapter` — configuration for exposing
  Prometheus series as custom and external metrics —
  <https://github.com/kubernetes-sigs/prometheus-adapter>.
- Google SRE Workbook — *Managing Load* and the scale-up / scale-down
  asymmetry principle —
  <https://sre.google/workbook/>.
- NIST AI Risk Management Framework (AI RMF 1.0) — Manage function,
  continuous monitoring of performance against targets, informing the
  SLO-anchored metric choice —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- Local exercise context: target values for `queue_depth` and
  `ttft_p99_ms` come from Exercise 01 (base model) and Exercise 02
  (workload classes). `maxReplicas` is bounded above by the provider
  quota plan in Exercise 04 and is empirically validated by the load
  test in Exercise 06.
