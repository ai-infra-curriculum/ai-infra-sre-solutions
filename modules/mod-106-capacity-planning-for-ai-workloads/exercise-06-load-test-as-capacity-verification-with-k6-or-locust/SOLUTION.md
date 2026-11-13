# Exercise 06 — Load Test as Capacity Verification with k6 or Locust — Solution

## 1. Solution overview

The base model (Exercise 01), the forecast worksheet (Exercise 02), the
fleet plan (Exercise 03), the quota pool (Exercise 04), and the HPA
tuning (Exercise 05) are all *predictions*. A capacity plan is not
trusted until a load test puts traffic through the real serving tier and
shows the SLOs survive. This exercise is the empirical verification
step.

The learner is asked to build a k6 or Locust load test that:

1. Produces realistic LLM traffic — correct prompt/output length
   distribution, correct arrival pattern, streaming responses treated
   correctly for TTFT measurement.
2. Measures TTFT, TPOT, and error rate per workload class, not as a
   global average.
3. Ramps through at least three stages — baseline, forecast peak, and
   an over-forecast stress — and asserts SLO thresholds at each.
4. Reports capacity headroom (how far above the forecast the SLOs
   held) as a number the capacity review can consume.

The reference solution below supplies a runnable k6 script, a Locust
alternative, the rationale behind each design choice, and a review
rubric. The script is intentionally parameterised so the same harness
re-runs after each capacity change.

## 2. Worked implementation — the load test

### 2.1 Design principles

- **Open-model arrivals.** Use a wall-clock arrival generator that is
  independent of per-request latency: k6's `constant-arrival-rate` /
  `ramping-arrival-rate`, or Locust's `ArrivalRateUser` from
  `locust-plugins`. Avoid closed-loop "N virtual users looping" and
  avoid Locust's `constant_throughput` / `constant_pacing` wait-times
  for streaming workloads — both back-pressure the arrival rate on
  response time, which hides queueing effects and under-reports TTFT.
- **Traffic class decomposition.** Emit at least the Exercise-02
  classes — chat stream, RAG, batch — each at their forecast arrival
  rate. Tag every request with its class so the metrics pipeline can
  slice by class.
- **Streaming-aware TTFT.** For streaming APIs, measure TTFT as time
  from request send to the first *content* byte on the stream (not the
  HTTP response headers). This requires reading bytes in the handler,
  not relying on request-complete timing.
- **Length distribution faithful to production.** Sample `L_in` and
  `L_out` from the measured production distribution (mean + p95 at a
  minimum). A flat 500-token test is not a verification; it is a demo.
- **Stage ramps with SLO gates.** The run proceeds through baseline →
  forecast → over-forecast stages. Each stage has pass/fail thresholds;
  fail at any stage fails the test.
- **Observability on both sides.** Load generator metrics *and*
  server-side Prometheus scraped during the run. TTFT measured at the
  client is the truth; server-side metrics explain *why* when the
  client-side number misses.

### 2.2 k6 script (runnable skeleton)

```javascript
// capacity-verification.js
// Run: k6 run --env BASE_URL=https://inference.example.com \
//              --env TOKEN=... capacity-verification.js
//
// Requires a custom k6 build that includes the xk6-sse extension so SSE
// events are delivered to the handler incrementally (rather than after
// the full body arrives, as with the stock `k6/http` module):
//
//   xk6 build --with github.com/phymbert/xk6-sse

import sse from 'k6/x/sse';
import { Trend, Rate, Counter } from 'k6/metrics';
import { SharedArray } from 'k6/data';
import exec from 'k6/execution';

// Per-class metrics (custom, so dashboards can slice by workload class).
const ttft_chat     = new Trend('ttft_ms_chat',     true);
const ttft_rag      = new Trend('ttft_ms_rag',      true);
const ttft_batch    = new Trend('ttft_ms_batch',    true);
const tpot_chat     = new Trend('tpot_ms_chat',     true);
const tpot_rag      = new Trend('tpot_ms_rag',      true);
const tpot_batch    = new Trend('tpot_ms_batch',    true);
const err           = new Rate('errors');
const transportErr  = new Counter('transport_failures');
const dropped429    = new Counter('rate_limited_429');

// Prompt samples drawn from the measured production distribution
// (sanitised for the test harness).
const prompts = new SharedArray('prompts', function () {
  return JSON.parse(open('./prompts.json'));   // [{class, prompt, max_tokens}]
});

// Index prompts by workload class once, at init time. Using `.find()` in
// the hot path would always return the first matching sample and silently
// collapse the length distribution to a single point — defeating the
// production-faithful L_in / L_out requirement. We randomise across all
// samples within the chosen class instead.
const PROMPTS_BY_CLASS = (function () {
  const b = { chat: [], rag: [], batch: [] };
  for (const p of prompts) {
    if (b[p.class]) b[p.class].push(p);
  }
  if (!b.chat.length || !b.rag.length || !b.batch.length) {
    throw new Error('prompts.json must contain samples for chat, rag, and batch');
  }
  return b;
})();

export const options = {
  discardResponseBodies: false,
  thresholds: {
    'ttft_ms_chat':      ['p(95)<800',  'p(99)<1500'],
    'ttft_ms_rag':       ['p(95)<1500', 'p(99)<2500'],
    'ttft_ms_batch':     ['p(95)<60000'],
    'tpot_ms_chat':      ['p(95)<50'],
    'tpot_ms_rag':       ['p(95)<60'],
    // `errors` includes transport failures (status 0) as well as 4xx/5xx,
    // so an overloaded service that drops TCP/TLS or times out still
    // trips the gate. `transport_failures` is a separate hard ceiling.
    'errors':            ['rate<0.005'],
    'transport_failures': ['count<10'],
  },
  scenarios: {
    baseline: {
      executor: 'constant-arrival-rate',
      rate: 35,              // from Exercise 02 baseline
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 200,
      maxVUs: 1000,
      exec: 'mixed',
      tags: { stage: 'baseline' },
    },
    forecast_peak: {
      executor: 'constant-arrival-rate',
      rate: 70,              // from Exercise 02 peak forecast
      timeUnit: '1s',
      startTime: '6m',
      duration: '10m',
      preAllocatedVUs: 400,
      maxVUs: 2000,
      exec: 'mixed',
      tags: { stage: 'forecast' },
    },
    over_forecast: {
      executor: 'ramping-arrival-rate',
      startRate: 70,
      stages: [
        { target: 100, duration: '5m' },   // 1.4x forecast
        { target: 100, duration: '5m' },   // hold
        { target: 140, duration: '5m' },   // 2x forecast — SLO is allowed
                                           // to breach here, but the service
                                           // must stay up (no crash, no
                                           // global 5xx).
      ],
      timeUnit: '1s',
      startTime: '17m',
      preAllocatedVUs: 800,
      maxVUs: 4000,
      exec: 'mixed',
      tags: { stage: 'overload' },
    },
  },
};

function pickPrompt() {
  // Class mix follows Exercise 02's forecast (e.g. 60% chat, 30% rag,
  // 10% batch). The iteration counter keeps the ratio stable across VUs;
  // the per-class draw is uniform over all samples in that class so the
  // production L_in / L_out distribution is exercised, not just its
  // first entry.
  const roll = (exec.scenario.iterationInTest * 2654435761) % 10;
  const cls = roll < 6 ? 'chat' : roll < 9 ? 'rag' : 'batch';
  const bucket = PROMPTS_BY_CLASS[cls];
  return bucket[Math.floor(Math.random() * bucket.length)];
}

export function mixed() {
  const p = pickPrompt();
  const body = JSON.stringify({
    model: 'inference-serve',
    stream: true,
    max_tokens: p.max_tokens,
    messages: [{ role: 'user', content: p.prompt }],
  });

  const start = Date.now();
  let firstByteTs = null;
  let lastTokenTs = null;
  let tokenCount = 0;
  let clientError = false;

  // sse.open streams events to the handler as they arrive — this is what
  // makes TTFT a real streaming measurement rather than a measurement of
  // total response time. The underlying HTTP response status is returned
  // on `response.status`.
  const response = sse.open(`${__ENV.BASE_URL}/v1/chat/completions`, {
    method: 'POST',
    body: body,
    headers: {
      'Authorization': `Bearer ${__ENV.TOKEN}`,
      'Content-Type': 'application/json',
      'Accept': 'text/event-stream',
      'X-Workload-Class': p.class,
    },
    timeout: '120s',
    tags: { workload_class: p.class },
  }, function (client) {
    client.on('event', function (event) {
      if (!event.data || event.data === '[DONE]') {
        if (event.data === '[DONE]') client.close();
        return;
      }
      const now = Date.now();
      if (firstByteTs === null) firstByteTs = now;
      lastTokenTs = now;
      tokenCount += 1;
    });
    client.on('error', function () {
      clientError = true;
      client.close();
    });
  });

  // Status 0 means the connection never completed (timeout, connection
  // refused, DNS, TLS, mid-stream reset). Count it as a transport failure
  // and an error — otherwise overloaded services look healthy.
  if (!response || response.status === 0 || clientError) {
    transportErr.add(1);
    err.add(1);
    return;
  }
  if (response.status === 429) {
    dropped429.add(1);
    err.add(1);
    return;
  }
  if (response.status >= 400) {
    err.add(1);
    return;
  }
  err.add(false);

  if (firstByteTs === null) {
    // 200 with no content events ever delivered: treat as a transport-class
    // failure for the purpose of the gate (no latency sample means a
    // silently broken stream, not a success).
    transportErr.add(1);
    err.add(1);
    return;
  }

  const ttft = firstByteTs - start;
  const tpot = (tokenCount > 1)
    ? (lastTokenTs - firstByteTs) / (tokenCount - 1)
    : NaN;

  (p.class === 'chat'  ? ttft_chat  :
   p.class === 'rag'   ? ttft_rag   :
                         ttft_batch).add(ttft);
  if (!Number.isNaN(tpot) && p.class !== 'batch') {
    (p.class === 'chat' ? tpot_chat : tpot_rag).add(tpot);
  }
}
```

Correctness notes on this script:

- `constant-arrival-rate` is an *open* model; `preAllocatedVUs` and
  `maxVUs` are safety ceilings, not the arrival driver.
- **Streaming is real streaming.** `sse.open` (from `xk6-sse`) dispatches
  each SSE event to the handler as the server flushes it, so `firstByteTs`
  is the wall-clock time the first *content* token arrives. The stock
  `k6/http` module returns the full body in one blocking call; walking
  that body post-hoc would make TTFT equal end-to-end response time and
  make TPOT equal local string-processing time — both wrong.
- TPOT is measured over inter-token gaps, not divided out of total
  duration — this matters because the first token carries prefill time
  in it and would distort an "all tokens averaged" number.
- Each class has its own trend metric and its own threshold in
  `options.thresholds`. The run fails if any class misses its SLO.
- **Transport failures count.** `response.status === 0`, connection
  resets observed by the SSE client, and 200 responses that deliver zero
  content events all increment `transport_failures` and `errors`. An
  overloaded service that times out or drops connections cannot sneak
  past the error-rate gate by producing no status code and no latency
  sample.
- **Prompts are sampled uniformly within each class.** `pickPrompt` picks
  the class from the forecast mix, then draws from `PROMPTS_BY_CLASS[cls]`
  — not `prompts.find()`, which would always return the first matching
  entry and collapse the production length distribution to a single point.

### 2.3 Locust alternative

If the team runs Locust (Python-native, better for shaping complex user
behaviour per-class), the equivalent user class:

```python
# locustfile.py
import json, time, random
from collections import defaultdict
from locust import task, events
# ArrivalRateUser provides *open-model* arrivals (inter-arrival time
# independent of task duration). Install via `pip install locust-plugins`.
from locust_plugins.users.arrival_rate_user import ArrivalRateUser

CLASSES = [
    ('chat',  0.6, {'ttft_p95_ms': 800,   'tpot_p95_ms': 50}),
    ('rag',   0.3, {'ttft_p95_ms': 1500,  'tpot_p95_ms': 60}),
    ('batch', 0.1, {'ttft_p95_ms': 60_000}),
]

with open('prompts.json') as f:
    PROMPTS = json.load(f)   # list of {class, prompt, max_tokens}

# Index prompts by class once. Using next(p for p in PROMPTS if ...) in
# the hot path returns the first matching entry every call, which
# collapses the production L_in / L_out distribution to a single sample
# and overstates capacity. Sample uniformly within the class instead.
PROMPTS_BY_CLASS = defaultdict(list)
for _p in PROMPTS:
    PROMPTS_BY_CLASS[_p['class']].append(_p)
for _cls, _w, _ in CLASSES:
    assert PROMPTS_BY_CLASS[_cls], f'prompts.json has no samples for {_cls!r}'


class InferenceUser(ArrivalRateUser):
    # Target *global* arrival rate (requests/s across all users). Set via
    # CLI: `locust --target-arrival-rate 70 ...`. ArrivalRateUser schedules
    # tasks on a wall-clock cadence that does NOT wait for the previous
    # task to finish — so a streaming response that takes 5 s does not
    # back-pressure the offered load, which is the point of a capacity
    # test. The stock Locust `constant_throughput(N)` / `constant_pacing`
    # wait-times measure only *between* tasks per user, so if generate()
    # exceeds 1/N seconds a user drops below the target rate and the
    # queueing this exercise is meant to verify disappears.
    target_arrival_rate = 70   # req/s, matches Exercise 02 forecast peak

    @task
    def generate(self):
        cls = _sample_class()
        prompt = random.choice(PROMPTS_BY_CLASS[cls])
        start = time.perf_counter()
        first_byte_ts = None
        last_ts = None
        tokens = 0
        transport_failed = False
        try:
            with self.client.post(
                '/v1/chat/completions',
                data=json.dumps({
                    'model': 'inference-serve',
                    'stream': True,
                    'max_tokens': prompt['max_tokens'],
                    'messages': [{'role': 'user', 'content': prompt['prompt']}],
                }),
                headers={
                    'Accept': 'text/event-stream',
                    'Content-Type': 'application/json',
                    'X-Workload-Class': cls,
                },
                stream=True,
                name=f'/v1/chat/completions[{cls}]',
                catch_response=True,
                timeout=120,
            ) as resp:
                if resp.status_code == 429:
                    resp.failure('rate_limited')
                    _fire_error(cls, 'rate_limited')
                    return
                if resp.status_code >= 400:
                    resp.failure(f'{resp.status_code}')
                    _fire_error(cls, f'status_{resp.status_code}')
                    return
                try:
                    for line in resp.iter_lines():
                        if not line or not line.startswith(b'data: '):
                            continue
                        if line.strip() == b'data: [DONE]':
                            break
                        now = time.perf_counter()
                        if first_byte_ts is None:
                            first_byte_ts = now
                        last_ts = now
                        tokens += 1
                except Exception as ex:   # mid-stream reset, read timeout
                    transport_failed = True
                    resp.failure(f'stream_error:{type(ex).__name__}')
        except Exception as ex:          # connect/TLS/DNS failure
            transport_failed = True
            _fire_error(cls, f'transport:{type(ex).__name__}')
            return

        if transport_failed or first_byte_ts is None:
            # No first-byte sample — treat as a transport-class failure so
            # an overloaded server that silently stalls does not pass.
            _fire_error(cls, 'no_first_byte')
            return

        ttft_ms = (first_byte_ts - start) * 1000
        tpot_ms = ((last_ts - first_byte_ts) / max(tokens - 1, 1)) * 1000 \
            if tokens > 1 else None
        events.request.fire(
            request_type='POST',
            name=f'ttft[{cls}]',
            response_time=ttft_ms,
            response_length=tokens,
            exception=None,
            context={'class': cls},
        )
        if tpot_ms is not None and cls != 'batch':
            events.request.fire(
                request_type='POST',
                name=f'tpot[{cls}]',
                response_time=tpot_ms,
                response_length=tokens,
                exception=None,
                context={'class': cls},
            )


def _fire_error(cls, reason):
    events.request.fire(
        request_type='POST',
        name=f'error[{cls}]',
        response_time=0,
        response_length=0,
        exception=Exception(reason),
        context={'class': cls, 'reason': reason},
    )


def _sample_class():
    r = random.random()
    acc = 0.0
    for cls, w, _ in CLASSES:
        acc += w
        if r < acc:
            return cls
    return CLASSES[-1][0]
```

Pair this with the standard Locust `--check-exceptions` and a thresholds
check in CI (e.g. `locust-plugins`' `set_fail_on_perf` or a custom
stage-end hook) so the run fails on SLO breach.

Three correctness notes on the Locust version:

- **Open-model arrivals require `ArrivalRateUser`.** `wait_time =
  constant_throughput(N)` and `constant_pacing(N)` both schedule
  *between* tasks on a single user, so if a streamed generation takes
  longer than `1/N` seconds — common for anything but the shortest
  chat — the user falls below `N` req/s. As latency grows under load,
  offered load falls, hiding precisely the queueing this test is meant
  to verify. `ArrivalRateUser` (from `locust-plugins`) dispatches tasks
  on wall-clock without waiting for prior tasks to return, which is
  what k6's `constant-arrival-rate` does natively. If `locust-plugins`
  is not available, prefer the k6 script above; a closed-loop Locust
  run will systematically understate required capacity.
- **Transport failures count.** `except` blocks around both the request
  and the stream-read fire explicit error events; a stream that opens,
  returns 200, then stalls without delivering a single content event
  also counts as `no_first_byte` rather than a silent pass.
- **Prompts are sampled uniformly within each class.** `PROMPTS_BY_CLASS`
  is built once at import time; `random.choice(PROMPTS_BY_CLASS[cls])`
  draws from the full production distribution rather than always
  returning `next(... first match ...)`.

### 2.4 Stage design and pass/fail rules

| Stage | Arrival rate | Duration | Pass criteria |
|---|---|---|---|
| Baseline | current production average | 5 min | All per-class SLOs green; no 5xx; <0.5% 429 |
| Forecast peak | Exercise-02 forecast peak | 10 min | All per-class SLOs green; no 5xx; HPA scales within stabilisation window |
| Over-forecast (2×) | 2× forecast peak | 15 min with ramp | Service stays up; SLOs may breach; 5xx stays below 1%; 429s shed batch before chat |

The over-forecast stage verifies *graceful* degradation. The admission
controller from Exercise 04 should shed batch before chat; the HPA from
Exercise 05 should scale up; the fleet from Exercise 03 should have
headroom — this stage tests all three in composition.

### 2.5 What "capacity headroom" means in the report

The output of the test is a single number per workload class:

```
headroom_c = (arrival_rate at which class c's p95 SLO broke) / forecast_peak_c
```

A headroom of 1.4 means the class survived 40% over forecast before the
p95 crossed the SLO. The capacity review wants this number to be ≥1.3
for interactive classes and ≥1.1 for batch classes at the forecast
horizon. A headroom below 1.0 means the forecast is already under-sized
and the plan needs rework before it is signed off.

## 3. Validation steps

1. **Open-model arrival generator.** Not closed-loop VU-count.
2. **Streaming TTFT measured at first content byte**, not at response
   completion or headers.
3. **Per-class metrics**, each with its own threshold matching the
   class's SLO.
4. **Three-stage run** with pass/fail gates at each stage.
5. **Over-forecast stage verifies graceful degradation**, not just
   "performance goes down" — it must verify shed order and that the
   service stays up.
6. **Length distribution matches production** (mean and p95, not a
   single fixed value).
7. **Run outputs a headroom number per class.**
8. **Server-side observability is scraped during the run** so a client
   miss can be root-caused (dashboard link captured in the run
   artifact).
9. **Cost of the run is bounded.** Model-provider spend is tracked and
   under a budget — load tests against hosted models can get expensive
   quickly.

## 4. Rubric / review checklist

Score out of 10. Pass = 7+.

| # | Item | Points |
|---|---|---|
| 1 | Open-model arrival generator (k6 `constant-arrival-rate` / `ramping-arrival-rate`, or Locust `ArrivalRateUser`); closed-loop VU-count, `constant_throughput`, or `constant_pacing` not accepted for streaming workloads | 1 |
| 2 | Streaming-aware TTFT at first content byte; TPOT over inter-token gaps | 2 |
| 3 | Per-class metrics and per-class thresholds | 2 |
| 4 | Three-stage design (baseline / forecast / over-forecast) with gates | 2 |
| 5 | Over-forecast stage verifies shed-order and service stays up | 1 |
| 6 | Prompt length distribution matches production (mean + p95) | 1 |
| 7 | Headroom number reported per class | 1 |

Deductions:

- Closed-loop VU model: −3.
- TTFT measured at response completion rather than first byte: −3.
- Global averaged metrics only (no per-class slice): −2.
- No over-forecast stage: −2.
- No server-side observability captured: −1.
- Fixed single prompt length: −1.
- Model-provider spend uncapped: −1.

## 5. Common mistakes

- **Closed-loop load.** Each VU waits for its response before sending
  the next request, so arrivals are back-pressured by latency. Real
  users are not that polite; the test under-reports queueing and over-
  reports TPS capacity.
- **TTFT at response completion.** For a streaming endpoint this
  measures `TTFT + N · TPOT`, not TTFT. The number is wrong in both
  level and shape.
- **One prompt.** A single benchmark prompt produces a single benchmark
  answer. Prefill cost is linear in prompt length; one prompt validates
  nothing about the long-prompt tail.
- **No per-class breakdown.** A global p95 hides a batch class
  dragging the number down and a chat class breaching.
- **Treating 429s as a pass.** A 429 is a failure of capacity planning,
  not a successful shed. Shed is what the admission controller does;
  429 is what gets to the client. The test should count 429s as errors.
- **Running the over-forecast stage against a self-hosted fleet without
  checking facilities.** 2× forecast at high prompt length can draw
  more power than a rack can supply. Coordinate with facilities.
- **Ignoring model-provider cost.** A sustained over-forecast stage
  against a hosted provider can burn through a month of budget in an
  afternoon. Cap it.
- **Not warming the fleet before the baseline stage.** The first 60 s
  look like the service is terrible; the number is really the measure
  of cold-start, which should be a separate experiment.
- **No version pin on the inference service.** Capacity numbers are
  only comparable across runs if the serving container digest and the
  model weights are pinned.

## 6. References

- k6 documentation — scenarios, executors (`constant-arrival-rate`,
  `ramping-arrival-rate`), `Trend` / `Rate` / `Counter` custom metrics,
  and thresholds — <https://grafana.com/docs/k6/latest/>.
- Locust documentation — `FastHttpUser`, `constant_throughput`,
  `events.request.fire`, and the open-vs-closed-model discussion —
  <https://docs.locust.io/>.
- Google SRE Workbook — *Non-Abstract Large System Design* and
  *Managing Load*: load testing as the verification step of a capacity
  plan, graceful degradation, shed-order —
  <https://sre.google/workbook/>.
- NIST AI Risk Management Framework (AI RMF 1.0) — Measure function
  (continuous testing against defined performance metrics informs the
  per-class SLO threshold design) —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- Local exercise context: this load test is the empirical verification
  of the base model (Exercise 01), the forecast worksheet (Exercise 02),
  the fleet plan (Exercise 03), the provider quota plan (Exercise 04),
  and the HPA tuning (Exercise 05). A capacity plan that has not passed
  this test is not a plan; it is a hypothesis.
