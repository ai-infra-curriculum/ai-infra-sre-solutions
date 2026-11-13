# Exercise 06 — Structured Log Discipline with PII Redaction at High QPS (Solution)

## 1. Solution overview

Define a logging contract for an LLM request path that is (a) structured, (b) PII-aware, and (c) fast enough not to be the first thing to break at peak QPS. The target is a service pushing a few thousand requests per second per pod where logging naively will add tail latency and create a compliance surface.

Four invariants:

1. **One log event per request**, JSON, flat. Child systems emit at most one "chunk" or "step" event each; no per-token logs in production.
2. **Prompts and completions never land in general logs.** They live on OTel span events behind the content-capture flag from Exercise 01, or in a dedicated audit channel with access controls.
3. **PII redaction happens before serialization**, in-process, deterministically. Downstream redaction is a backstop, not a primary control.
4. **Logging is non-blocking on the request path.** Any call that could stall (network, disk fsync, synchronous regex on a 1 MB body) runs off the hot path.

## 2. Worked implementation

### 2.1 Log schema (minimum viable)

```json
{
  "ts": "2026-11-12T20:11:04.812Z",
  "level": "info",
  "service": "llm-gateway",
  "env": "prod",
  "trace_id": "9f13...e3",
  "span_id": "7a0c...c1",
  "request_id": "req_01HQ...",
  "user_id_hash": "sha256:9d4e...b7",
  "session_id_hash": "sha256:2aab...03",
  "route": "POST /v1/chat",
  "workload": "chat",
  "model": "gpt-4o-mini",
  "status": 200,
  "latency_ms": 834,
  "ttft_ms": 212,
  "tpot_ms": 18.4,
  "input_tokens": 183,
  "output_tokens": 96,
  "cached_prefix_tokens": 128,
  "error_type": null,
  "redactions": {"email": 1, "phone": 0, "ssn": 0}
}
```

Notes on the schema:

- `trace_id` and `span_id` are the OTel IDs from Exercise 01; they let a reader jump from log → trace instantly.
- `user_id_hash` and `session_id_hash` are HMAC-SHA-256 with a per-environment key so user identity cannot be recovered from the log, but repeat visits by the same user *can* be correlated during an investigation.
- `redactions` is a *count*, not a copy of what was redacted. Counting lets you detect regressions (sudden drop in email redactions = suspicious).
- No `prompt`, no `completion`, no free-text user input.

### 2.2 Redaction pipeline

Layered, fail-closed:

1. **Explicit field strip** — never pass the raw prompt into the logger at all. The handler passes a `LogView` struct that excludes the body.
2. **Regex pass** on any remaining free-text fields (headers, error messages). Fast, deterministic, and covers common patterns.
3. **Model-based detector** (Presidio or equivalent) runs **asynchronously** on sampled audit traffic — too slow for the hot path. It validates the regex pass and surfaces new PII shapes.
4. **Downstream backstop**: a logging pipeline stage in your log aggregator (Vector, Fluent Bit) re-runs redaction for defense in depth.

Regex rules (minimum):

```python
import re
PATTERNS = {
    "email": re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+"),
    "phone_us": re.compile(r"\b(?:\+?1[-. ]?)?\(?\d{3}\)?[-. ]?\d{3}[-. ]?\d{4}\b"),
    "ssn": re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
    "card_candidate": re.compile(r"\b(?:\d[ -]*?){13,19}\b"),
    "ipv4": re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b"),
}

def redact(text: str) -> tuple[str, dict[str, int]]:
    counts = {k: 0 for k in PATTERNS}
    out = text
    for name, pat in PATTERNS.items():
        out, n = pat.subn(f"<{name.upper()}>", out)
        counts[name] = n
    return out, counts
```

Caveats on regex redaction:

- Luhn-validate credit-card candidates before treating them as hits, or you'll flood logs with false-positive numeric IDs.
- Phone patterns are locale-specific; expand per market.
- Regex misses contextual PII (names, addresses) — that is why the model-based detector exists as a validator.

### 2.3 Hot-path performance pattern (Python)

Non-blocking JSON logging with `structlog`:

```python
# logging_setup.py
import logging
import orjson
import structlog
from logging.handlers import QueueHandler, QueueListener
from queue import SimpleQueue

queue = SimpleQueue()
queue_handler = QueueHandler(queue)
stream_handler = logging.StreamHandler()
stream_handler.setFormatter(logging.Formatter("%(message)s"))
listener = QueueListener(queue, stream_handler, respect_handler_level=True)
listener.start()

root = logging.getLogger()
root.handlers = [queue_handler]
root.setLevel(logging.INFO)

def _serialize(_, __, event_dict):
    return orjson.dumps(event_dict).decode()

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        _serialize,
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    cache_logger_on_first_use=True,
)
log = structlog.get_logger()
```

The `QueueHandler` + `QueueListener` pattern moves the actual write off the request thread; `orjson` serializes an order of magnitude faster than the stdlib. The combination keeps logging under ~50µs per event on modern hardware. Measure in your service before trusting any such number.

Go equivalent: `uber-go/zap` production config (zap.NewProductionEncoderConfig + zapcore.NewSamplerWithOptions). Node equivalent: `pino` with `pino.transport({ target: 'pino/file', options: { destination: 1 } })` async writer.

### 2.4 Sampling at high QPS

- Sample the **successful** happy-path events at ~1%; emit every error unsampled.
- Keep the sampling ratio in a label so Prometheus can reconstruct absolute counts: `log_events_emitted_total{sampled="true",rate="0.01"}`.
- Always emit trace IDs; if you need the full-res story of a request you can go to the trace backend.

### 2.5 Transport & storage posture

- **Transport**: Fluent Bit or Vector agent on the node; backpressure must drop at the agent, not the application. The app logger's queue has a bounded size and drops oldest.
- **At-rest**: logs in a bucket/index with short retention (14–30 days for general operational logs). Audit-channel content lives in a separate index with long retention, access controls, and legal review.
- **Index hygiene**: never index free-text fields globally; index on IDs, model, workload, status, and timestamp.

### 2.6 Framework mapping

- **NIST AI RMF GOVERN 1.4 and MAP 3.5** call for data handling controls and a documented data lifecycle; the schema's "no prompt in logs" rule is one concrete control.
- **OWASP ML Top 10** includes risks around sensitive data disclosure through model inputs/outputs; keeping those out of logs removes one leak surface.
- **MITRE ATLAS** techniques around information gathering can exploit over-logged pipelines to find training data or user content; this exercise's "no prompt in general logs" rule mitigates that path.

> Note: confirm exact NIST AI RMF function and subcategory IDs for data handling controls against the active NIST AI RMF 1.0 document; the mapping above is directionally correct but subcategory numbering should be verified before citing in a compliance context.

## 3. Validation steps

1. **Schema conformance**: run a JSON Schema validator against a 10-minute log capture. Zero violations.
2. **No-prompt check**: grep the capture for a canary string that you injected in a request body; it must not appear.
3. **Redaction regression test**: unit test the redactor with a table of inputs covering each PII class, plus false-positive cases (e.g., numeric IDs that aren't credit cards).
4. **Luhn test**: a 16-digit non-card number (e.g., `1234 5678 1234 5678`) should **not** be redacted; a real-shape test card should be.
5. **Throughput benchmark**: `k6`/`wrk` the service with and without logging enabled. Overhead at p99 should be <5% at your target QPS.
6. **Backpressure drill**: stop the aggregator. Confirm the app keeps serving traffic (logs drop; p99 unaffected).
7. **Audit-channel ACL check**: a random developer can read general logs; only the audit-access group can read the audit channel.
8. **Redaction-count alerting**: breakdown of `redactions.*` counts is visible in a dashboard; a sudden zero for a previously nonzero field triggers investigation.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Single-event rule | One structured event per request; no per-token logs |
| No prompts in logs | Canary string search returns zero in general logs |
| ID hashing | `user_id` and `session_id` are hashed; raw values never serialized |
| Redaction layering | Field-strip + regex + sampled model-based + downstream backstop |
| Hot-path discipline | Logging is async / non-blocking; benchmark artifact included |
| Sampling | Sampling ratio applied and labeled so absolute counts are reconstructable |
| Transport posture | Backpressure drops at agent; app queue is bounded |
| Access controls | Audit channel ACL is separate from general logs |
| Framework mapping | Explicit mapping to NIST AI RMF, OWASP ML Top 10, and MITRE ATLAS where applicable |

## 5. Common mistakes

- **Writing prompts to logs for "debuggability".** It ends up on every disk and replica. Use span events behind the content-capture flag, or an audit-only channel with ACLs.
- **Trusting one regex pass for PII.** False negatives (contextual PII, obfuscated patterns) and false positives (numeric IDs that look like SSNs) both occur. Layer defenses.
- **Synchronous logging on the request thread.** First thing to blow up tail latency under load. Use a queue/async writer.
- **Unbounded in-process queue.** Memory grows until OOM when the aggregator stalls. Bound the queue; drop on overflow and emit a counter about the drop.
- **Indexing free-text fields globally.** Crashes search latency and makes PII searchable by anyone with log access. Index on IDs, status, model — not content.
- **No redaction metrics.** A silent regression (redactor accidentally disabled, pattern list truncated) goes unnoticed until an audit. Alert on sudden drops in `redactions.*{field}` counts.
- **Treating sampling as uniform.** Sample **successful** events; always emit errors at 100%.
- **No correlation IDs.** Logs without `trace_id`/`span_id` are useless during incidents; readers can't join to the trace platform.

## 6. References

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- OWASP Machine Learning Security Top 10: https://owasp.org/www-project-machine-learning-security-top-10/
- Microsoft Presidio (PII detection toolkit): https://microsoft.github.io/presidio/
- NIST AI Risk Management Framework: https://www.nist.gov/itl/ai-risk-management-framework
- NIST SP 800-92 — Guide to Computer Security Log Management: https://csrc.nist.gov/publications/detail/sp/800-92/final
- Python structlog documentation: https://www.structlog.org/
- Fluent Bit documentation (backpressure, redaction filters): https://docs.fluentbit.io/
- Vector documentation (transform / parse / redact): https://vector.dev/docs/
- MITRE ATLAS: https://atlas.mitre.org/
