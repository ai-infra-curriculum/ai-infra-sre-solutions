# Exercise 01 — Canary Rollout With Flagger or Argo Rollouts — Reference Solution

Paired learning file: `lessons/mod-111-release-engineering-and-progressive-delivery/exercises/exercise-01-canary-rollout-with-flagger-or-argo-rollouts.md`.

> **Scope note.** The paired exercise prompt asks the learner to design and author a progressive-delivery pipeline for a model-backed service using one of the two reference tools. The reference solution below works both the Argo Rollouts and the Flagger shapes side-by-side so a reviewer can see what the solution looks like in either tool and score a submission that picks either. The illustrative product is the same fabricated `chatcat` RAG-chat surface used across this module family.

## 1. Solution overview

A good canary rollout for a model-backed service has to do three jobs that a classic microservice canary does not have to do:

1. **Gate on model-quality signals, not only serving-path signals.** A canary that passes CPU and latency but regresses groundedness is a failed canary. The weight step must be gated on both an "SLO health" analysis (error rate, latency, saturation) and an "eval health" analysis (groundedness, refusal rate, structured-output validity, a user-visible quality proxy).
2. **Weight carefully.** Model weights (especially for a hosted model) have a non-linear cost curve: the first 1 % of traffic is almost free, the first 25 % often reveals the real failure mode, and 100 % is where rare-cohort regressions surface. The step ladder in the solution reflects that.
3. **Abort to a known-good baseline, not to "the last deploy".** The target of rollback is the version with the last green eval, not simply the previous image.

The solution below implements a canary that runs `chatcat-serving` (a RAG chat route served by vLLM behind an Envoy-based gateway with NGINX ingress) with three analysis templates (two tool-native, one webhook-driven for the eval score). Both an Argo Rollouts manifest set and a Flagger manifest set are given. Reviewers should score either implementation against the rubric in §4.

### 1.1 Decision rationale — Argo Rollouts vs. Flagger

| Dimension | Argo Rollouts | Flagger |
|-----------|---------------|---------|
| Rollout controller model | Replaces the `Deployment` resource with a `Rollout` CRD; the controller owns the replica progression and weight steps. ([argo-rollouts.readthedocs.io / Architecture](https://argo-rollouts.readthedocs.io/en/stable/architecture/)) | Watches a `Deployment` and generates a parallel canary `Deployment` plus service-mesh / ingress routing. ([docs.flagger.app / How it works](https://docs.flagger.app/usage/how-it-works)) |
| Traffic control | Supports several mesh/ingress backends (SMI, Istio, NGINX, ALB, Traefik, etc.). ([argo-rollouts.readthedocs.io / Traffic Management](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/)) | Supports Istio, Linkerd, App Mesh, NGINX, Gloo, Contour, Kuma, Traefik, Open Service Mesh, Skipper, and Gateway API. ([docs.flagger.app / Deployment strategies](https://docs.flagger.app/usage/deployment-strategies)) |
| Analysis mechanism | `AnalysisTemplate` with per-metric providers (Prometheus, Kayenta, Wavefront, Web, Job, etc.). ([argo-rollouts.readthedocs.io / Analysis](https://argo-rollouts.readthedocs.io/en/stable/features/analysis/)) | `MetricTemplate` with Prometheus / Datadog / New Relic / Google-Cloud / CloudWatch / Graphite providers plus custom webhooks. ([docs.flagger.app / Metrics analysis](https://docs.flagger.app/usage/metrics)) |
| Argo CD integration | Native; same project. | Standalone operator; integrates with GitOps tooling of choice. |
| When to pick which | You already run Argo CD and want progressive delivery to live in the same control plane. | You already run a service mesh and want the canary logic to live close to the mesh's routing primitives, or you need the broader list of mesh integrations. |

Both are legitimate choices. The reference solution produces one artifact set per tool so reviewers can follow whichever the learner chose.

### 1.2 Step ladder and gates

The reference step ladder for `chatcat-serving` is `5% → 25% → 50% → 100%` with a 15-minute analysis interval between steps. The ladder is chosen because:

- **5 %** is small enough that a catastrophic quality regression on a specific cohort is survivable (and small enough that the eval harness can be primed with a short burst of shadowed real traffic without breaking the groundedness sampler).
- **25 %** is the first step where cache-contention and shared-state effects (shared prompt cache, shared KV cache, shared retrieval index) become visible.
- **50 %** balances eval variance: the sampled groundedness signal has enough mass to be statistically useful at 15-minute windows.
- **100 %** is a cutover, not a step; it should only happen after both the "SLO health" and "eval health" analyses have passed at 50 %.

Three analysis templates gate every step:

- **`slo-health`** — error rate, TTFT p95, provider error rate. Thresholds tied to the route's SLO (`mod-103`'s SLO stack is the source of truth).
- **`eval-health`** — groundedness, refusal rate, structured-output validity rate from the hourly judge pipeline sampled against the canary. Threshold is a *relative* bound ("no worse than baseline minus 2 sigma"), not an absolute.
- **`cost-health`** — tokens-per-request and `$-per-request` on the canary vs. baseline. Threshold is `canary <= 1.15 * baseline` as a cost-regression guardrail.

An analysis failure aborts the rollout; a missing signal (eval harness not reporting) is treated as a failure, not as a pass, because the gate is designed to fail closed.

## 2. Implementation — worked answer

### 2.1 Argo Rollouts shape

`k8s/argo-rollouts/chatcat-serving.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: chatcat-serving
  namespace: chatcat
  labels:
    app.kubernetes.io/name: chatcat-serving
    app.kubernetes.io/component: inference
spec:
  replicas: 20
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: chatcat-serving
  template:
    metadata:
      labels:
        app: chatcat-serving
    spec:
      containers:
        - name: vllm
          image: ghcr.io/example/chatcat-serving:REPLACED-BY-CI
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
            failureThreshold: 3
          resources:
            limits:
              nvidia.com/gpu: 1
  strategy:
    canary:
      canaryService: chatcat-serving-canary
      stableService: chatcat-serving-stable
      trafficRouting:
        nginx:
          stableIngress: chatcat-serving
      analysis:
        startingStep: 0
        templates:
          - templateName: slo-health
          - templateName: eval-health
          - templateName: cost-health
        args:
          - name: service-name
            value: chatcat-serving
          - name: namespace
            value: chatcat
      steps:
        - setWeight: 5
        - pause: { duration: 15m }
        - analysis:
            templates:
              - templateName: slo-health
              - templateName: eval-health
              - templateName: cost-health
        - setWeight: 25
        - pause: { duration: 15m }
        - analysis:
            templates:
              - templateName: slo-health
              - templateName: eval-health
              - templateName: cost-health
        - setWeight: 50
        - pause: { duration: 15m }
        - analysis:
            templates:
              - templateName: slo-health
              - templateName: eval-health
              - templateName: cost-health
        - setWeight: 100
```

`k8s/argo-rollouts/analysis-slo-health.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: slo-health
  namespace: chatcat
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    - name: canary-error-rate
      interval: 1m
      count: 10
      successCondition: result[0] <= 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(
              chatcat_requests_total{service="{{args.service-name}}",canary="true",status=~"5..|429"}[5m]
            ))
            /
            sum(rate(
              chatcat_requests_total{service="{{args.service-name}}",canary="true"}[5m]
            ))
    - name: canary-ttft-p95
      interval: 1m
      count: 10
      successCondition: result[0] <= 2.0
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.95,
              sum by (le) (
                rate(chatcat_ttft_seconds_bucket{service="{{args.service-name}}",canary="true"}[5m])
              )
            )
    - name: provider-error-rate
      interval: 1m
      count: 10
      successCondition: result[0] <= 0.02
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(
              chatcat_provider_errors_total{service="{{args.service-name}}",canary="true"}[5m]
            ))
            /
            sum(rate(
              chatcat_provider_requests_total{service="{{args.service-name}}",canary="true"}[5m]
            ))
```

`k8s/argo-rollouts/analysis-eval-health.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: eval-health
  namespace: chatcat
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    - name: groundedness-delta
      interval: 5m
      count: 3
      successCondition: result[0] >= -0.02
      failureLimit: 1
      provider:
        web:
          url: http://eval-gate.chatcat.svc:8080/v1/eval-delta
          method: POST
          headers:
            - key: Content-Type
              value: application/json
          body: |
            {
              "service": "{{args.service-name}}",
              "metric": "groundedness",
              "window_minutes": 30,
              "canary_label": "canary=true",
              "baseline_label": "canary=false"
            }
          jsonPath: "{$.delta}"
    - name: refusal-rate-delta
      interval: 5m
      count: 3
      successCondition: result[0] <= 0.05
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            (
              sum(rate(chatcat_refusals_total{service="{{args.service-name}}",canary="true"}[15m]))
              /
              sum(rate(chatcat_requests_total{service="{{args.service-name}}",canary="true"}[15m]))
            )
            -
            (
              sum(rate(chatcat_refusals_total{service="{{args.service-name}}",canary="false"}[15m]))
              /
              sum(rate(chatcat_requests_total{service="{{args.service-name}}",canary="false"}[15m]))
            )
    - name: structured-output-validity
      interval: 5m
      count: 3
      successCondition: result[0] >= 0.98
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(chatcat_structured_output_valid_total{service="{{args.service-name}}",canary="true"}[15m]))
            /
            sum(rate(chatcat_structured_output_attempts_total{service="{{args.service-name}}",canary="true"}[15m]))
```

`k8s/argo-rollouts/analysis-cost-health.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: cost-health
  namespace: chatcat
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    - name: tokens-per-request-ratio
      interval: 2m
      count: 5
      successCondition: result[0] <= 1.15
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            (
              sum(rate(chatcat_tokens_total{service="{{args.service-name}}",canary="true"}[15m]))
              /
              sum(rate(chatcat_requests_total{service="{{args.service-name}}",canary="true"}[15m]))
            )
            /
            (
              sum(rate(chatcat_tokens_total{service="{{args.service-name}}",canary="false"}[15m]))
              /
              sum(rate(chatcat_requests_total{service="{{args.service-name}}",canary="false"}[15m]))
            )
```

### 2.2 Flagger shape

`k8s/flagger/chatcat-serving-canary.yaml`:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: chatcat-serving
  namespace: chatcat
spec:
  provider: nginx
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chatcat-serving
  ingressRef:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: chatcat-serving
  autoscalerRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: chatcat-serving
  progressDeadlineSeconds: 3600
  service:
    port: 8080
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 100
    stepWeights: [5, 25, 50, 100]
    metrics:
      - name: canary-error-rate
        templateRef:
          name: chatcat-error-rate
          namespace: chatcat
        thresholdRange:
          max: 1
        interval: 1m
      - name: canary-ttft-p95
        templateRef:
          name: chatcat-ttft-p95
          namespace: chatcat
        thresholdRange:
          max: 2
        interval: 1m
      - name: canary-groundedness-delta
        templateRef:
          name: chatcat-groundedness-delta
          namespace: chatcat
        thresholdRange:
          min: -0.02
        interval: 5m
      - name: canary-tokens-ratio
        templateRef:
          name: chatcat-tokens-ratio
          namespace: chatcat
        thresholdRange:
          max: 1.15
        interval: 2m
    webhooks:
      - name: pre-rollout-gate
        type: pre-rollout
        url: http://eval-gate.chatcat.svc:8080/v1/pre-rollout
        timeout: 30s
        metadata:
          type: bash
          cmd: "check-canary-eval chatcat-serving"
      - name: post-rollout-record
        type: post-rollout
        url: http://release-recorder.chatcat.svc:8080/v1/record
        timeout: 15s
```

`k8s/flagger/metric-template-groundedness.yaml`:

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: chatcat-groundedness-delta
  namespace: chatcat
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring.svc:9090
  query: |
    (
      avg_over_time(chatcat_groundedness_score{service="{{ target }}",canary="true"}[15m])
    )
    -
    (
      avg_over_time(chatcat_groundedness_score{service="{{ target }}",canary="false"}[15m])
    )
```

### 2.3 CI integration (both tools)

The pipeline that promotes a new image calls the rollout and blocks on completion. The wrapper gives the operator a single exit code and a `promotion.json` record for the release ledger (which `mod-111/exercise-06` later ingests).

`scripts/rollout.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE="${1:-chatcat-serving}"
IMAGE="${2:?image tag required}"
TOOL="${TOOL:-argo-rollouts}"  # or "flagger"

case "$TOOL" in
  argo-rollouts)
    kubectl argo rollouts set image "$SERVICE" "vllm=$IMAGE" -n chatcat
    kubectl argo rollouts status "$SERVICE" -n chatcat --timeout 3600s
    ;;
  flagger)
    kubectl set image "deployment/$SERVICE" "vllm=$IMAGE" -n chatcat
    # Poll the Canary status until it reports Succeeded or Failed.
    for _ in $(seq 1 180); do
      phase=$(kubectl -n chatcat get canary "$SERVICE" -o jsonpath='{.status.phase}')
      [[ "$phase" == "Succeeded" ]] && exit 0
      [[ "$phase" == "Failed" ]] && exit 1
      sleep 20
    done
    exit 2  # timed out
    ;;
  *)
    echo "unknown tool: $TOOL" >&2
    exit 64
    ;;
esac
```

### 2.4 Abort criteria recap

The rollout aborts on any of:

- any `slo-health` metric failing its success condition beyond `failureLimit`,
- any `eval-health` metric failing (groundedness delta below −0.02, structured-output validity below 98 %),
- a `cost-health` tokens-per-request ratio above 1.15,
- the webhook `pre-rollout-gate` returning non-200 (used for eval staleness: if no canary eval ran in the last 24 h, fail closed).

On abort, the controller:

1. Drops the canary weight to 0 and keeps the stable version receiving 100 %.
2. Emits a Prometheus alert `ChatcatRolloutAborted` with labels `{service, reason, step}`.
3. Writes a line to the release ledger: `{status: aborted, reason, metrics_snapshot}`.

Nothing downgrades the canary pods automatically — the operator inspects first. The `rb-rollout-abort` runbook (from the `mod-108` library) takes over from there.

## 3. Validation steps

1. **Static validation of manifests.**

   ```bash
   kubectl apply --dry-run=client -f k8s/argo-rollouts/ -R
   kubectl apply --dry-run=client -f k8s/flagger/ -R
   ```

   Both must succeed. CRDs for the chosen tool must be installed.

2. **Analysis template parses.** For Argo Rollouts:

   ```bash
   kubectl argo rollouts lint -f k8s/argo-rollouts/chatcat-serving.yaml
   ```

   For Flagger, apply the manifests into a kind cluster and watch the `Canary` resource go `Initializing → Initialized` with no events of type `Warning`.

3. **Dry-run rollout in kind.**
   - Install the chosen controller + a Prometheus seeded with the `chatcat_*` metrics via a mock exporter.
   - Deploy the serving manifest with image tag `baseline`.
   - Update the image tag to `canary` and run `scripts/rollout.sh chatcat-serving canary`.
   - Walk through every weight step by inspecting the mesh / ingress config at each pause.

4. **Fault-injection.**
   - Force `chatcat_groundedness_score{canary="true"}` below baseline via the mock exporter and confirm the rollout aborts at the 5 % step without reaching 25 %.
   - Pause the eval-gate webhook and confirm the rollout aborts (fail-closed on missing eval signal).
   - Drop the canary's `canary-error-rate` to 1 % equivalent and confirm the rollout proceeds through all steps.

5. **Release-ledger record.** On a successful rollout, `scripts/rollout.sh` emits a `promotion.json` with `{service, from_image, to_image, start_ts, end_ts, aborted: false, final_weight: 100}`. Verify the record is picked up by the DORA ledger in exercise 06.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Tool choice justified *(hard gate)* | Not stated | Stated but not tied to environment | Stated and tied to one of: mesh present, Argo CD present, metric provider availability |
| 2 | Step ladder *(hard gate)* | Single step 0→100 | Linear 10/20/…/100 | Non-linear with reasoning (5/25/50/100 or equivalent) |
| 3 | SLO health gate *(hard gate)* | Missing | One metric only | Error-rate + latency + provider errors with route-SLO-anchored thresholds |
| 4 | Eval health gate *(hard gate)* | Missing | Present but not relative to baseline | Present with baseline-relative threshold and fail-closed on staleness |
| 5 | Cost gate | Missing | Present but absolute | Ratio vs. baseline with explicit bound |
| 6 | Abort behaviour | Rolls back automatically to arbitrary prior image | Drops canary weight but no alert / ledger | Drops weight, alerts, writes ledger record, hands off to runbook |
| 7 | Static validity | `kubectl apply --dry-run` fails | Succeeds but with warnings | Succeeds cleanly on an empty cluster with CRDs |
| 8 | Dry-run demonstrated | No evidence | Rollout seen on cluster but no fault injection | Rollout + at least one abort shown |
| 9 | Release-ledger emission | Missing | Emits but incomplete | Emits a full `promotion.json` consumable by exercise 06 |
| 10 | Documentation | Missing | Manifests only | Manifests + a short run-book extract naming thresholds and their owner |

## 5. Common mistakes

- **Treating a Kubernetes-side rollout as sufficient for a hosted-model change.** If the "new model" is a provider-side version bump behind the same image, the rollout controller sees nothing. The eval-gate webhook must carry the `genai.request.model` version check, not only the image SHA.
- **Only gating on 5xx.** 2xx responses with bad groundedness, bad structured-output validity, or vendor-429 masked as 2xx by a client library are the ones that burn users. Eval and provider-error gates exist for that reason.
- **Fail-open on missing signal.** A gate that passes when the eval harness is broken is worse than no gate — it ships regressions with the appearance of safety. Default to fail-closed.
- **Step ladder too fine.** `1/2/5/10/25/50/75/100` is tempting but each step spends real wall-clock time analysing; the eval-health window needs enough sample mass to be useful, which the fine ladder breaks.
- **Ignoring shared-state effects.** The prompt cache, KV cache, and retrieval index are shared across stable and canary; the 25 % step is where cache contention first shows. A canary that passes 5 % but fails 25 % is usually a cache-pressure regression, not a quality regression — but the right fix is to catch it here, not at 100 %.
- **No release-ledger record.** Without a ledger entry, the DORA metrics in exercise 06 cannot be computed. The rollout is only "done" when the record is written.

## 6. References

- Argo Rollouts, *Architecture* and *Analysis* — `argo-rollouts.readthedocs.io/en/stable/architecture/`, `argo-rollouts.readthedocs.io/en/stable/features/analysis/`.
- Flagger, *How it works*, *Metrics analysis*, and *Deployment strategies* — `docs.flagger.app/usage/how-it-works`, `docs.flagger.app/usage/metrics`, `docs.flagger.app/usage/deployment-strategies`.
- Kubernetes Ingress NGINX canary annotations — `kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary` — the ingress-level routing primitives both tools use for non-mesh traffic steering.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — the industry-recognised vocabulary for ML-specific failure modes; `ML06: ML supply chain` motivates why a canary needs a provider-error gate, and `ML09: output integrity attack` motivates a structured-output validity gate.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-techniques framework; the "ML model inference API access" technique class justifies gating on refusal-rate anomalies as a signal that abuse traffic is reaching the canary differently than the stable fleet.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the GOVERN and MANAGE functions name "monitored deployment" as a control; the canary manifest is one implementation.
