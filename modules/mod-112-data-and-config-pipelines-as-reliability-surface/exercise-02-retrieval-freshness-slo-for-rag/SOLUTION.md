# Exercise 02 — Retrieval-Freshness SLO for RAG — Reference Solution

Paired learning file: `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/exercises/exercise-02-retrieval-freshness-slo-for-rag.md`.

> **Scope note.** The paired exercise asks the learner to treat the recency of the RAG corpus as a user-facing reliability property and to define, measure, and alert against a *retrieval-freshness SLO* — not just an index rebuild job's success rate. The reference solution produces (a) the SLO definition and its SLI arithmetic, (b) the instrumentation that produces the SLI, (c) the error budget and alerting policy, and (d) the runbook and corrective-action primitives when the budget is burning. The illustrative product is the fabricated `chatcat` RAG surface used across this module family.

## 1. Solution overview

The reliability property users feel in a RAG system is not "the index rebuild job finished green". It is "the documents in the response are current enough that the answer reflects reality today". Those are different measurements, and conflating them is how an organisation can have a 100 %-green pipeline and a daily stream of user escalations about stale answers.

The SLO target of this exercise is therefore phrased from the *answer*, not from the *job*: of the chunks that influenced an answer in production, what fraction were "fresh enough" by the business rule for this route? This is the control named in the NIST AI Risk Management Framework `MEASURE 2.3` ("AI system performance or assurance criteria are measured qualitatively or quantitatively and demonstrated for conditions similar to deployment setting(s)") — the production signal has to measure the *as-deployed* behaviour, not an upstream proxy.

### 1.1 Freshness is a *route-level* contract

Different routes have different tolerance for staleness. A FAQ-style assistant pulling from product docs can tolerate a day; a price-sensitive routing assistant pulling from pricing tables cannot tolerate more than minutes; a historical lookup cannot be evaluated by recency at all. The design puts the "fresh enough" rule on the route config, not on the index:

```yaml
# chatcat/routes/faq.yaml
route: chatcat/faq
retrieval:
  index: chatcat-docs-v3
  k: 6
  freshness:
    max_age: 48h               # a chunk older than 48h counts as stale
    source_timestamp_field: doc_updated_at
    exempt_source_classes:     # evergreen content excluded from the SLI
      - legal-terms
      - product-changelog
```

The SLI then evaluates against this per-route rule. Two routes against the same index can have different SLO verdicts, and that is correct.

### 1.2 SLI definition

Define the SLI as the fraction of production answers in the window whose retrieved chunks were all "fresh enough":

- **Numerator:** count of answers in window `W` for route `R` where, among the `k` retrieved chunks that were passed to the generation step, none were older than `max_age` **and** of non-exempt source class.
- **Denominator:** total answers in window `W` for route `R` where retrieval actually ran (exclude answers served from a cache / early-return / refusal).

A conservative version counts the answer as unfresh if *any one* chunk is stale. A looser version counts it as unfresh only when a majority are stale. The reference uses the conservative version because an LLM can lean heavily on any one of the retrieved chunks and a single stale chunk is enough to produce a wrong answer; the looser variant is fine for very large `k` where majority voting is realistic.

### 1.3 SLO target and error budget

- **Target.** `chatcat/faq` serves 99 % fresh answers on a 28-day rolling window.
- **Error budget.** 1 % of answers per 28 days may be unfresh (≈ 302 unfresh answers per 30 000 answers / day).
- **Alerting.** Multi-window burn-rate alerts per the Google SRE workbook recipe: a fast burn (14.4× for 1 h and 5 min windows) pages; a slow burn (6× for 6 h and 30 min windows) tickets. These burn-rate multipliers are the ones the workbook recommends for a 99 % objective with a 30-day budget.

The budget is not negotiable with the index-rebuild job's uptime. If the job runs green but the fetcher upstream is lagging, the SLO burns and we investigate; the job's "green" is not sufficient evidence to silence the alert.

## 2. Implementation — worked answer

### 2.1 Chunk-level timestamp hygiene

Every chunk in the index carries two timestamps and a source class:

- `source_timestamp`: the time the *source document* was authored or last authoritatively changed in the system of record.
- `ingest_timestamp`: the time our pipeline embedded the chunk.
- `source_class`: a free-form label used by exemption rules.

The SLI uses `source_timestamp` for freshness. The `ingest_timestamp` is used by the *index-staleness* secondary signal (§2.5) to distinguish "our pipeline is behind" from "the upstream doc has not changed in a while".

Pipelines that overwrite `source_timestamp` with "now" at ingest time are a common bug — the chunk looks always-fresh in the SLI, even when upstream has not updated it in weeks. The ingest job enforces: if the source carries a `updated_at` field, use it; otherwise mark `source_timestamp_missing=true` and the SLI treats the chunk as unfresh (fail closed).

### 2.2 Serving-side instrumentation

The serving process emits, per request, a span attribute set under the OpenTelemetry GenAI semantic conventions:

- `genai.retrieval.chunk_count` — total retrieved chunks.
- `genai.retrieval.max_age_s` — the max of `now - source_timestamp` across non-exempt retrieved chunks, in seconds.
- `genai.retrieval.all_fresh` — boolean, computed server-side, so downstream can aggregate without re-deriving.

And a Prometheus counter / histogram pair:

```python
from prometheus_client import Counter, Histogram

RETRIEVED_ANSWERS = Counter(
    "chatcat_retrieval_answers_total",
    "Answers served where retrieval actually ran.",
    ["route", "fresh"],
)

RETRIEVAL_MAX_AGE = Histogram(
    "chatcat_retrieval_max_age_seconds",
    "Max age of a non-exempt retrieved chunk, per answer.",
    ["route"],
    buckets=(60, 300, 900, 3600, 6*3600, 12*3600, 24*3600, 48*3600, 7*24*3600),
)


def record_retrieval_outcome(route: str, chunks, route_cfg) -> None:
    import time
    now = time.time()
    max_age = 0.0
    all_fresh = True
    for c in chunks:
        if c.source_class in route_cfg.exempt_source_classes:
            continue
        if c.source_timestamp is None:
            all_fresh = False
            continue
        age = now - c.source_timestamp
        if age > max_age:
            max_age = age
        if age > route_cfg.max_age_s:
            all_fresh = False
    RETRIEVAL_MAX_AGE.labels(route).observe(max_age)
    RETRIEVED_ANSWERS.labels(route, "true" if all_fresh else "false").inc()
```

Cache-hit and early-return paths do not call `record_retrieval_outcome`. Refusals likewise skip it. This keeps the SLI denominator clean.

### 2.3 PromQL SLI and burn-rate alerts

`prometheus/rules/freshness-slo.yaml`:

```yaml
groups:
  - name: chatcat-freshness-slo
    interval: 30s
    rules:
      - record: chatcat:retrieval_fresh_ratio:5m
        expr: |
          sum by (route) (rate(chatcat_retrieval_answers_total{fresh="true"}[5m]))
          /
          sum by (route) (rate(chatcat_retrieval_answers_total[5m]))

      - record: chatcat:retrieval_fresh_ratio:1h
        expr: |
          sum by (route) (rate(chatcat_retrieval_answers_total{fresh="true"}[1h]))
          /
          sum by (route) (rate(chatcat_retrieval_answers_total[1h]))

      - record: chatcat:retrieval_fresh_ratio:6h
        expr: |
          sum by (route) (rate(chatcat_retrieval_answers_total{fresh="true"}[6h]))
          /
          sum by (route) (rate(chatcat_retrieval_answers_total[6h]))

      - alert: ChatcatFreshnessFastBurn
        expr: |
          (1 - chatcat:retrieval_fresh_ratio:1h) > (14.4 * 0.01)
          and
          (1 - chatcat:retrieval_fresh_ratio:5m) > (14.4 * 0.01)
        for: 2m
        labels:
          severity: page
          slo: chatcat-freshness
        annotations:
          summary: "chatcat retrieval freshness burning fast ({{ $labels.route }})"
          runbook: https://runbooks.example/rb-retrieval-freshness-burn

      - alert: ChatcatFreshnessSlowBurn
        expr: |
          (1 - chatcat:retrieval_fresh_ratio:6h) > (6.0 * 0.01)
          and
          (1 - chatcat:retrieval_fresh_ratio:1h) > (6.0 * 0.01)
        for: 15m
        labels:
          severity: ticket
          slo: chatcat-freshness
        annotations:
          summary: "chatcat retrieval freshness burning slow ({{ $labels.route }})"
          runbook: https://runbooks.example/rb-retrieval-freshness-burn
```

The burn-rate multipliers (14.4× for the fast alert, 6× for the slow) are the Google SRE workbook recipe for a 99 % objective; see `sre.google/workbook/alerting-on-slos/`.

### 2.4 Index-staleness secondary signal

A freshness SLO breach can have two very different root causes: the *pipeline is behind* (ingest lag) or the *upstream has gone quiet* (nobody is updating docs). The Prometheus-side secondary signal separates the two:

```yaml
- record: chatcat:ingest_lag_seconds:p95
  expr: |
    histogram_quantile(0.95,
      sum by (le) (rate(chatcat_index_ingest_lag_seconds_bucket[1h]))
    )
```

`chatcat_index_ingest_lag_seconds_bucket` is emitted by the ingest job: for each chunk written, it records `ingest_timestamp - source_timestamp`. If the p95 ingest lag is low but the freshness SLI is burning, the pipeline is on time and the upstream is quiet — different fix.

### 2.5 Corrective-action primitives

When the alert fires, three operations are available to the on-call:

1. **`retrievalctl rebuild-index --index <idx>`** — triggers a full rebuild from the system of record, bypassing the usual watermark. Safe but slow.
2. **`retrievalctl backfill --since <ts>`** — triggers an incremental ingest from `<ts>`. Used when the ingest job is stuck on a specific document class.
3. **`retrievalctl set-route-tolerance --route <r> --max-age <d>`** — temporarily widens the freshness window on a specific route. Written to a tolerance-override table with an auto-expiry. Audit-logged; expires in 24 h.

Op #3 is the one SRE will reach for when burn-rate is from a known-good source that just slowed down (holiday weekend, planned source migration). It is deliberately auto-expiring so a widened tolerance never becomes permanent by accident.

### 2.6 Runbook extract — `rb-retrieval-freshness-burn`

```markdown
# rb-retrieval-freshness-burn

## When
- A `ChatcatFreshness{Fast,Slow}Burn` alert fires.

## Triage (first 5 minutes)
1. Identify the route from the alert labels.
2. Open the `Retrieval freshness` dashboard; note the current fresh ratio, ingest lag p95, and the `source_timestamp_missing` rate.
3. Decide which lane:
   - Ingest lag p95 is above its own alert threshold → **lane A (pipeline is behind)**.
   - Ingest lag p95 is fine, `source_timestamp_missing` rate is normal → **lane B (upstream is quiet)**.
   - `source_timestamp_missing` rate is above baseline → **lane C (schema drift at the source)**.

## Lane A — pipeline is behind
1. `retrievalctl status --index <idx>` and identify the worker lagging.
2. If the worker is healthy but the queue is deep, scale the workers (runbook `rb-index-worker-scale`).
3. If the worker is unhealthy, follow `rb-index-worker-recovery`.
4. Expect SLO to recover within one max-age window of fixing the lag.

## Lane B — upstream is quiet
1. Confirm with the owning team that the source is in fact quiet (holiday, migration, deliberate freeze).
2. If this is expected and time-limited: `retrievalctl set-route-tolerance --route <r> --max-age <d>` for the known duration.
3. If unexpected: open a joint incident with the source owners; freshness is a shared SLO.

## Lane C — schema drift at the source
1. This is the case where the source team has renamed or removed the timestamp field.
2. Do NOT widen tolerance; the signal is now missing, not merely stale.
3. Hand off to the feature-drift runbook (exercise 03) — the pattern is the same.
```

## 3. Validation steps

1. **Unit tests on the SLI computation.** Feed the `record_retrieval_outcome` function fixtures with: all fresh, one stale, one missing `source_timestamp`, all-exempt, mixed. Each case must land in the expected counter buckets.
2. **PromQL recording-rule evaluation.** Use `promtool test rules prometheus/rules/freshness-slo.yaml prometheus/tests/freshness-slo-tests.yaml` to drive the recording rules through a timeline of synthetic observations and assert the fast / slow alerts fire at the expected timestamps.
3. **End-to-end in a kind cluster.** Deploy an ingest worker with artificial lag, deploy a serving process, drive traffic. Assert the `chatcat:retrieval_fresh_ratio:1h` series drops below `1 - 0.01` after the 48-h fixture clock advances past `max_age`, and that the fast-burn alert fires within two minutes of the drop.
4. **Runbook step-walk.** On-call rotates through the three lanes using the `retrievalctl` primitives against a fixture index. Each lane completes in under 15 minutes with the SLI recovering.
5. **Audit of exempt classes.** `retrievalctl exempt-report --route chatcat/faq --days 30` shows the number of answers dropped from the SLI denominator because of exempt source classes. If this number is > 10 % the exemption list has outgrown its purpose; flag for review.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | SLI defined per route *(hard gate)* | Single global SLI | Per route but same rule | Per route with route-specific `max_age` and exemption list |
| 2 | SLI measured on answers, not jobs *(hard gate)* | Measures the ingest job's success | Measures index age | Measures retrieved-chunk age per served answer |
| 3 | Timestamps come from the source *(hard gate)* | Uses `ingest_timestamp` as if it were source | Uses source but falls back to ingest silently | Uses source, fails closed on missing, logs separately |
| 4 | Burn-rate alerting *(hard gate)* | Static threshold alerts | Single-window burn rate | Multi-window multi-burn-rate per SRE workbook recipe |
| 5 | Denominator discipline | Everything counted | Cache hits excluded | Cache hits, refusals, and retrieval-skipped answers all excluded |
| 6 | Secondary signal for root cause | Missing | Ingest lag present but unused | Ingest lag present and used to split lanes in the runbook |
| 7 | Corrective-action primitives | None | Rebuild only | Rebuild + backfill + auto-expiring tolerance override |
| 8 | Runbook present | Missing | Named but not stepwise | Stepwise with three lanes and handoff rules |
| 9 | Auditing of exemptions | None | Exempt list exists but never reviewed | Exempt list has periodic review and report |
| 10 | Integration with upstream modules | Isolated | Names one | Names `mod-104 / exercise-01` (OTel GenAI attrs) and `mod-108` runbook library |

## 5. Common mistakes

- **Counting "the job ran green" as the SLI.** The job can run green on a schedule while the upstream fetcher is lagging; the user-facing freshness is unchanged by the job's success. Measure the served answers.
- **Using `ingest_timestamp` as the source timestamp.** A chunk re-embedded at noon today looks always-fresh, even if its content was last changed two weeks ago. The source has to carry the authoritative timestamp.
- **Treating exemption as permanent.** Exempt source classes solve the "evergreen content" problem; they create the "we quietly stopped measuring" problem the moment nobody reviews them. Keep the exemption count on a dashboard and review quarterly.
- **A single global SLI across routes.** A route whose freshness need is minutes drags down the SLI's averaged ratio for a route whose freshness need is a day, and vice versa. Per-route SLOs are not a nice-to-have.
- **No distinction between "pipeline is behind" and "upstream is quiet".** The two have different owners and different fixes. If the runbook only has one lane, every freshness burn becomes a pipeline debugging session.
- **Widening the tolerance to silence the alert.** Widening is a legitimate mitigation; making it permanent is how the SLO becomes a vanity metric. The override must auto-expire.
- **Mixing cache-hit traffic into the denominator.** A cache-hit answer did not call retrieval; counting it as "fresh" is only correct if caches invalidate on upstream edits — and almost no RAG cache does.

## 6. References

- Google SRE Workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — source of the multi-window multi-burn-rate recipe used in §2.3.
- Google SRE Book, *Service Level Objectives* — [sre.google/sre-book/service-level-objectives/](https://sre.google/sre-book/service-level-objectives/) — the SLO / SLI / error budget vocabulary.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — `MEASURE 2.3` names the requirement that performance be measured in deployment-like conditions.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML02 data poisoning` motivates tracking source timestamps: if a stale (or adversarially backdated) chunk slips into the index, the SLI is one of the signals that surfaces it.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the `AML.T0020 — Poison Training Data` and related techniques target the data layer; freshness tracking is a necessary (not sufficient) control for noticing when the layer has been tampered with.
- OpenTelemetry GenAI semantic conventions — introduced in `mod-104 / exercise-01` — are the source of the `genai.retrieval.*` attributes used in §2.2.
