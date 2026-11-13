# exercise-04 — Agent Cascade Recognition with Tool-Call Depth Histogram — Solution

> The learning exercise prompt is a `TBD` stub. This solution interprets the exercise against chapter `06-agent-tool-call-cascades.md` and `01-ai-specific-failure-taxonomy.md` section 2.7. The deliverable has two parts: (1) a specification of the tool-call-depth histogram and its joint alert surface, and (2) a trace-driven recognition drill in which the learner classifies three cascading sessions into the three sub-classes (recursive loop, rate-limit-on-nested-tool, mis-plumbed schema) and chooses the correct reversible lever.

---

## 1. Solution overview

Agent-tool-call cascade (class 2.7) is the one failure class in the taxonomy where the user-visible correctness signal is *fine* — the agent eventually produces an answer — and the reliability failure is entirely cost, latency, and burn on provider or GPU capacity. The on-call who watches only quality SLIs will see no alert; the on-call who watches cost will see a blow-up but will not know where to route it. The signals this chapter authors — tool-call-depth histogram, per-tool error rate, per-session cost, per-session wall-clock — are therefore the *detection mechanism* for the class.

Recognition has three steps:

1. **Joint alert.** Depth-histogram p99 shifted upward (or depth-cap hit count increased) **and** per-session cost or wall-clock shifted upward **on the same surface**.
2. **Sample a cascading trace.** Pull a representative high-depth trace from the trace store and read the tree.
3. **Classify the sub-class from the trace shape.**
   - Same tool called N times with similar args → **2.7.a recursive loop**.
   - Same tool called N times all erroring with 429-shaped errors → **2.7.b rate-limit on nested tool**.
   - Same tool called N times all erroring with schema-validation errors → **2.7.c mis-plumbed schema**.
   - Chain of different tools each erroring → structured-output conformance issue or environment-side pattern.

The exercise's deliverable is a histogram specification + alert rules + a drill that exercises trace reading.

## 2. Implementation — worked answer

### 2.1 Tool-call-depth histogram specification

**Emission.** Each agent framework span tree emits one `agent.session.tool_call_depth` sample per session, where depth is the longest path from the root LLM-call to a terminal tool span. Alternative for linear agents: the length of the tool-call sequence.

Labels on the sample:

- `agent.framework` — `"langgraph" | "openai-assistants" | "anthropic-tool-use" | "pydantic-ai" | "autogen" | "crewai" | "<in-house>" | ...`
- `agent.surface` — the product surface identifier (one per deployed agent).
- `agent.model` — the model behind the agent's LLM calls.
- `agent.session_id` — not a label (high-cardinality); on the span attribute set.

Illustrative Prometheus-style histogram name:

```
agent_session_tool_call_depth_bucket{framework="langgraph", surface="rag-agent-v2", model="claude-sonnet-4-6", le="1|2|3|5|10|20|40|100|+Inf"}
agent_session_depth_cap_hits_total{framework="langgraph", surface="rag-agent-v2"}
agent_session_tool_call_errors_total{framework="langgraph", surface="rag-agent-v2", tool_name="search_documents", error_class="429|schema_validation|timeout|generic"}
agent_session_cost_usd{framework="langgraph", surface="rag-agent-v2"}
agent_session_wall_clock_seconds{framework="langgraph", surface="rag-agent-v2"}
```

(Exact metric names and bucket boundaries depend on the pipeline; `mod-104` chapter 04 — LLM-trace platform — enumerates the corresponding span attribute set.)

**Healthy shape.** 70–90% of sessions in depth 1–3; tail in 4–10; sessions in depth > 10 are rare. The histogram is characteristic of the surface — an interactive chat agent's distribution differs from a batch research agent's. Capture the "known-good shape" per surface so alerts have a baseline.

**Observability traps (chapter 06 section 7) the spec must defuse.**

- **Child-span parenting.** Every tool-call span carries the parent span-ID. Without it, tree depth is uncomputable and the histogram is garbage.
- **Tool-call span vs. LLM-call span.** The depth metric counts *tool* spans, not *LLM* spans; both counts are useful but they are not the same.
- **Session ID as join key.** Multiple trace trees per session in some frameworks; the session ID must be on every span.
- **Nested agents.** A tool that itself is an agent opens a sub-surface; observability propagates through; cost and depth include the nested cost and depth.
- **Streaming visible-thinking agents.** Partial-run tool calls are distinguished from completed ones.

### 2.2 Alert rules (joint)

```yaml
# Primary: depth + cost or wall-clock, on the same surface
- alert: AgentCascadeJoint
  expr: |
    (
      (
        histogram_quantile(0.99, sum by (surface, le) (
          rate(agent_session_tool_call_depth_bucket[10m]))
        )
        > on(surface) group_left()
          agent_session_tool_call_depth_p99_baseline{surface=~".+"} * 1.5
      )
      or
      (
        rate(agent_session_depth_cap_hits_total[10m])
        > on(surface) group_left()
          agent_session_depth_cap_hits_baseline{surface=~".+"} * 2
      )
    )
    and
    (
      (
        histogram_quantile(0.99, sum by (surface, le) (
          rate(agent_session_cost_usd_bucket[10m]))
        )
        > on(surface) group_left()
          agent_session_cost_p99_baseline{surface=~".+"} * 1.5
      )
      or
      (
        histogram_quantile(0.95, sum by (surface, le) (
          rate(agent_session_wall_clock_seconds_bucket[10m]))
        )
        > on(surface) group_left()
          agent_session_wall_clock_p95_baseline{surface=~".+"} * 1.5
      )
    )
  for: 10m
  labels:
    severity: page
    class: 2.7
    route: agent-triage
  annotations:
    summary: "Agent-cascade joint: depth + cost or wall-clock shifted upward on same surface."
    runbook: "runbooks/agent-cascade-triage.md"

# Secondary (no joint required): per-tool error rate spike
- alert: AgentToolErrorRateSpike
  expr: |
    sum by (surface, tool_name, error_class) (
      rate(agent_session_tool_call_errors_total[5m])
    )
    > on(surface, tool_name, error_class) group_left()
      agent_tool_error_rate_baseline * 3
  for: 5m
  labels:
    severity: page
    class: 2.7
    route: agent-triage
  annotations:
    summary: "Per-tool error-rate spike on {{$labels.tool_name}} ({{$labels.error_class}}); cascade-adjacent."

# Capacity bollard: per-session cost hard cap
- alert: AgentSessionCostCapHit
  expr: increase(agent_session_cost_cap_hits_total[10m]) > 0
  for: 1m
  labels:
    severity: page
    class: 2.7
    route: agent-triage
  annotations:
    summary: "Agent session hit cost cap and was truncated — cascade blast radius bounded, investigate root cause."
```

Thresholds (`* 1.5`, `* 2`, `* 3`) are defensible starting points; the learner should document why. The joint pattern is non-negotiable.

### 2.3 The recognition drill — three trace vignettes

The instructor supplies trace fragments (one or two screens of trace tree per session). A learner's answer classifies each into a sub-class, names the lever, and writes the hand-off. Three canonical vignettes below cover the three sub-classes.

#### Vignette A — recursive loop (2.7.a)

Trace shape:

```
session s_01
├── llm.call #1 → tool_call: search_documents(q="kubernetes pod lifecycle events")
│   ├── tool.search_documents → 7 docs, mid-relevance
├── llm.call #2 → tool_call: search_documents(q="k8s pod life-cycle event ordering")
│   ├── tool.search_documents → 8 docs, mid-relevance
├── llm.call #3 → tool_call: search_documents(q="kubernetes pod state machine transitions")
│   ├── tool.search_documents → 6 docs, mid-relevance
├── llm.call #4 → tool_call: search_documents(q="pod container lifecycle hooks preStop")
│   ├── tool.search_documents → 9 docs, mid-relevance
... (24 more similar iterations) ...
└── llm.call #28 → tool_call: search_documents(...) → terminal answer at depth cap
```

Classification: **2.7.a recursive loop**. Same tool, slightly reworded arguments, middling results on every call, agent never deciding the results are sufficient. The exit is the depth cap, not convergence.

Suspect root cause: prompt-side (system message does not define a termination condition such as "if you have 5+ documents of relevance ≥ 0.6, synthesise"), or model-side (the model is not converging on this task), or environment-side (the documents the retrieval returns are not usable).

Lever: lower the depth cap (bounded blast radius) + rollback the most-recent prompt/tool-descriptor change + open a prompt-engineering ticket to strengthen the termination condition. Route-to: application / prompt-layer owner + agent-framework owner.

#### Vignette B — rate-limit on nested tool (2.7.b)

Trace shape:

```
session s_02
├── llm.call #1 → tool_call: web_search(q="2026 Q3 earnings AAPL")
│   └── tool.web_search → HTTP 429, "Too Many Requests"
├── llm.call #2 → tool_call: web_search(q="2026 Q3 earnings AAPL")
│   └── tool.web_search → HTTP 429
├── llm.call #3 → tool_call: web_search(q="2026 Q3 earnings AAPL")
│   └── tool.web_search → HTTP 429
... (34 more identical retries) ...
└── llm.call #37 → tool_call: web_search(...) → terminal "I couldn't find the information" at depth cap
```

Per-tool error metric: `agent_session_tool_call_errors_total{tool_name="web_search", error_class="429"}` spiked 10× over baseline on this surface over the last 30 minutes. Upstream `web_search` provider's status page: "Elevated rate-limiting due to capacity rebalance" (posted 20 min ago).

Classification: **2.7.b rate-limit on nested tool**. Same tool, same args, uniform 429 errors. The agent is interpreting 429 as transient-retry forever.

Suspect root cause: tool-client code interprets 429 as retry-forever; framework may auto-retry transparently; tool's error contract does not distinguish transient from permanent. Upstream is also legitimately rate-limited; cascade is partly a self-inflicted amplification.

Lever: short-circuit the `web_search` tool at the gateway for the incident window (route to a canned "search temporarily unavailable" response in the agent's system message), or route-around to a secondary search provider if configured; separately, lower depth cap; pin the retry-policy issue for the follow-up. Route-to: application / tool-integration owner + the nested-tool's owner (internal or external); vendor-risk if the tool is external and the rate-limit is a sustained provider-side change.

#### Vignette C — mis-plumbed tool schema (2.7.c)

Trace shape:

```
session s_03
├── llm.call #1 → tool_call: create_ticket(title="...", priority="high", due_date="2026-11-18")
│   └── tool.create_ticket → "schema_validation_error: unexpected property 'due_date'; expected 'due_at'"
├── llm.call #2 → tool_call: create_ticket(title="...", priority="high", due_date="2026-11-18")
│   └── tool.create_ticket → "schema_validation_error: unexpected property 'due_date'; expected 'due_at'"
... (18 more identical retries) ...
└── llm.call #20 → tool_call: create_ticket(...) → terminal "I was unable to create the ticket" at depth cap
```

Tool schema registry diff: `create_ticket` tool schema renamed `due_date` → `due_at` three days ago; the agent's tool descriptor cache / tool-registration config was not refreshed on the agent surface.

Classification: **2.7.c mis-plumbed schema**. Same tool, same (invalid) args, uniform schema-validation errors.

Suspect root cause: descriptor-sync pipeline between the tool registry and the agent's tool manifest did not refresh; or (alt) descriptor was refreshed but a cache in the agent framework held the old version; or (alt) the inference engine's structured-output guarantee is weaker than the tool expects and conformance is below spec on this schema.

Lever: roll back the tool-schema change (if business-reversible for the window) **or** push the refreshed descriptor to the agent surface via the sync pipeline. Also: lower the depth cap for the window. Route-to: application-layer / tool-descriptor-owner + tool owner. Engine peer if structured-output conformance is below spec.

### 2.4 Hand-off ticket template (chapter 06 section 6.3)

Every classification above flows into the same hand-off format:

```markdown
# Agent-cascade hand-off — class 2.7

## Classification
- Sub-class: 2.7.a | 2.7.b | 2.7.c | structured-output conformance
- Surface: <name>, framework: <name>, model: <label>

## Evidence
- Sampled trace IDs (3–5 representative cascading traces): s_01, s_02, s_03
- Depth-histogram diff (before vs. during): attached
- Per-tool error-rate breakdown by error_class: attached
- Joint-signal screenshot (depth + cost/wall-clock on same surface, deploy markers overlay): attached

## Blast radius
- Fraction of surface traffic cascading: <%>
- Cost impact: $<rate> per hour over baseline
- Wall-clock p99 shift: <seconds>
- User-visible impact: <none | degraded latency | truncated answers | ...>

## SRE mitigation in place
- Depth cap lowered: <old> → <new>
- Tool short-circuit: <yes/no>, which tool
- Rollback: <yes/no>, which change
- Throttle: <yes/no>, what rate

## Owning peer's investigation scope
- Application / prompt layer: termination condition; tool descriptor refresh
- Tool owner: schema evolution; 429 retry policy
- Model layer or provider: structured-output conformance; degraded reasoning (2.9 cross-ref)
```

## 3. Validation steps

1. **The histogram emission is specified down to label set.** `agent.framework`, `agent.surface`, `agent.model` minimum; bucket boundaries named; session-ID as attribute not label.
2. **Observability-trap defusers present.** Child-span parenting, tool-call vs. LLM-call distinction, session-ID join, nested-agent propagation, streaming-partial distinction — all five chapter-06-section-7 traps named.
3. **Alert is joint.** Depth OR depth-cap-hit-count, combined with cost OR wall-clock. Single-signal alert fails the check (except the secondary per-tool error-rate alert, which is intentionally single-signal because an immediate 429 spike is actionable without the joint).
4. **Capacity bollard present.** Per-session cost-cap alert that bounds blast radius.
5. **Three vignettes classified correctly.** 2.7.a, 2.7.b, 2.7.c distinguished by trace shape.
6. **Lever matches sub-class.**
   - 2.7.a → depth cap + prompt rollback + prompt-engineering ticket.
   - 2.7.b → short-circuit / route-around + retry-policy fix.
   - 2.7.c → descriptor refresh / schema rollback.
7. **Compound-incident flags raised.** Vignette A may trigger a 2.2 hallucination risk (the agent that cannot find the answer and extrapolates); vignette B likely cross-references 2.9 if the upstream is a critical provider; vignette C's structured-output conformance path routes to the engine peer. Chapter 06 section 9 is the cross-reference.
8. **Hand-off ticket is complete.** All five sections from the template; evidence envelope is specific (trace IDs, histogram diff, per-tool breakdown).
9. **Cost cap is framed as a reliability lever, not a cost lever.** Chapter 06 section 10 is explicit; per-session cost caps prevent one pathological session from starving the rest.
10. **Token-explosion failure named.** Learners should be aware that per-iteration cost compounding in long conversations is the real shape of 2.7's cost blow-up (chapter 06 section 8).

## 4. Rubric / review checklist

0–3 per criterion; 30 points total; pass threshold 24. Score per vignette, aggregate with min() across vignettes.

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Histogram emission specified | Missing | Name only | Name + labels | Name + labels + bucket boundaries + session-ID handling |
| 2 | Observability traps defused | None | 1–2 of 5 | 3–4 of 5 | All 5 of chapter 06 §7 |
| 3 | Primary alert is joint | Single-signal | Joint but weak | Joint on both axes | Joint + capacity bollard + secondary per-tool alert |
| 4 | Three sub-classes distinguished by trace shape | Not treated | One sub-class differentiated | Two differentiated | All three with trace-shape heuristics |
| 5 | Lever matches sub-class | Wrong lever | Generic rollback | Sub-class-appropriate | Sub-class-appropriate + alternative lever + rationale |
| 6 | Hand-off ticket complete | Missing | 1–2 sections | 3–4 sections | All 5 sections; evidence envelope is specific |
| 7 | Compound-incident flags raised | None | One cross-ref | Two cross-refs | All applicable (2.2 + 2.5 + 2.9) from chapter 06 §9 |
| 8 | Routing is peer-specific | "The team" | One peer | Right peer per sub-class | Peer composition where chapter 06 §6.2 names it |
| 9 | Per-session cost cap framed as reliability lever | Missing | Mentioned | Framed as reliability | Framed + ties to chapter 06 §10 capacity story |
| 10 | Token-explosion failure named | Missing | Mentioned | Explained | Explained + implication ("alert on per-session cost, not per-iteration") |

## 5. Common mistakes

- **Alerting on depth alone.** Depth shifts for legitimate workload changes (a new task class, a new tenant). The joint matters.
- **Counting LLM-call spans as tool-call-depth.** The agent loop has two spans per iteration; mixing them inflates the histogram and mis-classifies healthy sessions as cascading.
- **Missing the parent-span-ID requirement.** Orphaned tool spans are the single most common failure to measure depth — frameworks emit these more often than they should.
- **Treating the cost cap as a cost lever, not a reliability lever.** Chapter 06 section 10 is explicit: per-session caps prevent one pathological session from starving others. Framing it to leadership as "cost control" misses the reliability argument.
- **Classifying 2.7.b as 2.7.a.** The trace signature is identical (same tool, repeated calls) unless the operator reads the error class. Learners who skip the error-class breakdown misclassify half of 2.7.b incidents as 2.7.a.
- **Fixing 2.7.c by editing the agent's prompt.** The prompt is not the problem; the tool descriptor is stale. Prompt-engineering a workaround is a band-aid; descriptor refresh is the fix.
- **Scaling capacity to absorb a cascade.** Cascades are per-session blow-ups; capacity does not help a single session. The right lever is depth cap or short-circuit.
- **Not recognising nested agents.** A `summarise_results` tool that itself is an LLM call has its own depth and its own cost. Observability must propagate through or "cost per session" under-reports.
- **Assuming the quality SLI will catch the cascade.** It won't — the terminal answer is usually fine. Cost and depth are the detection surfaces.
- **Rolling back the model as the first lever.** Rare that the model is the root cause; typically prompt / descriptor / tool-client is. Model rollback is reserved for the 2.7 + 2.9 compound (provider-degraded reasoning).

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/06-agent-tool-call-cascades.md` — the authoritative in-module source; sections 1 (agent-surface shape), 2 (three root causes), 3 (signals), 4 (joint signals), 5 (first fifteen minutes), 6 (mitigation and route), 7 (observability traps), 8 (token-explosion), 9 (interactions with other classes), 10 (capacity), 11 (chaos program).
  - `lessons/mod-105-non-deterministic-failure-diagnosis/01-ai-specific-failure-taxonomy.md` section 2.7.
  - `lessons/mod-104-observability-for-llm-and-gpu-workloads/` chapter 04 — LLM-trace platform (Langfuse / Phoenix) span-attribute set.
  - `lessons/mod-103-slo-engineering-for-model-backed-services/` chapter 04 — cost SLI.
  - `lessons/mod-106-capacity-planning-for-ai-workloads/` — per-session caps and per-surface concurrency caps as capacity levers.
  - `lessons/mod-109-chaos-engineering-for-ai/` — fault-injection on tools, schema-drift injection, prompt-regression-tests on convergence.
- Authoritative-project references:
  - NIST AI 600-1 — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. *Value chain and component integration* is the risk category most-adjacent to agent-tool composition; *Information security* when the agent has uncontrolled tool access.
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Tool-call-related attack surfaces have adjacent OWASP ML coverage.
  - MITRE ATLAS — <https://atlas.mitre.org/>. Adversarial-view counterpart to the harm-view sources; useful when 2.7 is adversarially induced (e.g., prompt injection causing an agent to recursion-loop).

<!-- follow-up-research: before shipping the alert thresholds (p99 depth shift * 1.5, cost shift * 1.5, per-tool error rate * 3) into production, calibrate against the baseline shape of the specific surface; the chapter 06 text does not prescribe exact multipliers. -->
