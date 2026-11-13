# Exercise 01 — OTel GenAI Semantic Conventions on One Request Path (Solution)

## 1. Solution overview

Instrument a single LLM request path end to end using the OpenTelemetry **GenAI semantic conventions** so that every span can be grouped and filtered by the standard attribute names regardless of which trace backend ingests it. The scope of the exercise is intentionally narrow: one HTTP entrypoint, one model call, one span tree. Everything else (streaming, retries, multi-turn chat) is a follow-on.

The design has three invariants:

1. **One root span per inbound request** with transport/HTTP semconv attributes.
2. **One child span per LLM inference call** named `chat <model>` (or `text_completion <model>` / `embeddings <model>`) carrying the `gen_ai.*` attributes.
3. **Span events, not log lines**, for prompt and completion content — gated behind a `capture_content` switch so PII is never emitted when it is turned off.

Keep the span tree flat and shallow; most incident readers only need the root span plus the LLM child.

## 2. Worked implementation

The reference implementation is a FastAPI service with the OpenAI SDK wrapped in OpenTelemetry. The same attribute set applies to vLLM, SGLang, TRT-LLM, Bedrock, Anthropic, or a self-hosted route — only `gen_ai.system` changes.

### 2.1 Span naming

Per the OpenTelemetry GenAI spec, span names follow the pattern `<operation.name> <request.model>`, e.g. `chat gpt-4o-mini`. Keep them low-cardinality — do not put the request ID in the span name.

### 2.2 Required and recommended attributes (model-call span)

| Attribute | Required? | Example |
|---|---|---|
| `gen_ai.system` | required | `openai`, `anthropic`, `vllm`, `az.ai.inference` |
| `gen_ai.operation.name` | required | `chat`, `text_completion`, `embeddings` |
| `gen_ai.request.model` | required | `gpt-4o-mini` |
| `gen_ai.response.model` | recommended | `gpt-4o-mini-2024-07-18` |
| `gen_ai.request.max_tokens` | recommended | `512` |
| `gen_ai.request.temperature` | recommended | `0.2` |
| `gen_ai.request.top_p` | recommended | `1.0` |
| `gen_ai.response.id` | recommended | `chatcmpl-...` |
| `gen_ai.response.finish_reasons` | recommended | `["stop"]` |
| `gen_ai.usage.input_tokens` | recommended | `183` |
| `gen_ai.usage.output_tokens` | recommended | `96` |
| `server.address`, `server.port` | recommended | `api.openai.com`, `443` |
| `error.type` | on error | `openai.RateLimitError` |

Prompt/completion bodies go on **span events** named `gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.tool.message`, `gen_ai.choice`. Gate them behind a config flag — see §2.5.

### 2.3 Minimal Python reference

```python
# app.py
import os
from fastapi import FastAPI
from pydantic import BaseModel
from openai import OpenAI
from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode

tracer = trace.get_tracer("llm-gateway", "0.1.0")
client = OpenAI()
CAPTURE_CONTENT = os.getenv("OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT", "false").lower() == "true"

app = FastAPI()

class ChatIn(BaseModel):
    user_message: str
    model: str = "gpt-4o-mini"
    temperature: float = 0.2
    max_tokens: int = 512

@app.post("/v1/chat")
def chat(body: ChatIn):
    # The HTTP server span is created by the FastAPI auto-instrumentation.
    # We add the explicit LLM child span below.
    span_name = f"chat {body.model}"
    with tracer.start_as_current_span(span_name, kind=SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.system", "openai")
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.request.model", body.model)
        span.set_attribute("gen_ai.request.temperature", body.temperature)
        span.set_attribute("gen_ai.request.max_tokens", body.max_tokens)
        span.set_attribute("server.address", "api.openai.com")
        span.set_attribute("server.port", 443)

        if CAPTURE_CONTENT:
            span.add_event(
                "gen_ai.user.message",
                attributes={"content": body.user_message, "role": "user"},
            )

        try:
            resp = client.chat.completions.create(
                model=body.model,
                temperature=body.temperature,
                max_tokens=body.max_tokens,
                messages=[{"role": "user", "content": body.user_message}],
            )
        except Exception as exc:
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            span.set_attribute("error.type", type(exc).__name__)
            raise

        choice = resp.choices[0]
        span.set_attribute("gen_ai.response.id", resp.id)
        span.set_attribute("gen_ai.response.model", resp.model)
        span.set_attribute("gen_ai.response.finish_reasons", [choice.finish_reason])
        if resp.usage:
            span.set_attribute("gen_ai.usage.input_tokens", resp.usage.prompt_tokens)
            span.set_attribute("gen_ai.usage.output_tokens", resp.usage.completion_tokens)

        if CAPTURE_CONTENT:
            span.add_event(
                "gen_ai.choice",
                attributes={
                    "finish_reason": choice.finish_reason,
                    "index": choice.index,
                    "message": {"role": "assistant", "content": choice.message.content},
                },
            )

        return {"reply": choice.message.content}
```

### 2.4 Tracer and exporter bootstrap

```python
# otel_bootstrap.py — call before FastAPI auto-instrumentation.
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

resource = Resource.create({
    "service.name": "llm-gateway",
    "service.version": "0.1.0",
    "deployment.environment": "staging",
})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)
# Later, after the FastAPI app is created:
# FastAPIInstrumentor.instrument_app(app)
```

Use `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and `OTEL_RESOURCE_ATTRIBUTES` env vars for non-code config.

### 2.5 Content capture switch

The OTel GenAI spec defines `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` (boolean). Default it to `false` in production and only turn it on in a debug namespace with access controls. When off, you still emit the structural events (role, finish_reason) but the `content` field is omitted. This is the smallest responsible default; it means a trace is useful for triage without becoming a PII exfiltration surface.

### 2.6 Streaming (TTFT/TPOT)

For streaming responses the span stays open for the whole stream. Add two span events to make TTFT/TPOT derivable:

- `gen_ai.choice` with `event.timestamp` on the first token.
- Span `end` at stream close.

Then TTFT = `first_choice_event.timestamp - span.start_time`, total = `span.end_time - span.start_time`, TPOT = `(total - TTFT) / output_tokens`. Exercise 02 computes these as PromQL metrics; this exercise only requires the trace fields be present.

## 3. Validation steps

1. **Local trace export sanity check**. Run with `OTEL_TRACES_EXPORTER=console` and a test request. Confirm:
   - A single root span for the HTTP request.
   - One child span named `chat <model>`.
   - All five required attributes present (`gen_ai.system`, `gen_ai.operation.name`, `gen_ai.request.model`, plus response model and usage when the call succeeds).
2. **Collector pipeline**. Point OTLP at an OpenTelemetry Collector with the `debug` exporter. Confirm attributes arrive unchanged.
3. **Semconv conformance**. Spot-check against the current GenAI semconv page (see References) for any attribute marked `required` that is missing on your spans.
4. **Content gating**. Flip `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` on and off; verify the `content` field disappears when off but the structural events remain.
5. **Error path**. Force a `401`/`429` from the provider; confirm `error.type` is set and span status is `ERROR`.
6. **Cardinality audit**. Export 1000 synthetic requests; confirm no attribute (especially `gen_ai.request.model`) exceeds a small bounded set of values.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Span structure | Exactly one LLM child span per model call; name follows `<op> <model>` |
| Required attributes | All attributes marked `required` in the current GenAI semconv are present on successful spans |
| Response attributes | `response.model`, `response.id`, `finish_reasons`, both `usage.*_tokens` present when the provider returns them |
| Content handling | Prompt/completion go on events, not attributes; gated by `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` |
| Error semantics | `error.type` + span status `ERROR` on exceptions; no silent swallow |
| Resource attributes | `service.name`, `service.version`, `deployment.environment` set |
| Export path | Spans reach the configured OTLP endpoint without the application patching attribute names |

Full pass = all seven. Any missing required attribute is a fail regardless of the other rows.

## 5. Common mistakes

- **Putting prompts in attributes instead of events.** Attributes are indexed and have cardinality limits; large prompt bodies will either get dropped by the backend or blow up index cost. Events are the right surface.
- **Logging the prompt separately from the span.** The content-capture flag only governs OTel output — if a `logger.info(prompt)` sits alongside, flipping the flag off does nothing. Audit `logger.*` calls in the same hot path.
- **Mixing vendor attribute names with semconv names** (`model`, `llm.model`, `openai.model`, `gen_ai.request.model` all on one span). Pick semconv names and delete the aliases; downstream dashboards will read one of them and silently miss the others.
- **Naming spans with request IDs or user IDs.** This breaks span-name aggregation in every backend. Put those on attributes (`http.request.id`, hashed user ID) instead.
- **Treating streaming as "one span, one timestamp"** and losing TTFT. Emit a `gen_ai.choice` event on the first token or the ITL metric in Exercise 02 has nothing to anchor on.
- **Hard-coding `gen_ai.system = "openai"`** for a gateway that fronts multiple providers. Derive it per call.
- **Forgetting the error path.** Many implementations set attributes only on the happy path; incident reviewers then can't tell a 429 storm from a model latency regression.

## 6. References

- OpenTelemetry Semantic Conventions, Generative AI: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- OpenTelemetry Semantic Conventions, GenAI spans: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
- OpenTelemetry Semantic Conventions, GenAI events: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/
- OpenTelemetry general trace semantic conventions (HTTP, server.*): https://opentelemetry.io/docs/specs/semconv/
- OWASP Machine Learning Security Top 10 (context on sensitive data exposure risks): https://owasp.org/www-project-machine-learning-security-top-10/
- NIST AI Risk Management Framework (MEASURE and MANAGE functions for traceability): https://www.nist.gov/itl/ai-risk-management-framework
