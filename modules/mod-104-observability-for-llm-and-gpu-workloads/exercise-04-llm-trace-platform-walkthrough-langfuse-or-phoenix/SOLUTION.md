# Exercise 04 — LLM Trace Platform Walkthrough (Langfuse or Phoenix) (Solution)

## 1. Solution overview

Run one LLM-specific trace platform — **Langfuse** or **Arize Phoenix** — against the request path already instrumented in Exercise 01, and complete a guided walkthrough covering: install, instrument, submit traces, search/filter, evaluate, export. The deliverable is a short decision rationale ("which one, for which team context") plus a reproducible setup for the one you picked.

Both platforms accept OpenTelemetry-shaped traces and both publish Python/JS SDKs. The right choice is driven less by feature parity and more by deployment constraints, evaluation workflow, and where your team already lives.

## 2. Worked implementation (walkthrough)

### 2.1 Decision rationale (pick one, keep it)

| Dimension | Langfuse | Phoenix |
|---|---|---|
| Hosting | OSS self-host; managed cloud available | OSS self-host; Arize AX is the paid managed sibling |
| Primary data model | Trace → observations (generation / span / event) | OTel trace → spans, LLM semconv |
| Ingest paths | SDK (Py/JS), OpenTelemetry OTLP, OpenInference | OpenTelemetry OTLP + OpenInference instrumentors |
| Evaluation | Built-in LLM-as-judge, custom scores, dataset experiments | Built-in evaluators, dataset experiments, retrieval evals |
| Prompt management | Versioned prompts, variables, deployment labels | Prompt playground, versioning |
| Fit | Product teams mixing eval + analytics + prompt ops | Teams already on OTel; ML/data-science evaluation workflows |

Pick **Langfuse** when the team wants prompt management and analytics in one place and has a product-shaped workflow. Pick **Phoenix** when the trace surface is already OpenTelemetry-native and the primary workflow is systematic evaluation on datasets. Either is defensible; don't run both in production.

### 2.2 Install (Langfuse self-host)

```bash
# docker-compose.yaml — minimal self-host
# Use the versioned compose file from the Langfuse repo rather than copy-pasting.
curl -O https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml
docker compose up -d

# UI at http://localhost:3000 — create project, mint LF_PUBLIC_KEY / LF_SECRET_KEY.
```

> Note: the exact file name in the Langfuse repo has changed across releases (`docker-compose.yml` vs `docker-compose.build.yml`); confirm against the current `/langfuse/langfuse/README` before copying into a runbook.

### 2.3 Install (Phoenix self-host)

```bash
# Simplest path: run Phoenix as a container.
docker run -d --name phoenix -p 6006:6006 -p 4317:4317 arizephoenix/phoenix:latest
# UI at http://localhost:6006
# OTLP gRPC endpoint on :4317
```

### 2.4 Instrument the request path (Langfuse via the SDK)

```python
# langfuse_wrap.py — layered on top of the Exercise 01 handler
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

lf = Langfuse()  # reads LANGFUSE_HOST / _PUBLIC_KEY / _SECRET_KEY

@observe(name="chat_endpoint")
def chat(body):
    langfuse_context.update_current_trace(
        user_id=body.user_id,
        session_id=body.session_id,
        tags=[f"workload:{body.workload}"],
    )
    reply = call_model(body)  # @observe(as_type="generation")
    return reply

@observe(as_type="generation", name="openai.chat")
def call_model(body):
    langfuse_context.update_current_observation(
        model=body.model,
        model_parameters={"temperature": body.temperature, "max_tokens": body.max_tokens},
        input=[{"role": "user", "content": body.user_message}],
    )
    resp = client.chat.completions.create(...)
    langfuse_context.update_current_observation(
        output=resp.choices[0].message.content,
        usage={
            "input": resp.usage.prompt_tokens,
            "output": resp.usage.completion_tokens,
            "total": resp.usage.total_tokens,
        },
    )
    return resp.choices[0].message.content
```

### 2.5 Instrument the request path (Phoenix via OpenTelemetry + OpenInference)

```python
# phoenix_bootstrap.py
from phoenix.otel import register

tracer_provider = register(
    project_name="llm-gateway",
    endpoint="http://localhost:6006/v1/traces",   # or OTLP gRPC :4317
    auto_instrument=True,  # picks up OpenInference instrumentors installed in the env
)
# `pip install openinference-instrumentation-openai` (or anthropic/bedrock/etc.) before import.
```

If you already emit OpenTelemetry GenAI semconv spans from Exercise 01, Phoenix ingests them directly — no SDK migration required. OpenInference semconv adds a few LLM-specific attributes beyond the OTel GenAI set.

### 2.6 Walkthrough tasks (both platforms)

1. **Record a trace.** Send one request from a fresh terminal. Confirm it shows up in the UI within a few seconds.
2. **Open the trace timeline.** Verify the root span has `user_id`, `session_id`, the workload tag, and total latency; the child generation span shows model name, temperature, input/output text, and token usage.
3. **Filter.** In the UI, filter by `workload = chat` and by `model = gpt-4o-mini`. Confirm your request appears.
4. **Attach a score.** From the UI (and via the SDK), attach a numeric score (0..1) representing a quality signal. Confirm it renders on the trace.
5. **Run an LLM-as-judge.** Create a judge template ("is the response helpful and factual?") against yesterday's traces. Inspect a handful of outputs; recalibrate the rubric if the judge is lenient.
6. **Create a dataset.** Promote 10 real traces to a dataset. Re-run the current gateway against that dataset and compare outputs/latency; this is your regression harness.
7. **Export.** Export the last 24h as JSON/Parquet. Confirm the schema is stable enough to feed into your warehouse or notebook.

### 2.7 How this fits with the OTel work in Exercise 01

- The OTel GenAI semconv attributes from Exercise 01 are the **source of truth** for metrics and generic trace backends (Tempo, Jaeger, Honeycomb).
- Langfuse/Phoenix sits **on top**: either ingesting those same OTel spans directly (Phoenix is OTel-native; Langfuse has an OTel ingest path) or layering the SDK for the extras (prompt versioning, dataset experiments, judge scores).
- Do not double-emit. Either pipe OTel spans through the trace platform's OTLP endpoint, or use the platform SDK and have it emit — not both, or you will debug duplicated spans for an afternoon.

## 3. Validation steps

1. **UI smoke**: a single request produces one trace with one generation span visible in <10 seconds.
2. **PII posture**: open the trace; confirm message content is present only if `capture_content` is on (Exercise 01 switch). If you expect PII, confirm your SDK's redaction hook actually fires on a known test string.
3. **Latency sanity**: `trace.latency ≈ external request latency ± 50ms`. A big delta means either clock skew or the SDK is holding a batch.
4. **Search coverage**: filter by `model`, `user_id`, `session_id`, and `tags`. All four should work.
5. **Score roundtrip**: post a score via SDK; refresh the trace; score visible.
6. **Dataset re-run**: dataset experiment from a 10-row dataset returns one row per input and the diff view renders.
7. **Retention sanity**: check what retention/sampling is on for the project; a free-tier default may silently drop older traces during an investigation.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Decision rationale | Written, concrete, references team context — not just a feature table |
| Install reproducible | Docker compose or image run documented with pinned versions or links to pinned refs |
| Instrumentation | One request produces a trace with: model, params, token usage, input, output |
| User/session context | `user_id` and `session_id` present on every trace (hashed if PII) |
| Workload tagging | At least one workload label visible for filtering |
| Evaluation | One working judge or scoring workflow demonstrated end to end |
| Dataset | At least one dataset seeded from real traces |
| Integration honesty | OTel vs SDK decision explicit, no double-emission |

## 5. Common mistakes

- **Running both Langfuse and Phoenix in parallel "to compare".** The team ends up maintaining neither well. Pick one for 90 days; revisit with real usage data.
- **Capturing full prompts/completions by default in production.** PII, legal review, and storage all suffer. Default to off; add an opt-in debug lane with ACLs.
- **Treating the trace platform as the alerting system.** It is not. Metrics in Prometheus (Exercise 02) alert; traces explain. Alerting from trace counts is brittle.
- **Instrumenting only the happy path.** Error traces — 4xx/5xx from the provider, timeouts, retries — are the ones you'll actually need during an incident. Make sure they end up in the platform.
- **No `user_id` or `session_id` plumbed in.** You then can't answer "what did user X see?" during a complaint, which is the primary non-SRE use case.
- **Trusting one LLM-as-judge pass without calibration.** Judges drift; always ground them with a small human-labeled set before trusting a score.
- **Forgetting to re-point the SDK at a staging project during drills.** Load tests dumped into the prod project corrupt the dataset you later train judges on.
- **Local self-host with no backups.** Default container volumes vanish on restart. Mount a persistent volume for the DB (Postgres for Langfuse; Phoenix persistence is backend-dependent).

## 6. References

- Langfuse documentation: https://langfuse.com/docs
- Langfuse OpenTelemetry integration: https://langfuse.com/docs/opentelemetry
- Langfuse self-hosting: https://langfuse.com/self-hosting
- Arize Phoenix documentation: https://arize.com/docs/phoenix
- Phoenix tracing (OTel setup): https://arize.com/docs/phoenix/tracing/llm-traces-1
- OpenInference semantic conventions: https://github.com/Arize-ai/openinference
- OpenTelemetry GenAI semantic conventions (foundation for both): https://opentelemetry.io/docs/specs/semconv/gen-ai/
- NIST AI Risk Management Framework (context on measurement & traceability requirements): https://www.nist.gov/itl/ai-risk-management-framework
