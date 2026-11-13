# Exercise 02 — Prometheus PromQL Recipes for TTFT, TPOT, ITL, and Cache Hit (Solution)

## 1. Solution overview

Give incident responders a small set of PromQL queries that read the four latency and efficiency signals every LLM service needs:

- **TTFT** — Time To First Token (end user perception of "did it hang?")
- **TPOT** — Time Per Output Token (steady-state generation speed)
- **ITL** — Inter-Token Latency (jitter during generation; related to TPOT but not identical)
- **Cache hit** — prefix cache (or KV cache / prompt cache) hit rate

All recipes assume the engine (vLLM, SGLang, TRT-LLM, llama.cpp, or an OTel-instrumented gateway) exports **Prometheus histograms** for the latency signals and **counters** for cache events. When an engine only exports gauges or summaries, the Appendix (§2.6) shows the substitutions.

Recipes follow one shape:

1. a base query (per-service p-quantile over a rolling window),
2. a per-workload-class dimension (`workload=~"chat|summarize|batch"`),
3. a burn/alerting variant.

## 2. Worked implementation (recipes)

Metric names below follow what vLLM exports by default (`vllm_*`); substitute your engine's names if different. The names are documented in the vLLM Production Metrics guide and in SGLang's /metrics endpoint. Where an engine does not expose a metric natively, the OTel semconv names from Exercise 01 are shown as the fallback (`gen_ai_*`).

### 2.1 TTFT

Base p95 over 5 minutes:

```promql
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(vllm_time_to_first_token_seconds_bucket[5m])
  )
)
```

Per workload class (requires engine to label histograms by `workload` or a relabel rule to inject one from the ingress):

```promql
histogram_quantile(
  0.95,
  sum by (le, workload) (
    rate(vllm_time_to_first_token_seconds_bucket{service="llm-gateway"}[5m])
  )
)
```

OTel-semconv fallback (engine does not natively expose TTFT, but you emit it as a histogram named `gen_ai.client.time_to_first_token`):

```promql
histogram_quantile(
  0.95,
  sum by (le, gen_ai_request_model) (
    rate(gen_ai_client_time_to_first_token_seconds_bucket[5m])
  )
)
```

### 2.2 TPOT (time per output token, steady state)

TPOT is reported directly by vLLM as `vllm_time_per_output_token_seconds` (histogram, seconds-per-token). Reading it:

```promql
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(vllm_time_per_output_token_seconds_bucket[5m])
  )
)
```

Derived TPOT when the engine only exports total generation time and output tokens:

```promql
# Average over a window: total gen time / total output tokens
sum by (service) (rate(vllm_e2e_request_latency_seconds_sum[5m]))
/
sum by (service) (rate(vllm_generation_tokens_total[5m]))
```

Caveat: the derived form is a **window average**, not a true quantile. Use the histogram form for SLOs; use the derived form only when the histogram is unavailable.

### 2.3 ITL (inter-token latency)

ITL is the per-token gap between successive emissions during generation. Where the engine exports a dedicated histogram (vLLM 0.6+ exposes `vllm_inter_token_latency_seconds_bucket`, available alongside TPOT), read it directly:

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(vllm_inter_token_latency_seconds_bucket[5m])
  )
)
```

> Note: confirm the `vllm_inter_token_latency_seconds` metric name against your current vLLM release; some versions expose only `vllm_time_per_output_token_seconds`.

If only TPOT is available, use it as the ITL proxy while noting the limitation (TPOT is a *per-token average over a request*; true ITL is *per gap*).

Jitter alert — p99 ITL to p50 ratio as a stall detector:

```promql
(
  histogram_quantile(0.99, sum by (le, service) (rate(vllm_inter_token_latency_seconds_bucket[5m])))
  /
  histogram_quantile(0.50, sum by (le, service) (rate(vllm_inter_token_latency_seconds_bucket[5m])))
) > 5
```

### 2.4 Cache hit rate (prefix / prompt cache)

vLLM exposes prefix cache hits and queries as counters. Hit rate over a 5-minute window:

```promql
sum(rate(vllm_prefix_cache_hits_total[5m]))
/
sum(rate(vllm_prefix_cache_queries_total[5m]))
```

> Note: verify current vLLM prefix-cache metric names against the release in use; older releases exposed `vllm_cache_hit_rate` as a gauge, and SGLang publishes radix cache hit metrics under a different name.

Generic form when the engine only exports `*_tokens_hit` and `*_tokens_total`:

```promql
sum by (service) (rate(prefix_cache_tokens_hit_total[5m]))
/
sum by (service) (rate(prefix_cache_tokens_total[5m]))
```

Dashboard tip: display cache hit rate next to TTFT. A drop in cache hit rate typically shows up as a TTFT regression first; the pair is diagnostic.

### 2.5 Burn-rate variants (for alerting)

Multi-window multi-burn — pair a 5-minute fast window with a 1-hour slow window on the TTFT SLO. Replace `0.5` with your SLO target in seconds.

```promql
# Fast burn: 1h observed vs SLO, AND 5m observed vs SLO, both breaching
(
  histogram_quantile(0.95, sum by (le) (rate(vllm_time_to_first_token_seconds_bucket[1h]))) > 0.5
)
and
(
  histogram_quantile(0.95, sum by (le) (rate(vllm_time_to_first_token_seconds_bucket[5m]))) > 0.5
)
```

Full burn-rate math (good-event ratio) depends on how the SLO is modeled — see mod-103 for the companion SLO authoring. The queries above are the raw-latency variant and work as "is the p95 blown right now" alerts.

### 2.6 When the engine only exports gauges or summaries

- **Gauge of current latency** (not a histogram): use `avg_over_time(vllm_current_ttft_seconds[5m])` or a `quantile_over_time` as a weak substitute. Call it out in the dashboard — gauges cannot give you real quantiles.
- **Prometheus summary type** (pre-aggregated quantiles): read via `vllm_time_to_first_token_seconds{quantile="0.95"}`. These **cannot be aggregated across instances** correctly, so do not sum or average them. Scrape per-instance and render per-instance series.

### 2.7 Recording rules

For dashboard performance, pre-compute the p95 series as recording rules:

```yaml
# prometheus/rules/llm.yml
groups:
  - name: llm.recording
    interval: 30s
    rules:
      - record: service:ttft_p95:5m
        expr: |
          histogram_quantile(
            0.95,
            sum by (le, service) (rate(vllm_time_to_first_token_seconds_bucket[5m]))
          )
      - record: service:tpot_p95:5m
        expr: |
          histogram_quantile(
            0.95,
            sum by (le, service) (rate(vllm_time_per_output_token_seconds_bucket[5m]))
          )
      - record: service:prefix_cache_hit_rate:5m
        expr: |
          sum by (service) (rate(vllm_prefix_cache_hits_total[5m]))
          /
          sum by (service) (rate(vllm_prefix_cache_queries_total[5m]))
```

## 3. Validation steps

1. **Metric presence check**. For each engine instance, `curl :<port>/metrics | grep -E 'ttft|time_to_first|time_per_output|prefix_cache'`. Confirm metric names exist before copy-pasting queries.
2. **Histogram bucket coverage**. Query `histogram_quantile(0.999, sum by (le) (rate(<metric>_bucket[5m])))` — if it equals your largest `le` bucket's boundary, your buckets are underpowered; add buckets above that boundary. For TTFT that typically means buckets up to 60s.
3. **Workload label plumbing**. Run `sum by (workload) (rate(<metric>_bucket[5m]))` and confirm every workload class you care about shows up. If `workload` is empty, your ingress relabeling is wrong.
4. **Recording-rule sanity**. After deploying the rules, compare `service:ttft_p95:5m` to the on-the-fly query over the same window; they should match within rounding.
5. **Cache-hit cross-check**. In a load test with a fixed prompt prefix, hit rate should asymptote toward 1.0 as warm-up completes. If it stays near 0, the engine's prefix cache is off or the metric name is wrong.
6. **Negative test**. Pause scraping for 2 minutes, then query p95 over `[5m]`. The result should go stale (or be `NaN`) — not silently return zeros. If it returns zero, your dashboards will lie during outages.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| TTFT recipe | Uses histogram_quantile on `_bucket` series with `sum by (le, ...)` aggregation |
| TPOT recipe | Prefers native engine histogram; the derived average form is clearly labeled as an average |
| ITL recipe | Either native histogram or an explicit TPOT-as-proxy note |
| Cache hit | Counter rate ratio form; not a raw gauge |
| Dimensions | Per-workload class breakout provided, not just per-service |
| Burn/alert | At least one multi-window (fast + slow) variant |
| Recording rules | Expensive quantiles pre-computed; interval set explicitly |
| Fallbacks | Gauge/summary substitutions called out with their limits |

## 5. Common mistakes

- **Averaging quantiles across instances.** Prometheus `summary` quantiles cannot be aggregated with `sum` or `avg`. Only histograms can. A `sum(vllm_ttft{quantile="0.95"})` is always wrong.
- **`rate()` without `_bucket`.** `rate(histogram_metric[5m])` on the raw metric name is a type error; you want `_bucket`.
- **Forgetting the `le` label in the `sum by ()`.** Without it, `histogram_quantile` has nothing to compute against. The pattern is always `sum by (le, <other dims>) (rate(..._bucket[5m]))`.
- **Using `_count` and `_sum` to derive quantiles.** That gives you an average, not a quantile. Call it an average in the dashboard title if you must use it.
- **Short windows for p99.** A `[1m]` window at low QPS has insufficient samples for a stable p99. Use `[5m]` minimum for p95, `[15m]`–`[30m]` for p99 on low-QPS services.
- **Unbounded label cardinality.** Labeling histograms with `request_id`, `user_id`, or `prompt_hash` will blow out Prometheus memory. Keep `workload`, `model`, `service`, `env` only.
- **Alerting on raw p95 breaching once.** Pair with a window or burn rate; raw breach fires on single-request outliers.

## 6. References

- Prometheus Querying Basics (histograms and `histogram_quantile`): https://prometheus.io/docs/prometheus/latest/querying/basics/
- Prometheus histograms best practices: https://prometheus.io/docs/practices/histograms/
- Prometheus recording rules: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/
- vLLM Production Metrics (metric names, histograms exposed): https://docs.vllm.ai/en/latest/serving/metrics.html
- SGLang server metrics: https://docs.sglang.ai/
- OpenTelemetry GenAI metrics semconv (fallback metric names): https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/
- Multi-window multi-burn alerting (SRE Workbook, Chapter 5 — "Alerting on SLOs"): https://sre.google/workbook/alerting-on-slos/
