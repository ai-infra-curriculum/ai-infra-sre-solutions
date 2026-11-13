# exercise-06 — Top-Ten Failure-Class Mock Triage Walkthroughs — Solution

> The learning exercise prompt is a `TBD` stub. This solution interprets the exercise against the module `README.md` objective — "walk mock incidents end-to-end for the top ten failure classes — surface, diagnose, mitigate, hand off — with trace / metric / log artifacts for each" — and chapters 01–07. The deliverable is ten short end-to-end walkthroughs, one per class, each with the same six-field shape: **surface → diagnose → classify → mitigate → hand off → retrospective note**.

> Each walkthrough is intentionally brief (≈ one page). They are training scaffolds, not real incidents. The signals, metric names, and timings are illustrative (chapter 08 is still a `TBD` stub in the learning repo; see the `follow-up-research` marker at the bottom).

---

## 1. Solution overview

Ten classes, ten walkthroughs. Each walkthrough is written in the shape a retrospective takes — a surface narrative, the signals the on-call read, the classification they declared, the lever they pulled, the peers they handed off to, and the one retrospective lesson worth carrying forward.

The walkthroughs are useful *as a set*: a learner who reads all ten in sequence has seen the whole taxonomy applied end-to-end and has a pattern library to match the next real incident against.

Every walkthrough's **classification** cites the chapter that authoritatively defines the class so the training loop stays grounded.

## 2. Implementation — worked answers (ten walkthroughs)

### Walkthrough 1 — Class 2.1 Silent output drift (sub-class 1.1 weight change, provider alias move)

**Surface.** `chat-support-v1`, hosted-provider path, `gen_ai.request.model = gpt-4o`.

**Diagnose (first 15 min).**

- 03:42 UTC — SLO burn alert on factuality quality SLI; 14× burn over 30-day window.
- Dashboard: `gen_ai.response.model` label flipped from `gpt-4o-2024-08-06` to `gpt-4o-2024-11-20` in a 10-minute window at 03:20 UTC. No deploy on our side at 03:20.
- Canary-eval pass-rate on the hourly `canary-set/v3` run at 03:30 UTC dropped from baseline 0.91 to 0.79.
- Output-length distribution shifted; new snapshot is more verbose by ~15% on the same inputs.
- Cost per request rose ~12% on the slice.
- TTFT / TPOT unchanged (same provider backend; inference path the same).
- Provider status page clean.

**Classify.** Class **2.1** — silent output drift, sub-class **1.1 model-weight change**, provenance **provider alias move**. Chapter 03 §1.1. Also composite with **2.9** (provider's behaviour changed without a contract change) — chapter 07 §2.5.

**Mitigate (SRE lane).**

- Pin `gen_ai.request.model` from the floating alias `gpt-4o` to the dated snapshot `gpt-4o-2024-08-06` on the request path.
- Verify next hourly canary returns to baseline on the pinned path; verify quality SLI stops burning.
- Mitigation complete at 04:03 UTC (21 minutes from alert).

**Hand off.**

- Vendor-risk workflow (`mod-107`) owns the provider conversation — "your alias moved without notification; what is your pinning policy; what was in the new snapshot."
- ML platform owns the longer-term pinning policy and the eval-peer's re-baselining against both snapshots.
- Application owns the "should we migrate forward to the new snapshot after eval re-baselines" decision.

**Retrospective note.** The floating alias is an operational hazard; chapter 07 §2.8 names "version-pinned requests by default" as the architectural guidance. The action item is a stack-wide audit of which surfaces use floating aliases and a policy of defaulting to dated snapshots.

External vocabulary for the retrospective: NIST AI 600-1 *Confabulation* (the factuality burn is the user-visible impact) + *Value chain and component integration*. MIT Domain 7 (AI system safety, failures, and limitations); OECD *Robustness, security and safety*. Chapter 02 cross-walk.

---

### Walkthrough 2 — Class 2.2 Hallucination spike (compound with 2.6)

**Surface.** `rag-docs-v2`, groundedness quality SLI.

**Diagnose.**

- 10:15 UTC — SLO burn alert on groundedness SLI (fast window, 10-minute burn hit 14×).
- Dashboard: no `gen_ai.response.model` label change; no deploy on application side in the window.
- Retrieval panel: retrieval-grounded-answer pass-rate dropped from 0.88 to 0.61. Retrieval-score distribution of top-k scores shifted bimodal.
- Document-age-at-retrieval histogram p95 unchanged.
- Ingestion-freshness SLI burning: `mod-112` ingestion pipeline failed at 09:40 UTC; latest batch of docs incorrectly re-chunked.
- Spot-check 15 retrieved documents: many are garbage fragments (truncated mid-sentence, encoding artifacts).
- Judge-vs-human agreement unchanged in the last window; 2.8 ruled out.

**Classify.** Primary **2.6** retrieval-freshness regression, sub-class **2.6.c document corruption**, chapter 05 §2.4. Hallucination burn (2.2) is the downstream symptom — chapter 05 §3.2 names this masquerade pattern.

**Mitigate.**

- Isolate the corrupted ingestion batch; identify the chunker change that caused the truncation.
- Roll back the ingestion-pipeline chunker change; re-ingest the affected sources.
- Short-term: route retrieval queries to the previous index snapshot (if the system has one) or throttle the surface for the window.
- Mitigation complete at 11:04 UTC.

**Hand off.** Data / config pipeline peer (`mod-112`) owns the ingestion-pipeline forensics (what in the chunker change caused the corruption). Application layer owns any query-path change if retrieval rewrite is involved. Eval peer re-runs the groundedness eval against the healed index.

**Retrospective note.** Groundedness burn with unchanged model and unchanged prompt is a retrieval-side flag; the matrix's disambiguator ("retrieval-score shift + document corruption signal") narrowed in under 10 minutes. Action item: a pre-ingestion-promotion canary that checks chunk-length distribution against a reference.

External vocabulary: NIST *Information integrity*. MIT *Misinformation* and *AI system safety*. OECD *Robustness*.

---

### Walkthrough 3 — Class 2.3 Toxic / unsafe blast (declared incident, minute 1)

**Surface.** `creative-writer-v1`, open-domain.

**Diagnose.**

- 20:08 UTC — safety-category counter spike: `safety.category = self_harm_adjacent` rate jumped from a baseline of ~0.0001 to 0.009 over a 5-minute window.
- External: three user reports filed in the product-support queue in the last 15 minutes.
- Model-version label: provider `gen_ai.response.model` moved 3 hours ago from `provider-x-2026-09-15` to `provider-x-2026-11-10`.
- Prompt-injection surface scan: several of the flagged inputs match a known injection pattern catalogued by `mod-110`.

**Classify.** **Minute 1: declared incident per chapter 04 §7.** Class **2.3** toxic / unsafe output blast. Composite with **2.1** (provider alias move) → class **2.10** (policy / safety regression). Two of three flagged sessions are attributable to the model's weaker safety training on the new snapshot; one is attributable to a prompt-injection input.

**Mitigate.**

- Short-circuit the output path for the `self_harm_adjacent` category: route to a canned safe response + resource referral.
- Pin provider model to the previous dated snapshot on this surface.
- Rate-limit the input class matching the injection pattern pending `mod-110` review.
- Preserve all affected session logs (do not delete) under the retention-and-disclosure procedure.

**Hand off.**

- Security-safety-compliance (`mod-110`) peer (primary).
- Vendor-risk (`mod-107`) peer — provider's safety training on the new snapshot is weaker on this category.
- Legal / PR / executive channel per `mod-110` severity matrix.
- Red-team peer to evaluate the injection pattern.

**Retrospective note.** The 2-minute declared-incident posture matters: the SRE declared at minute 1 even though the absolute rate was small, which started the compliance clock with the right state. Preserving the session logs under the right retention policy was done pre-triage. Action items: (a) provider-snapshot-pinning policy for creative surfaces; (b) a pre-production safety-category eval against new provider snapshots before they can route user traffic.

External vocabulary: NIST *Dangerous, violent, or hateful content* / *Obscene, degrading, and/or abusive content* and *Value chain*; MIT *Discrimination and toxicity*; OECD *Human-centred values and fairness*. Chapter 02.

---

### Walkthrough 4 — Class 2.4 Refusal-rate spike

**Surface.** `legal-doc-assistant-v1`, enterprise tenant.

**Diagnose.**

- 08:30 UTC — refusal-SLI burn alert; `gen_ai.response.finish_reasons = content_filter` counter up 6× on this surface.
- `gen_ai.response.model` label moved 2 hours ago: provider snapshot alias flipped.
- Prompt / system-message deployment history: no change in the window.
- Content-filter threshold on our side: unchanged.
- Spot-check: 15 refused inputs are all on-scope ("summarise clause 7 of this contract", "explain fiduciary duty in Delaware"); the model is refusing on legitimate legal-assistant work.
- Over-cautious side signal: user session abandonment rate up 2.1× on the surface.

**Classify.** Class **2.4** refusal-rate spike, composite with **2.1** (provider alias move). Chapter 04 §1.3.

**Mitigate.**

- Pin the provider snapshot on this surface's request path to the previous dated version.
- Verify refusal-counter returns to baseline on the pinned path; verify session-abandonment metric relaxes.
- If the new snapshot must be rolled forward for business reasons, hand off the prompt-redesign work to the application layer with the eval peer in loop.

**Hand off.** Vendor-risk (`mod-107`) + application / prompt layer + eval peer (re-baseline helpfulness rubric against the new snapshot). This surface has a tightly regulated SLO; include the compliance-contact in the hand-off.

**Retrospective note.** Refusal spikes often co-occur with model-version changes; the dashboard panel that overlays `gen_ai.response.model` with the refusal SLI (chapter 04 §6) narrowed classification in under 10 minutes. Over-cautious refusal — not explicit-refusal — is the harder dimension to detect and requires the session-abandonment side signal.

External vocabulary: NIST *Human-AI configuration* (over-refusal impairs legitimate use); MIT *Human-computer interaction*; OECD *Inclusive growth*; AIID CSET *Access / service denial*.

---

### Walkthrough 5 — Class 2.5 Prompt-cache cliff (sub-class 2.5.c tokenizer change)

**Surface.** `qa-assistant-v3`, self-hosted vLLM.

**Diagnose.**

- 02:40 UTC — joint-signal alert: `vllm_cache_prefix_hit_rate` dropped 0.82 → 0.11; TTFT p95 regressed 280ms → 1 300ms; cost per request rose; all on the same surface, same 10-minute window.
- No application-code deploy; deploy marker is a nightly image rebuild.
- Image diff: `transformers==4.46.1 → 4.47.0`; tokenizer library pin shifted inside the dependency tree.
- `gen_ai.response.model` unchanged; prompt text unchanged.
- Concurrent: retrieval-grounded-answer passage rate on the surface started a slow burn → 2.6.b embedding-drift companion (chapter 05 §3.1).

**Classify.** Class **2.5** prompt-cache cliff, sub-class **2.5.c tokenizer change**; compound with **2.6.b** embedding drift. Chapter 05 §1.2.3 and §3.1.

**Mitigate.**

- Roll back the container image to the previous `transformers` pin.
- Pin the tokenizer version explicitly in the application's dependency pin file.
- Verify cache-hit, TTFT, retrieval-grounded-answer pass-rate return to baseline.
- Mitigation complete at 03:22 UTC.

**Hand off.** Inference-engine peer (engine-side tokenizer plumbing + cache implementation) + application-layer owner (dependency pin policy). Vendor-risk if the tokenizer shift came from a provider SDK rather than in-stack.

**Retrospective note.** The cliff came from a dependency bump nobody connected to the cache; the source-control diff that caused it did not touch the prompt text. Action item: a `pinned-transitive-deps.lock` policy for the tokenizer and embedding-model libraries, and a CI check that fails the image rebuild if those pins drift.

External vocabulary: NIST has no direct category for cache cliffs; the retrospective frames the user-visible impact (cost / latency). MIT *AI system safety, failures, and limitations*.

---

### Walkthrough 6 — Class 2.6 Retrieval-freshness regression (sub-class 2.6.b embedding drift)

**Surface.** `product-search-rag-v1`.

**Diagnose.**

- 16:00 UTC — retrieval-grounded-answer passage rate dropped from 0.85 to 0.42 over 30 minutes.
- Retrieval-score distribution shifted bimodal; p95 of top-k scores halved.
- Document-age histogram p95 unchanged — not staleness.
- Query-path embedding model: upgraded by the ML platform at 15:40 UTC from `text-embedding-3-small` to `text-embedding-3-large`. Index-build embedding model: still `text-embedding-3-small` on the retrieval index.
- Spot-check retrieved documents: documents exist, scores are low, retrieved documents are irrelevant to query.

**Classify.** Class **2.6** retrieval-freshness regression, sub-class **2.6.b embedding drift**, specifically the query-side change pattern. Chapter 05 §2.3.

**Mitigate.**

- Pin the query-path embedding model back to `text-embedding-3-small` on this surface.
- Schedule a background re-embed of the index under `text-embedding-3-large`; swap atomically once complete.
- Mitigation (query-side pin) complete at 16:42 UTC; full re-embed scheduled over the next 18 hours.

**Hand off.** ML platform peer (embedding-model promotion policy; the query-and-index-embedder-must-match invariant); application / prompt layer (query-path pipeline); data / config pipeline for the re-embed orchestration.

**Retrospective note.** The failure shape — "the index is not detectably broken, queries return results, recall collapses" (chapter 05 §2.3) — is the hardest of the three 2.6 sub-classes to recognise without the retrieval-score-distribution signal. Action item: an embedder-version invariant check in the gateway that fails fast when query-path and index-path diverge.

External vocabulary: NIST *Information integrity*; MIT *Misinformation* and *AI system safety*; OECD *Robustness; Transparency*.

---

### Walkthrough 7 — Class 2.7 Agent cascade (sub-class 2.7.a recursive loop)

**Surface.** `rag-research-agent-v2`.

**Diagnose.**

- 14:12 UTC — agent-cascade joint alert: `agent_session_tool_call_depth_bucket` p99 shifted from 4 to 28 over 10 minutes; `agent_session_cost_usd` p99 doubled.
- Sampled trace `s_01` (vignette A from exercise-04): 28 iterations of `search_documents` with slightly reworded queries; agent never converged; hit the depth cap at 28.
- Deploy diff: prompt system-message edited 2 hours ago — a termination condition was *removed* in favour of a looser "keep searching until you have enough information."
- No tool errors; 429s absent. Not 2.7.b.
- No schema-validation errors. Not 2.7.c.

**Classify.** Class **2.7** agent cascade, sub-class **2.7.a recursive loop**. Chapter 06 §2.1.

**Mitigate.**

- Lower the agent's depth cap on this surface from 40 to 8 (bounds blast radius).
- Roll back the system-message prompt change; restore the previous termination condition.
- Throttle the surface briefly during the rollback propagation.

**Hand off.** Application / prompt-layer owner (prompt / termination condition) + agent-framework owner (depth-cap default policy).

**Retrospective note.** The cost blow-up (chapter 06 §8) was token-explosion: each loop iteration re-read the full accumulated conversation. The per-session cost cap engaged on three of the cascading sessions, which bounded the financial blast but caused user-visible truncation. Action item: a prompt-review gate that rejects edits removing or weakening termination conditions on agent surfaces.

External vocabulary: NIST *Value chain and component integration*; AIID CSET *Autonomous system failure; runaway cost*; MIT *AI system safety*; OECD *Robustness; Accountability*.

---

### Walkthrough 8 — Class 2.8 Judge-model drift

**Surface.** Across-surface quality SLI dashboard.

**Diagnose.**

- 11:00 UTC — multiple surfaces' quality SLIs burn simultaneously. No deploy on any surface in the window. `gen_ai.response.model` labels unchanged across the board.
- Judge-version pin: the eval peer rolled a new judge model (`claude-opus-4-7 → claude-opus-4-8`) at 10:30 UTC.
- Judge prompt / rubric: unchanged.
- Judge-vs-human agreement, most recent calibration window (yesterday): Cohen's κ = 0.78, which was in-baseline.
- Spot-check 20 outputs the new judge marked as failing: 17 of 20 look fine to a human reviewer. The judge has drifted.

**Classify.** Class **2.8** judge-model drift. Chapter 07 §1 Pattern A — the judge changed, the model did not.

**Mitigate (SRE lane: HOLD).**

- Do **not** roll any model under test. Do not edit any prompt. Do not route-around.
- Declare the signal suspect in the incident channel.
- Pin the judge back to `claude-opus-4-7` (if eval peer is available to confirm; otherwise flag the SLI as uncalibrated for the window pending their confirmation).
- SLO retro-adjustment per `mod-103` chapter 05 §5.7 for the affected window.

**Hand off.** Eval peer owns the judge calibration, the decision whether to adopt `claude-opus-4-8` after re-calibration, and the human-annotation re-baselining.

**Retrospective note.** Judge drift is a *measurement* incident, not a service incident. The SRE's correct response is to recognise the class and hold; rolling the model would have made no reliability improvement and cost engineering time. Chapter 07 §1.4 is the canonical framing. Action item: tighter judge-change policy — any judge promotion goes through a calibration run before it replaces the current judge; pre-promotion agreement κ must not have dropped.

External vocabulary: NIST *Human-AI configuration* (measurement); MIT *AI system safety*; OECD *Transparency; Accountability*. The retrospective explicitly records that the affected window's quality numbers are suspect so auditors are not misled.

---

### Walkthrough 9 — Class 2.9 Provider-degraded mode

**Surface.** `summariser-v2`, two providers behind a gateway (A primary, B fallback).

**Diagnose.**

- 22:30 UTC — TTFT / TPOT regression on `gen_ai.system = provider-A` × `gen_ai.response.model = provider-a-standard`. p95 TTFT up 2.3×; TPOT up 1.8×.
- Judge-pass rate on the same slice dropped from 0.90 to 0.77.
- Provider A status page: clean, no declared incident.
- `gen_ai.response.model` label on the slice: unchanged.
- Triangulation: provider B slice unaffected; our self-hosted slice unaffected; other users on public forums reporting similar.
- Canary set (chapter 03) re-run against provider A reproduces the regression; same canary against provider B does not.

**Classify.** Class **2.9** provider-degraded mode. Chapter 07 §2. No composite with 2.1 because `gen_ai.response.model` is stable — provider has not moved the snapshot, they have degraded the backend.

**Mitigate.**

- Route around: the multi-provider gateway shifts primary traffic from provider A to provider B for the affected surface. Fallback quality has been pre-baselined on the canary set, so the quality impact of the shift is known-bounded.
- Pin provider A requests that must remain on A (for contractual reasons) to the dated snapshot (already the case).
- Scale the self-hosted fallback as a defense-in-depth measure.

**Hand off.** Vendor-risk (`mod-107`) opens an evidence-bearing support ticket with provider A: slice, timing, canary reproduction, public-signal corroboration. Internal comms: incident channel; external user comms per the "do not name the provider" guidance in chapter 07 §2.7.

**Retrospective note.** The architectural investment — multi-provider gateway, pre-baselined fallback — was what made the mitigation a 5-minute traffic shift rather than a surface-wide outage. Action item: validate the self-hosted fallback's canary monthly so it remains known-working.

External vocabulary: NIST *Value chain and component integration*; AIID CSET *Provider dependency failure*; MIT *AI system safety*; OECD *Robustness; Accountability*.

---

### Walkthrough 10 — Class 2.10 Policy / safety regression (composite 2.1 + 2.3 + 2.4)

**Surface.** `healthcare-chat-v1`, regulated jurisdiction.

**Diagnose.**

- 06:20 UTC — concurrent signals:
  - `safety.category = medical_advice_adjacent` counter spiked (2.3 signal).
  - `gen_ai.response.finish_reasons = content_filter` counter spiked on different queries (2.4 signal).
  - `gen_ai.response.model` label flipped 4 hours ago from `provider-y-2026-10-15` to `provider-y-2026-11-20` (2.1 signal).
- The 2.3 and 2.4 counters are on *different* query classes — the model now simultaneously refuses more legitimate medical-factoid questions *and* produces more borderline medical-advice answers where it previously deferred. Chapter 04 §2 — the trade-off surface tightened on one axis and loosened on another.

**Classify.** Class **2.10** policy / safety regression. Minute-1 declared incident per chapter 01 §2.10 + chapter 04 §7. The composition 2.1 + 2.3 + 2.4 fires concurrently; this is the chapter 01 §2.10 condition exactly.

**Mitigate.**

- Pin the provider snapshot to the previous dated version for this surface immediately.
- Short-circuit the `medical_advice_adjacent` category to a canned "please consult a licensed clinician" response pending review.
- Preserve all affected session logs under compliance retention.
- Rate-limit the input classes matching the refusal-spike pattern to bound the user-abandonment blast.

**Hand off.**

- Security-safety-compliance (`mod-110`) primary.
- Vendor-risk (`mod-107`) — provider shipped a safety-training change without notification; contractual-SLA conversation starts here.
- Legal / compliance (jurisdiction-specific healthcare regulator) per `mod-110` severity matrix.
- Application / prompt layer owns the "should we migrate forward to the new snapshot after safety eval re-baselines" decision.

**Retrospective note.** 2.10 is almost always a provider-snapshot incident; the floating-alias policy from walkthrough 1 and walkthrough 9 is the root prevention. Action items: (a) stack-wide dated-snapshot policy for regulated surfaces; (b) a pre-production safety eval gate that must pass before any new provider snapshot can take regulated traffic; (c) documented minute-1 compliance comms procedure that is practiced quarterly.

External vocabulary: NIST *Dangerous, violent, or hateful content* (where applicable to the category) + *Human-AI configuration* (over-refusal) + *Value chain*; MIT composite; OECD *Human-centred values; Robustness*. The composite is cited as such in the retrospective — chapter 01 §2.10.

---

## 3. Validation steps

A grader walks these checks on the submitted set of ten walkthroughs.

1. **All ten walkthroughs present, one per class.** Including 2.10 as composite; no merging.
2. **Every walkthrough has all six fields.** Surface, diagnose, classify, mitigate, hand off, retrospective note.
3. **Signals are specific.** Metric names, label values, deltas (not "quality went down"). Where the canonical metric name depends on a stack, the walkthrough names it illustratively and references the `mod-104` chapter it came from.
4. **Classification cites the chapter that authorities the class.** 2.1 → chapter 03, 2.2/2.3/2.4 → chapter 04, 2.5/2.6 → chapter 05, 2.7 → chapter 06, 2.8/2.9 → chapter 07, 2.10 → chapter 01 §2.10.
5. **Mitigation stays in the SRE lane.** Rollback / pin / route-around / throttle / short-circuit / depth-cap. No judge rubric edits, no prompt rewrites at 3 AM, no filter-threshold edits at 3 AM.
6. **2.3 and 2.10 walkthroughs are declared-incident from minute 1.** Includes legal / compliance / executive comms per `mod-110`.
7. **2.8 walkthrough is "hold", not rollback.**
8. **Compound-incident flags are raised** where chapter 01 §3 or chapters 03–07 name a compound (2.1+2.9, 2.2+2.6, 2.5+2.6, 2.3+2.1+2.4=2.10, 2.4+2.1, 2.7+2.2/2.5/2.9).
9. **Retrospective note has an action item.** Each walkthrough names at least one action, typically preventative.
10. **External vocabulary is translated.** Each retrospective note cites NIST, MIT, OECD, and/or AIID per chapter 02 §5 cross-walk.

## 4. Rubric / review checklist

0–3 per criterion; 30 points total; pass threshold 24. Score across all ten walkthroughs using min() per criterion (one broken walkthrough drags the score).

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Ten walkthroughs present, class names canonical | <10 or merged | 7–9 | 10, minor rename | 10 canonical |
| 2 | All six fields per walkthrough | <4 fields | 4–5 fields | 6 fields | 6 fields + sub-class where applicable |
| 3 | Signals are specific (metric, label, delta) | Vague | Named without values | Named with values | Named with values + chapter cross-reference |
| 4 | Classification cites authoring chapter | Absent | Half of walkthroughs | Most | All |
| 5 | Mitigation stays in SRE lane | Out-of-lane | Mostly in-lane | In-lane | In-lane + explicit peer hand-off for the fix |
| 6 | Declared-incident framing for 2.3 and 2.10 | Missing | Mentioned | Minute-1 framing | Minute-1 + `mod-110` comms posture |
| 7 | 2.8 walkthrough is hold | Rollback used | Mixed | Hold + pin judge | Hold + pin + SLO retro-adjustment reference |
| 8 | Compound-incident flags raised | None | 1 flagged | 2–3 flagged | All chapter-named compounds flagged |
| 9 | Retrospective note has an action item | None | Vague | Specific action | Specific action tied to a preventative control |
| 10 | External vocabulary per chapter 02 | Absent | One source cited | Two sources | All four (NIST / AIID / OECD / MIT) where chapter 02 §5 has entries |

## 5. Common mistakes

- **Writing walkthroughs that are only about the diagnosis.** The six-field shape (surface → diagnose → classify → mitigate → hand off → retrospective note) makes the retrospective practice reusable; a walkthrough that stops after "we classified it" misses the operational-and-compliance arc.
- **Classifying 2.10 as a tenth independent class.** It is explicitly a composite of 2.1 + 2.3 + 2.4 — chapter 01 §2.10. The walkthrough should carry the composite classification.
- **Skipping the compound-incident flags.** Chapter 01 §3 says explicitly: "a real incident will often belong to two classes." The on-call declares the first-route class but adds more as the incident unfolds. A walkthrough that treats each incident as a single class misses this discipline.
- **Rolling the model in walkthrough 8.** That is the headline failure mode of 2.8 recognition. The lesson of walkthrough 8 is to *not* roll the model when the judge is suspect.
- **Omitting the retrospective action item.** Walkthroughs without a preventative action are entertainment, not training. Every walkthrough should surface at least one process / instrumentation / policy change that would have prevented or shortened the incident.
- **Using "rollback" as the lever for a provider alias move without pinning.** Rollback restores the floating alias to its current pointer, which the provider may move again tomorrow. The lever is **pin to a dated snapshot**, not generic rollback.
- **Treating the 2.3 incident as minutes-to-classify.** The declared-incident posture is minute-1; the on-call declares *before* classifying the full composite. The minute-1 action starts the compliance clock and preserves evidence; downstream classification refines.
- **Not naming the hand-off peers.** Hand-off is the entire point of recognise-and-route. A walkthrough whose "hand off" field is "route to the AI team" fails the routability constraint from chapter 01 §3.
- **Inventing NIST / MIT / AIID labels.** The external-vocabulary field cites the chapter 02 §5 cross-walk; where the chapter already carries a `follow-up-research` marker, the walkthrough may propagate the marker but must not invent category names.

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/01-ai-specific-failure-taxonomy.md` — ten classes; compound-incident framing (§3); hardware exclusion (§4); working-document discipline (§6).
  - `lessons/mod-105-non-deterministic-failure-diagnosis/02-grounding-in-nist-aiid-oecd-mit.md` — composite cross-walk table (§5); worked retrospective passage (§7).
  - `lessons/mod-105-non-deterministic-failure-diagnosis/03-silent-output-drift-and-canary-evals.md` — class 2.1.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/04-hallucination-toxic-refusal-spikes.md` — classes 2.2 / 2.3 / 2.4.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/05-prompt-cache-cliffs-and-retrieval-regressions.md` — classes 2.5 / 2.6.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/06-agent-tool-call-cascades.md` — class 2.7.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/07-judge-drift-and-provider-degraded-mode.md` — classes 2.8 / 2.9.
  - `lessons/mod-103-slo-engineering-for-model-backed-services/` chapters 02 (quality SLI), 04 (cost SLI), 05 (multi-window multi-burn-rate; §5.7 SLO retro-adjustment), 06 (SLO ritual).
  - `lessons/mod-104-observability-for-llm-and-gpu-workloads/` chapters 02 (Prometheus metric conventions), 04 (LLM-trace platform), 05 (drift pipelines), 07 (dashboard authoring and deploy-marker overlay).
  - `lessons/mod-107-vendor-and-model-provider-risk/` — vendor-risk workflow (provider conversations, SLA credits, pinning policy).
  - `lessons/mod-108-incident-response-for-ai-systems/` — incident lifecycle (retrospective template; comms policy).
  - `lessons/mod-110-security-safety-and-compliance-reliability-crossover/` — declared-incident comms for 2.3 / 2.10; preservation-and-disclosure procedures.
  - `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/` — ingestion-freshness SLI (walkthrough 2 / 6).
- Authoritative-project references:
  - NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>.
  - NIST AI 600-1 (Generative AI Profile) — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>.
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>.
  - MITRE ATLAS — <https://atlas.mitre.org/>.
- Community research references (cite as research, not regulatory authority):
  - AI Incident Database — <https://incidentdatabase.ai/>.
  - OECD.AI Incidents Monitor — <https://oecd.ai/en/incidents>.
  - MIT AI Risk Repository — <https://airisk.mit.edu/>.

<!-- follow-up-research: the chapter 08 of the learning repo (`08-triage-decision-matrix-and-ten-walkthroughs.md`) is still a scaffolded stub at the time of writing. Each walkthrough above is reconstructed from chapters 01–07; when chapter 08 lands with its authoritative ten end-to-end walkthroughs (with real trace / metric / log artifacts), align the signal values, exact metric names, and retrospective external-vocabulary citations to the published text. The structure here (six-field shape; minute-1 declared-incident for 2.3/2.10; hold-not-rollback for 2.8) is load-bearing; the specific illustrative timings and deltas are not. -->
