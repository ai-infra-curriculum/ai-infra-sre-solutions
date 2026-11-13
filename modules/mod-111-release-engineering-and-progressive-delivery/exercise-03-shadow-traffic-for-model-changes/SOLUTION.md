# Exercise 03 — Shadow Traffic for Model Changes — Reference Solution

Paired learning file: `lessons/mod-111-release-engineering-and-progressive-delivery/exercises/exercise-03-shadow-traffic-for-model-changes.md`.

> **Scope note.** The paired exercise asks the learner to design a shadow-traffic (a.k.a. "mirror traffic", "dark launch") setup that routes production requests to a candidate model in parallel to the stable model without returning the candidate's response to the user, and to decide what to measure, what to persist, and what to do about the parts of a model request that cannot be shadowed safely. The reference solution uses the illustrative `chatcat` RAG-chat surface.

## 1. Solution overview

Shadow traffic for a classical microservice is a mostly-solved problem: duplicate the request, discard the response, and compare observability. For a model-backed service the shape is different in four ways:

1. **Side effects cannot be shadowed naïvely.** Tool calls, retrieval-index writes, downstream mutations (payments, email, memory writes), and any upstream vendor billing are not idempotent on the shadow path. These must be stubbed, dry-run, or elided.
2. **Cost is real.** A hosted-model call on the shadow path incurs the same per-token bill as a production call. An unbounded shadow pipeline doubles vendor cost. The solution must include rate controls.
3. **PII and policy constraints follow the shadowed request.** A user whose data flows to the stable model is not automatically a user whose data flows to the candidate. Shadowing a new provider needs the same DPA / processor coverage.
4. **"Equivalent" needs definition.** Even without a user seeing the response, the shadow must compare outputs on a metric that is meaningful for *this* model change: a tokenizer-change shadow cares about latency; a prompt-change shadow cares about groundedness; a provider swap cares about both.

The reference design has three components:

- **Mirror layer** — a gateway-level request mirror that runs a bounded fraction of traffic against the candidate stack, strips mutating side effects, and tags the mirrored request so its billing and log trail are separable.
- **Compare pipeline** — reuses the exercise-02 response-diff harness, but against production-shadow pairs rather than a fixture set. Runs continuously, not only at release-time.
- **Promotion gate** — a scheduled summary over the shadow comparison results that decides whether the candidate is eligible for a canary rollout (exercise 01) or further shadow soak.

Decision rationale:

- **Mirror at the gateway, not at the application.** The gateway owns authentication, rate limiting, and request tagging; it is the correct layer to clone the request and discard the clone's response. Most service meshes support traffic mirroring natively ([Envoy — Request Mirror Policy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-routeaction-requestmirrorpolicy); [Istio — Traffic Mirroring](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)).
- **Discard the response at the mirror, not at the model.** The mirror receives the response from the candidate stack and drops it; the model should see no difference from a real request. This keeps the shadow faithful.
- **Explicit side-effect stubs.** The candidate stack runs with a config flag `SHADOW=1` that replaces tool-call executors with no-ops that return a recorded transcript, replaces retrieval-index writers with reads, and replaces external mutators with log-only sinks.

## 2. Implementation — worked answer

### 2.1 Architecture

```
                     +--------------------+
     user request -->| chatcat-gateway    |
                     |  (Envoy / Istio)   |
                     +---------+----------+
                               |
             +-----------------+-----------------+
             |                                   |
             v                                   v
     chatcat-serving-stable            chatcat-serving-shadow
         (vLLM, normal)                 (vLLM, SHADOW=1)
             |                                   |
             v                                   v
       user response                   response -> drop
                                               \
                                                v
                                       shadow-recorder
                                       (records shadow + stable
                                        pair; writes to S3 and
                                        enqueues diff job)
```

### 2.2 Istio shadow routing

`k8s/istio/shadow-routing.yaml`:

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
            subset: stable
          weight: 100
      mirror:
        host: chatcat-serving
        subset: shadow
      mirrorPercentage:
        value: 5.0  # <-- shadow fraction; start small
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: chatcat-serving
  namespace: chatcat
spec:
  host: chatcat-serving
  subsets:
    - name: stable
      labels:
        version: stable
    - name: shadow
      labels:
        version: shadow
```

### 2.3 Side-effect stubs — `SHADOW=1` behaviour

The candidate stack ships with a single feature flag. When `SHADOW=1`:

| Subsystem | Normal behaviour | Shadow behaviour |
|-----------|------------------|------------------|
| Tool-call executor | Executes the tool and returns the result | Returns a recorded stub if the tool call matches a recorded call; otherwise returns `{"stub": true, "tool_name": …, "args": …}` to the model and records a "novel tool call" event |
| Retrieval index writer | Inserts into the index | Logs the insert; writes nothing |
| Chat-memory writer | Writes to the user's memory | Logs the write; writes nothing |
| External HTTP side-effect (email, payment, ticket creation) | Executes | Logs the call; writes nothing |
| Model API call | Live call to the provider (billable) | Live call to the provider (billable) — the model call *cannot* be shadowed; see §2.4 |
| Billing / usage counters | Increments | Increments under a separate meter (`shadow=true`) so the bill is attributable but not accidentally conflated with production |

### 2.4 What cannot be shadowed

- **The provider call itself.** You cannot "dry-run" a hosted LLM call and get a real output. Shadowing a new provider doubles provider cost by design; capacity planning (`mod-106`) must size for the shadow fraction.
- **Non-deterministic tool chains.** A tool whose output depends on wall-clock time, external state, or a random seed cannot have its output stubbed from a recording. The design records the stubbed call and marks the shadow pair as `incomparable` for prompts that hit a novel tool call; those pairs flow to a separate "needs deterministic recording" bucket.
- **Prompts that require writing memory for downstream turns.** Multi-turn flows that depend on side-effect writes degrade on the shadow path. For those, shadow only single-turn starts.

### 2.5 Shadow recorder

The recorder lives as a sidecar on the shadow pods. It receives the mirrored response (via a local socket), pairs it with the stable response (re-fetched from the audit log by trace ID), and writes a record to object storage. A Kafka-consumer later enqueues diff jobs.

`shadow_recorder.py`:

```python
import gzip
import json
import time
from pathlib import Path
from typing import Any


class ShadowRecorder:
    def __init__(self, sink_dir: Path, enqueue):
        self.sink_dir = sink_dir
        self.enqueue = enqueue

    def record(self, trace_id: str, request: dict[str, Any], shadow_response: dict[str, Any], stable_response: dict[str, Any]) -> None:
        payload = {
            "trace_id": trace_id,
            "recorded_at": time.time(),
            "route": request.get("route"),
            "candidate_model": shadow_response.get("model"),
            "stable_model": stable_response.get("model"),
            "request": request,
            "shadow": shadow_response,
            "stable": stable_response,
            "shadow_tool_calls_stubbed": shadow_response.get("stubbed_tool_calls", []),
        }
        key = f"{request.get('route', 'default')}/{trace_id}.json.gz"
        path = self.sink_dir / key
        path.parent.mkdir(parents=True, exist_ok=True)
        with gzip.open(path, "wb") as fh:
            fh.write(json.dumps(payload).encode("utf-8"))
        self.enqueue({"key": str(path), "trace_id": trace_id})
```

### 2.6 Compare pipeline

A worker reads from the queue, loads the pair, runs it through the exercise-02 diff harness, and writes a per-record verdict with the three-layer structure. Daily aggregation writes a route × verdict breakdown to a Postgres table `shadow_summaries`.

```python
from diff.aggregate import _score_pair
from diff.semantic import Embedder, Judge


def process_pair(pair: dict, embedder: Embedder, judge: Judge) -> dict:
    prompt_row = {
        "id": pair["trace_id"],
        "route": pair["route"],
        "kind": _infer_kind(pair),
        "deterministic": pair["request"].get("deterministic", False),
        "safety_adjacent": pair["request"].get("safety_adjacent", False),
        "prompt": pair["request"]["prompt"],
    }
    r = _score_pair(prompt_row, pair["stable"]["text"], pair["shadow"]["text"], embedder, judge)
    return {
        "trace_id": pair["trace_id"],
        "route": prompt_row["route"],
        "verdict": r.verdict,
        "cosine": r.semantic_cosine,
        "entity_jaccard": r.semantic_jaccard,
        "judge_equivalent": r.semantic_judge,
        "policy_direction": r.policy_direction,
        "candidate_model": pair["candidate_model"],
    }


def _infer_kind(pair: dict) -> str:
    content_type = pair["request"].get("expected_response_type")
    if content_type == "json":
        return "structured"
    if content_type == "number":
        return "numeric"
    return "freeform"
```

### 2.7 Promotion gate

A daily cron runs:

```sql
SELECT
  route,
  count(*)                                                         AS total,
  count(*) FILTER (WHERE verdict = 'equivalent')                   AS equiv,
  count(*) FILTER (WHERE verdict = 'quality-regression')           AS quality_reg,
  count(*) FILTER (WHERE verdict = 'safety-regression')            AS safety_reg,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY cosine)             AS cosine_p95
FROM shadow_summaries
WHERE candidate_model = :candidate
  AND recorded_at >= now() - INTERVAL '24 hours'
GROUP BY route;
```

The candidate is promotion-eligible iff every route has:

- `safety_reg = 0`
- `quality_reg / total <= route.budget` (same thresholds used by exercise-02 release gate)
- `total >= route.min_sample` (the minimum shadow sample required for a route-level verdict to be meaningful; typical default: 500 pairs per route)
- at least 24 h continuous shadow uptime

A candidate that passes the gate is tagged `shadow-green` in the release ledger and enters the exercise-01 canary pipeline. A candidate that fails on quality metrics is kept in shadow; a candidate that fails on safety metrics is detagged and the owning team is paged.

### 2.8 Operational guardrails

- **Shadow fraction ramps slowly.** Start at 1 %, double daily until 5 %, hold. Full-traffic shadow is almost never worth the cost; 5 % at 24 h gives enough sample to power a route-level verdict on most routes.
- **Shadow cost SLO.** Shadow pipeline cost per day must stay under a published budget. The gateway exposes a kill switch that drops shadow percentage to 0 if the daily budget is exceeded.
- **PII scope.** Routes that carry regulated data (health, finance, child-safety) must be explicitly allow-listed for shadowing to a new provider; default is deny. The gateway tag `pii.tier=…` gates the `mirror` policy.
- **Observability.** Dashboards: `shadow / coverage` (shadow fraction over time per route), `shadow / cost` (billable tokens per hour under `shadow=true`), `shadow / verdicts` (daily rollup of verdicts per route).

## 3. Validation steps

1. **Mirror is configured but not user-impacting.** Deploy the Istio manifest and send one request. The stable version returns to the client; the shadow pod logs show it received the request and discarded the response. Latency of the stable response does not depend on the shadow pod's health (Envoy mirror is fire-and-forget by default).

2. **Side-effect stubs exercised.** Send a request whose handler normally writes to memory. Verify the stable pod wrote the memory record and the shadow pod *did not* (database row count unchanged beyond the stable write).

3. **Shadow pair recorded.** Send 10 requests. 10 pair records exist in object storage with matching `trace_id`s. The diff worker processes all 10 within the queue SLA.

4. **Cost tagging.** Hit the provider under `SHADOW=1`. The usage report in the provider dashboard (or the cost-attribution column in the usage counter) marks those calls as `shadow=true`.

5. **Kill switch.** Trigger the kill switch by posting to `/admin/shadow-fraction` with value 0. Verify the Istio `VirtualService` is reconciled to `mirrorPercentage: 0` and no new pairs arrive within 5 minutes.

6. **Novel tool-call handling.** Send a request that triggers a tool call the shadow stub has no recording for. Verify the shadow pod returns the "novel tool call" stub to the model, the pair is marked `incomparable`, and it is excluded from the promotion gate's aggregates.

7. **Scope-controlled routing.** Send a request tagged with a PII tier that is not allow-listed for shadowing. Verify the mirror does not fire. The stable path returns normally.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Routing layer *(hard gate)* | Applied in the app | Applied in a mesh but synchronously | Applied in a mesh in fire-and-forget mode |
| 2 | Side-effect handling *(hard gate)* | Not addressed | Addressed as "disable writes" without enumeration | Explicit per-subsystem stub (tool, index, memory, external, billing) |
| 3 | Shadow fraction controls | Full mirror | Fixed fraction with no kill switch | Ramped fraction + kill switch + daily cost SLO |
| 4 | PII / policy scope *(hard gate)* | Not addressed | Addressed as policy prose only | Enforced by the gateway (tag-based allow-list) |
| 5 | Pair persistence | None | Logs only | Object storage with trace-ID pairing and retention policy |
| 6 | Comparison method | Exact string | Single-signal semantic | Reuses the three-layer diff from exercise 02 |
| 7 | "Incomparable" bucket | Missing | Present but not reasoned about | Present, with explicit rules (novel tool call, multi-turn start, stochastic tool) and excluded from the promotion gate |
| 8 | Promotion gate | Missing | Threshold on one metric | Multi-route, safety-first, min-sample gate |
| 9 | Observability | None | Two dashboards | Three dashboards (coverage, cost, verdicts) + alerts |
| 10 | Operational handoff to exercise 01 | Missing | Mentioned in prose | Candidate tag in the release ledger consumed by exercise-01's pre-rollout webhook |

## 5. Common mistakes

- **Synchronous mirroring.** A synchronous mirror blocks user responses on shadow-pod health. Envoy's mirror is fire-and-forget by design; preserving that property is the whole reason the mirror lives at the gateway.
- **Shadowing writes unconditionally.** Running the shadow stack with the same side-effect writes as the stable stack causes double tool calls, double memory writes, duplicate emails, and sometimes doubled charges. The stubs are not optional.
- **Assuming a 100 % shadow replay.** Full-replay doubles vendor cost and rarely produces more signal than a sampled shadow at a well-chosen fraction. A sampled shadow with per-route minimum sample sizes is the right design.
- **Letting the shadow contaminate production telemetry.** Shadow responses must not land in production logs or metrics — only the dedicated shadow-recorder. Mixing them produces false quality signals on the stable path.
- **Promoting from shadow without an incomparable bucket.** Prompts that exercise novel tool calls will be falsely counted as "equivalent" if the shadow returned the stub and the pair nonetheless embeds close. The `incomparable` flag is what prevents that.
- **No cost budget.** Shadowing a hosted-model change at 5 % with a 10 M-request/day product spends 500 K additional model calls per day. Without a budget and a kill switch, the finance team finds the shadow before the owners do.
- **Treating "passed shadow" as equivalent to "passed canary".** Shadow traffic doesn't observe real user reactions (regen, abandon, refund, support ticket). Shadow is a *gate on the way to* canary, not a replacement.

## 6. References

- Envoy proxy, *Request Mirror Policy* — [envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-routeaction-requestmirrorpolicy](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-routeaction-requestmirrorpolicy) — the fire-and-forget semantics the solution depends on.
- Istio, *Traffic Mirroring* — [istio.io/latest/docs/tasks/traffic-management/mirroring/](https://istio.io/latest/docs/tasks/traffic-management/mirroring/) — the manifest shape in §2.2 is drawn from this page.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML05: Model Theft` and `ML08: Model Skewing` motivate the PII / scope guardrails on which routes may be shadowed to a new provider.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "Exfiltrate via ML Inference API" technique motivates the separable shadow billing meter (so unusual provider traffic on the shadow path shows up distinctly).
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `MAP` and `MEASURE` functions name "pre-deployment testing in production-like environments" as a control; shadow traffic is the canonical implementation.
