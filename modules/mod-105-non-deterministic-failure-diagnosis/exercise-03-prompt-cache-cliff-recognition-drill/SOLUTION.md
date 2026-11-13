# exercise-03 — Prompt Cache Cliff Recognition Drill — Solution

> The learning exercise prompt is a `TBD` stub. This solution interprets the exercise against chapter `05-prompt-cache-cliffs-and-retrieval-regressions.md` section 1, chapter `01-ai-specific-failure-taxonomy.md` section 2.5, and chapter `02` of `mod-104` (prompt-cache metric naming). The drill is a recognition exercise: given a short incident vignette with trace / metric fragments, the learner must (a) confirm the joint signal, (b) classify the sub-class (prefix / template / tokenizer), (c) propose a mitigation on the correct reversible lever, and (d) hand off with the right evidence envelope.

---

## 1. Solution overview

Prompt-cache cliff (class 2.5) is a two-signal-joint recognition. One signal alone — a cache-hit-rate drop, a TTFT regression, or a cost shift — misclassifies against many legitimate events: traffic-shape changes, scheduler reconfigs, autoscaler events. The **joint** signal — cache-hit rate down **and** TTFT up **and** input-token cost up on the same slice and the same deploy boundary — is what classifies the incident.

Three sub-classes resolve to three *diff* targets:

- **Prefix invalidation (2.5.a)** — the prompt / system-message diff is where the invalidating token is.
- **Template change (2.5.b)** — the template structural diff moves every request.
- **Tokenizer change (2.5.c)** — the tokenizer / SDK / gateway dependency diff is the source-control change nobody connected to the cache; the prompt text diff is often empty.

The drill tests whether a learner can recognise the joint signal, match the diff shape to the sub-class, and route to the correct peer — and, critically, avoid the trap of treating the incident as a pure "performance incident" and reaching for an autoscaler lever that does not restore the cache.

## 2. Implementation — worked answer: three vignettes with model classifications

The exercise's intent is that the instructor supplies vignettes; this solution provides three canonical ones that cover the three sub-classes. A learner's submission that constructs similar vignettes and classifies them correctly is passing; an instructor using this as a graded drill can swap the vignettes and reuse the rubric.

### 2.1 Vignette A — prefix invalidation

**Incident fragment.**

- 09:00 UTC, deploy marker on `chat-support-v1` surface.
- 09:02 UTC, `vllm_cache_prefix_hit_rate{surface="chat-support-v1"}` drops from `0.78` to `0.04`.
- 09:03 UTC, TTFT p95 on the same surface regresses from 320 ms to 1 400 ms.
- 09:05 UTC, input-token spend per request rises from a baseline of ~1 100 tokens cached + 180 uncached to ~0 cached + 1 280 uncached.
- Output-length distribution unchanged; finish-reason mix unchanged; `gen_ai.response.model` unchanged; engine image tag unchanged.
- Deploy diff: `src/prompts/chat_support/system.md` — added a leading line `"Today's date: 2026-11-12"` and a line `"You are talking to {{user_first_name}}"` to the system message.

**Model classification.**

Joint signal present: cache-hit rate ↓, TTFT ↑, input-token cost ↑, all on the same deploy marker. Confirm class **2.5**.

Sub-class: **2.5.a prefix invalidation**. Evidence: the deploy diff is a prompt / system-message change, not a template structural rewrite, and no dependency / tokenizer movement. The specific invalidating pattern is characteristic — a *date* that changes daily (global invalidation every 24 h) and a *per-request variable* (`user_first_name`) injected into the prefix (per-request invalidation).

**Why not 2.5.b?** The template structure (system + user-turn + context slot + response) is unchanged; the content of the system prefix changed. The chapter 05 section 1.2.1 framing — "something in the prefix changed, position-dependent" — fits.

**Why not 2.5.c?** No dependency bump, no SDK bump, no gateway update, no `gen_ai.response.model` change. The tokenizer version is stable.

**Mitigation lever.** Prefix invalidation's correct lever is *move the varying content to a suffix slot*: move the date and the user-name injection out of the system prefix into the user turn. This restores most of the cache within minutes as the next wave of requests re-populate. Rollback of the prompt change is also valid; cache-warm is the alternative if neither move is possible immediately.

**Route-to.** Application / prompt-layer owner (the system-message editor).

**Evidence envelope.** Sub-class classification; joint-signal screenshots (cache-hit rate, TTFT p95, input-token cost on same window with deploy marker); prompt diff; mitigation in place; blast radius (what fraction of surface traffic is affected — in this case all of it, because the prefix is global).

### 2.2 Vignette B — template change

**Incident fragment.**

- 14:00 UTC, deploy marker on `rag-docs-v2` surface.
- 14:01 UTC, `vllm_cache_prefix_hit_rate{surface="rag-docs-v2"}` drops from `0.65` to `0.00`.
- 14:02 UTC, TTFT p95 regresses from 450 ms to 2 100 ms.
- 14:04 UTC, cost per request rises; p99 session-cost histogram shifts right.
- Output-length distribution shifted slightly rightward (new template elicits longer completions); hallucination-rate quality SLI starts a slow burn (not yet SLO breach).
- Deploy diff: `src/prompts/rag_docs/template.yaml` — restructured; XML tags added around context block; system message reordered; few-shot examples regenerated.

**Model classification.**

Joint signal present. Confirm class **2.5**.

Sub-class: **2.5.b template change**. Evidence: the deploy diff is a *structural* rewrite — reordered sections, added XML scaffolding, regenerated few-shots. No prompt-text-only tweak. The cache-hit rate goes to near-zero (not merely low) because no cache entry at all overlaps the new structure's prefix.

**Compound-incident callout.** The output-length shift and the hallucination-rate slow-burn are expected concomitants of a template change — the new template elicits different model behaviour. This is the chapter 05 section 3.3 case: a template rollout correlates with a hallucination-rate change *and* a cache cliff. Treat as a single compound incident and watch 2.2 for progression.

**Mitigation lever.** Roll back the template to the previous version, *or* configure the engine to warm the cache against the new template before the next traffic wave (if the quality side of the template change is a required delivery). Autoscaling is not the lever — adding replicas does not warm the cache.

**Route-to.** Application / prompt-layer owner (the template owner); flag the compound to the eval peer for the groundedness regression if the hallucination burn continues.

**Evidence envelope.** Sub-class classification; joint-signal fragments; template structural diff; hallucination-SLI slow-burn marker; mitigation in place; blast radius (every request on the surface).

### 2.3 Vignette C — tokenizer change

**Incident fragment.**

- 02:30 UTC, nightly image rebuild landed on `qa-assistant-v3` serving pod image. No application-code deploy on the surface.
- 02:40 UTC, `vllm_cache_prefix_hit_rate{surface="qa-assistant-v3"}` drops from `0.82` to `0.11`.
- 02:42 UTC, TTFT p95 regresses from 280 ms to 1 300 ms.
- 02:45 UTC, cost per request rises.
- Output-length distribution unchanged; finish-reason mix unchanged; `gen_ai.response.model` unchanged; prompt text unchanged.
- Container image diff: `transformers==4.46.1 → 4.47.0`; tokenizer library pin shifted inside the dependency tree (BPE merge-table minor update affecting unicode normalisation of a handful of tokens).
- Retrieval-grounded-answer passage rate on the surface starts a slow burn concurrently.

**Model classification.**

Joint signal present. Confirm class **2.5**.

Sub-class: **2.5.c tokenizer change**. Evidence: no prompt diff at all; no template diff; the source-control change is a *dependency bump* in an image rebuild. The prompt text is byte-identical to yesterday; the token IDs are not.

**Compound-incident callout.** The concurrent retrieval-grounded-answer slow-burn is the chapter 05 section 3.1 case: a tokenizer change commonly also breaks retrieval, because the query-path embedding model is tokenised by the same library. Expect both 2.5 and 2.6.b (embedding drift) to fire; route both to the same dependency/platform owner.

**Mitigation lever.** Roll back the image to the previous `transformers` pin. Pin the tokenizer version explicitly in the application's dependency pin file so subsequent transitive bumps cannot drift it. (Provider-hosted: pin the tokenizer version in the provider-SDK configuration if the provider exposes it.)

**Route-to.** Inference-engine peer (engine-side tokenizer plumbing) and application-layer owner (dependency pin). Vendor-risk if the tokenizer shift came from a provider SDK rather than an in-stack dependency.

**Evidence envelope.** Sub-class classification; joint-signal fragments; image / dependency diff; retrieval-grounded-answer slow-burn marker (compound); mitigation in place.

### 2.4 Non-vignette — the ambiguous / provider-side case

A short additional case worth teaching: no deploy diff of any kind, and still the joint signal fires. Chapter 05 section 1.4 step 3: "no diff on your side → provider-side invalidation; escalate per `mod-107`." The learner's answer here is to document the absence of an internal change, cross-reference against the provider's status page and community channels, and open the vendor ticket with the evidence envelope. The sub-class is "provider-side invalidation"; it is a 2.5 incident but the owner is vendor-risk, not an in-stack peer.

### 2.5 The dashboard panel the drill assumes

Per chapter 05 section 4, the panel rows the learner's dashboard must support for this drill:

| Row | Signal | Covers |
|---|---|---|
| 1 | Prefix-cache hit-rate (per engine / per model / per surface) | 2.5 |
| 2 | TTFT p95 (per surface) | 2.5 + 2.1 kernel |
| 3 | Input-token cost per request (per surface) | 2.5 + 2.1 |
| 4 | Deploy / template / dependency-bump / image-rebuild markers | overlay |
| 5 | Prompt / system-message diff viewer link | overlay for 2.5.a / 2.5.b |
| 6 | Container-image / SDK / dependency version table | overlay for 2.5.c |
| 7 | Output-length distribution + retrieval-grounded-answer passage rate | compound-incident flags |

## 3. Validation steps

A grader (or the learner during self-review) walks these checks on each vignette answer.

1. **Joint signal confirmation.** The learner names cache-hit rate, TTFT, and input-token cost together before classifying. Single-signal classification is a fail.
2. **Deploy-boundary identification.** The learner names the deploy / template / dependency / image event at the timestamp of the joint-signal shift. If the vignette has no internal diff, the learner escalates to provider-side investigation instead of inventing one.
3. **Sub-class classification.** 2.5.a vs. 2.5.b vs. 2.5.c vs. provider-side. The learner names the diff shape that justifies the sub-class.
4. **Mitigation lever is sub-class-appropriate.**
   - 2.5.a → move varying content to suffix, or roll back the prompt.
   - 2.5.b → roll back template, or cache-warm against new template.
   - 2.5.c → roll back the tokenizer / library / SDK bump and pin.
   - Provider-side → route-around if possible; otherwise absorb the cache-rebuild window.
5. **Autoscaling is not named as the mitigation.** Scaling does not warm the cache. The chapter 05 section 1.6 "easy to miss" framing is explicit: 2.5 presents as a performance / cost incident but is a content-plumbing incident; mitigating the symptom with scaling is a failure mode.
6. **Compound-incident flags raised.** Vignette B should flag the hallucination-SLI co-movement; vignette C should flag the retrieval-grounded-answer co-movement.
7. **Routing is peer-specific.** Application-layer owner for 2.5.a / 2.5.b; inference-engine peer + application for 2.5.c; vendor-risk for provider-side.
8. **Evidence envelope is complete.** Sub-class, joint-signal fragments, diff, mitigation in place, blast radius.
9. **No "quality SLI didn't fire" is used as a reason to deprioritise.** Chapter 05 section 1.6: if the operator only watches quality and availability, the cliff lives for days.
10. **No invented engine metric names.** Learners should use their stack's canonical names; `vllm_cache_prefix_hit_rate` is illustrative here (chapter 02 of `mod-104` enumerates per-engine names).

## 4. Rubric / review checklist

0–3 per criterion; 30 points total; pass threshold 24. Score per vignette, aggregate with min() across vignettes (a learner who passes two of three is not passing).

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Joint signal confirmed before classifying | Single-signal | Two of three | All three named | All three + explicit "on the same slice and same deploy boundary" |
| 2 | Deploy boundary identified or absence acknowledged | Missed | Named but not aligned | Aligned | Aligned + the event type (prompt / template / dependency / image / provider-side) is named |
| 3 | Sub-class classification matches diff shape | Wrong | Right class, wrong sub-class | Right sub-class | Right sub-class + rules out the other two |
| 4 | Mitigation lever is sub-class-appropriate | Wrong (autoscale, more capacity) | Generic rollback | Sub-class-appropriate | Sub-class-appropriate + alternative-lever named |
| 5 | Compound-incident flag raised where applicable | Missed | Noted, not followed | Noted + routing adjusted | Noted + routing adjusted + chapter-05 §3 cross-reference |
| 6 | Routing is peer-specific | "The team" | One peer | Right peer | Right peer + composition where chapter 05 §5 names it |
| 7 | Evidence envelope complete | Missing | 1–2 fields | 3–4 fields | All 5 fields (sub-class, joint signal fragments, diff, mitigation, blast radius) |
| 8 | Neither autoscaling nor 2.2/2.9 named as the mitigation | Autoscale named | Quality-side rollback used | Infra lever used | Infra lever used + rationale references content-plumbing framing |
| 9 | Provider-side case handled without inventing an internal cause | Invented | Hand-waved | Named as provider-side | Named + vendor-risk routing + `mod-107` reference |
| 10 | No invented metric names | Fabricated | Some invented | Mostly canonical / marked illustrative | Fully canonical / marked illustrative with chapter-02 reference |

## 5. Common mistakes

- **Alerting / classifying on cache-hit rate alone.** Many legitimate events drop cache-hit rate — a surge of new user queries, a canary expansion, a scheduler-reconfig that changes batching. The joint matters.
- **Reaching for the autoscaler.** TTFT is up; the instinct is capacity. Adding replicas does not help; the cache is cold on *every* replica. Chapter 05 section 1.6 names this as the headline failure mode of cache-cliff recognition.
- **Classifying a template change as a prefix invalidation.** The distinguishing shape: prefix invalidation has a diff *within* the existing template; template change replaces the template structure. Prefix invalidation keeps some cache entries valid for some request classes; template change takes cache-hit to ~zero.
- **Missing the tokenizer-change sub-class.** It is the most pernicious because the source-control diff that caused it (a dependency bump) does not touch the prompt text. Learners who only check prompt diffs misclassify 2.5.c as "no diff — provider-side."
- **Treating a tokenizer-change incident as a 2.5-only incident.** The concurrent 2.6.b (embedding drift) often fires. Chapter 05 section 3.1 is explicit.
- **Omitting the deploy marker.** The deploy marker is what pins "same deploy boundary." Without it, the joint-signal confirmation is just coincidental correlation.
- **Routing a tokenizer-change incident to the application-layer owner only.** The engine peer owns the engine-side cache implementation and tokenizer plumbing; the application owner owns the dependency pin. Both are on the hand-off.
- **Rolling back an engine image to mitigate a prefix invalidation.** Wrong lever; the prompt is the regression, not the engine. Learners sometimes reach for the engine rollback because it is a "deploy rollback" and the deploy marker was on the prompt.

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/05-prompt-cache-cliffs-and-retrieval-regressions.md` — the authoritative in-module source; section 1 (cache cliff), section 1.2 (sub-classes), section 1.3 (joint signals), section 1.4 (first fifteen minutes), section 1.5 (mitigation and route), section 1.6 ("why this class is easy to miss"), section 3 (interactions with retrieval-side), section 4 (dashboard panel), section 5 (peer boundary).
  - `lessons/mod-105-non-deterministic-failure-diagnosis/01-ai-specific-failure-taxonomy.md` section 2.5 — the one-paragraph class summary.
  - `lessons/mod-104-observability-for-llm-and-gpu-workloads/02-...` — Prometheus engine metric enumeration (per-engine cache-hit metric names).
  - `lessons/mod-103-slo-engineering-for-model-backed-services/` chapters 01 (TTFT SLI) and 04 (cost SLI).
  - `lessons/mod-107-vendor-and-model-provider-risk/` — provider-side invalidation escalation.
- Authoritative-project references:
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. (Adjacent coverage of dependency-bump-style failures; the tokenizer-change sub-class touches supply-chain hygiene.)
  - NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Cache cliffs do not have a direct NIST AI 600-1 category; the retrospective frames the user-visible impact (cost / latency) rather than the NIST category.
  - MITRE ATLAS — <https://atlas.mitre.org/>. (Not the primary source here; included for completeness with the module's standing source list.)

<!-- follow-up-research: engine-specific cache-hit metric names vary. `vllm_cache_prefix_hit_rate` is illustrative; `mod-104` chapter 02 enumerates the canonical names for vLLM, SGLang, TRT-LLM, Triton, NVIDIA Dynamo, and hosted-provider stacks. Before shipping this drill into a cohort's dashboard, verify the exact name against the engine in use. -->
