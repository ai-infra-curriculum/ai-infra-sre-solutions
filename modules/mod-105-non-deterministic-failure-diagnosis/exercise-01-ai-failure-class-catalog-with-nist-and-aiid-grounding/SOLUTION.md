# exercise-01 — AI Failure Class Catalog with NIST and AIID Grounding — Solution

> The learning exercise prompt is a scaffolded `TBD` stub at the time of writing. This solution interprets the exercise against the module's stated objectives (lessons `README.md` and chapters `01-ai-specific-failure-taxonomy.md`, `02-grounding-in-nist-aiid-oecd-mit.md`): produce a ten-class catalog of AI-specific failures, each entry naming its primary signal, suspect owner, time-to-recognition budget, and the authoritative-source entry (NIST AI RMF GenAI Profile, AIID / CSET, OECD.AI, MIT AI Risk Repository) that grounds it.

---

## 1. Solution overview

The deliverable is a **ten-row catalog table** plus a short written rationale. Each row answers five operational questions the on-call has at minute two of an incident:

1. What do I call this failure in the incident channel?
2. What is the single signal that confirms the class?
3. Who do I route to?
4. What is the external-taxonomy name a regulator or auditor will recognise?
5. How long should classification take before I escalate for dashboard / training gaps?

The catalog **is not a research artifact**. It is working on-call vocabulary that happens to carry its own compliance mapping. Anything that would require pulling a PDF at 3 AM belongs in the retrospective template, not the catalog.

A learner who completes the exercise should be able to recite the ten class names from memory, name one signal per class without opening a dashboard, name one owning peer per class without opening a directory, and point to the NIST / AIID / OECD / MIT row that the retrospective will cite.

## 2. Implementation — worked answer

### 2.1 The ten-class catalog

| # | Class name | Primary signal | Suspect owner(s) | Recognition budget | NIST AI 600-1 category | AIID / CSET axis | OECD principle | MIT domain |
|---|---|---|---|---|---|---|---|---|
| 2.1 | **Silent output drift** (weight / kernel / numeric) | Canary-eval pass-rate drop; output-embedding centroid shift; `gen_ai.response.model` label change | ML platform; inference-engine peer; provider | 10 min | *Information integrity* (indirect) | Performance degradation | Robustness, security and safety | AI system safety, failures, and limitations |
| 2.2 | **Hallucination-rate spike** | Factuality / groundedness quality SLI burn | Application / prompt layer; eval peer; retrieval; model layer | 5 min | *Confabulation* | Information harm — incorrect information | Transparency; Robustness | Misinformation; AI system safety |
| 2.3 | **Toxic / unsafe output blast** | Safety-category counter spike; external report escalation | Security-safety-compliance (`mod-110`); application; model; red-team | 2 min *(declared incident)* | *Dangerous, violent, or hateful content* and/or *Obscene, degrading, and/or abusive content* | Discrimination / hate / harmful content | Human-centred values and fairness | Discrimination and toxicity |
| 2.4 | **Refusal-rate spike** | `gen_ai.response.finish_reasons = content_filter` counter; refusal-SLI burn | Model layer; application layer; eval peer | 10 min | *Human-AI configuration* (over-refusal) | Access / service denial | Inclusive growth; Human-centred values | Human-computer interaction |
| 2.5 | **Prompt-cache cliff** (prefix / template / tokenizer) | Prefix-cache-hit-rate drop + TTFT regression + input-token cost rise on same deploy | Inference-engine peer; application / prompt layer; provider | 5 min | None direct (infra-side) | None direct | None direct | AI system safety, failures, and limitations |
| 2.6 | **Retrieval-freshness regression** (staleness / embedding drift / corruption) | Retrieval-grounded-answer pass-rate drop + retrieval-score-distribution shift + document-age / corruption signal | Data / config pipeline (`mod-112`); ML platform; application | 15 min | *Information integrity* | Information harm — outdated information | Robustness; Transparency | Misinformation; AI system safety |
| 2.7 | **Agent-tool-call cascade** (recursion / rate-limit / schema) | Tool-call-depth histogram p99 shift + per-session cost / wall-clock shift | Application / framework layer; tool owner; inference-engine peer | 10 min | *Value chain and component integration* and/or *Information security* | Autonomous system failure; runaway cost | Robustness; Accountability | AI system safety, failures, and limitations |
| 2.8 | **Judge-model drift** | Judge-vs-human agreement slip; judge version / prompt / rubric diff | Eval peer | 30 min | *Human-AI configuration* (measurement) | Measurement / evaluation failure | Transparency; Accountability | AI system safety, failures, and limitations |
| 2.9 | **Provider-degraded mode** | TTFT / TPOT / quality shift on the `gen_ai.system=<provider>` × `gen_ai.response.model=<id>` slice, with status page clean | Vendor-risk (`mod-107`); provider; SRE fallback routing | 20 min | *Value chain and component integration* | Provider dependency failure | Robustness; Accountability | AI system safety, failures, and limitations |
| 2.10 | **Policy / safety regression** (composite 2.1 + 2.3 + 2.4) | Either 2.3 or 2.4 fires concurrently with a `gen_ai.response.model` label change | Security-safety-compliance; vendor-risk; application layer | 5 min | Composite (whichever content category applies; plus *Value chain* if provider-caused) | Composite | Composite | Composite |

### 2.2 Written rationale — three constraints the catalog enforces

Submitters should justify the class list on the three constraints the lesson text names:

- **Mutually distinguishable at triage time.** No two classes share a single confirming signal. A prompt-cache cliff is confirmed on the cache-hit / TTFT / cost joint; a silent weight drift is confirmed on canary-eval + response-model label; an agent cascade is confirmed on tool-call-depth. A learner writing the catalog should be able to say for any pair which single dashboard panel separates them.
- **Routable to a distinct peer or composition.** Every row names an owner. "The AI team" is not an owner. If a class routes to more than one peer, the composition is named explicitly (2.3 goes to security-safety-compliance *and* vendor-risk *and* application; 2.10 is a declared composite).
- **Grounded.** Every row cites at least one authoritative source. Where no direct mapping exists (2.5 cache cliff has no NIST / AIID / OECD home), the row says so instead of inventing a category.

### 2.3 Example retrospective passage

A learner's catalog should carry a short worked-example retrospective entry showing how the operational class name translates into the external vocabulary. The example from chapter 02 section 7 is the model:

> On 2026-Mon-DD at HH:MM UTC the quality SLI for `chat-support-v1` breached budget at 14× burn. On-call declared **class 2.1 — silent output drift**, sub-class model-weight change, at HH:MM+7. Under NIST AI 600-1 this is a *Confabulation* risk event. Under the MIT AI Risk Repository it is a Domain 7 (AI system safety, failures, and limitations) event, subdomain *robustness failure*. Under the OECD AI Principles the primary principle violated is *Robustness, security and safety*. Closest AIID pattern: recommender-system model-update drift cases.

The operational name (`class 2.1`) goes in the incident channel. The external names go in the retrospective. Both live in the same document.

## 3. Validation steps

A learner — or the grader — walks these checks against the submitted catalog. Each check is yes / no, no interpretation needed.

1. **Ten rows, ten class names.** No merging. No fewer than ten. If the learner collapsed e.g. 2.3 and 2.4 into a single "content safety" row, that is a fail — the trade-off surface the lesson names (section 2 of chapter 04) requires the separation.
2. **Each row has a primary signal** that is one line and references a concrete metric, label, counter, or distribution — not a feeling ("quality went down"). The signal should be gettable from the `mod-104` observability stack without new instrumentation.
3. **Each row has at least one suspect owner** named from the peer directory, not from generic team names. "The ML platform peer (`ai-infra-ml-platform`)" passes; "the backend team" does not.
4. **Each row has a recognition budget** between 2 min and 30 min, consistent with the chapter 01 section 5 table.
5. **Each row has at least one NIST AI 600-1 category**, or a documented reason ("no direct mapping; infra-side").
6. **Each row has at least one AIID / CSET axis**, an OECD principle, and an MIT domain *or* a documented reason.
7. **The 2.3 row calls the class a declared-incident class** and names the 2-minute budget. This is a load-bearing operational distinction the chapter 04 section 7 comms-posture discussion requires.
8. **The 2.10 row is identified as a composite**, not as a tenth independent class.
9. **The worked retrospective example translates the operational name into the external one.** A grader should be able to read the example without the operational vocabulary and still understand the incident class in NIST / MIT terms.
10. **No invented facts.** Each cited source entry appears in the chapter 02 cross-walk table or in the authoritative source's own documentation. Where the chapter text carries a `follow-up-research` marker (exact NIST / MIT / CSET category labels), the catalog may propagate the same marker rather than paraphrase around it.

## 4. Rubric / review checklist

Scoring is 0–3 per criterion; 30 points total; pass threshold 24.

| # | Criterion | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| 1 | Ten classes present, each named as in chapter 01 | Missing classes or re-naming | Six–eight present | Nine present, minor renaming | All ten present, canonical names |
| 2 | Primary signal is concrete and operationally gettable | Vague ("quality drops") | Named but not sliceable | Named + sliceable | Named + sliceable + joint-signal awareness where chapter 01 flags it (2.5, 2.6, 2.7) |
| 3 | Suspect-owner routing is peer-specific | "The team" | One peer per class, some generic | One or more peers per class, named from the peer directory | Composition captured where chapter 01 names a composition (2.3, 2.10) |
| 4 | Recognition budget is present and defensible | Absent | Present but uniform | Per-class, within guidance | Per-class, with a one-line justification |
| 5 | NIST AI 600-1 mapping | Absent | Arbitrary mapping | Chapter-02-aligned mapping | Chapter-02-aligned + "none direct" called out honestly for 2.5 and others |
| 6 | AIID / CSET, OECD, MIT mappings | Absent | One of three present | Two of three present | All three present or honest "no direct match" |
| 7 | 2.3 flagged as declared-incident class with 2-minute budget | Not flagged | Mentioned | Flagged with budget | Flagged + comms-posture note referencing `mod-110` |
| 8 | 2.10 identified as composite | Treated as independent | Called a composite without mapping to 2.1/2.3/2.4 | Called a composite with partial mapping | Called a composite with the 2.1 + 2.3 + 2.4 composition explicit |
| 9 | Worked retrospective example translates operational → external vocabulary | Absent | Operational only | Both vocabularies but not integrated | Both vocabularies integrated into a single narrative |
| 10 | No invented facts; `follow-up-research` carried forward where chapter 02 flagged it | Invented NIST / MIT labels | Occasional invention | Mostly grounded | Fully grounded, markers propagated |

## 5. Common mistakes

- **Collapsing 2.3 and 2.4 into one "safety" class.** They are mirror images with opposing trade-offs; collapsing them hides the regression a safety-training update produces on the opposite axis.
- **Treating 2.1 as a tree.** Weight / kernel / numeric are *sub-classes with different owners*. The row should name all three and route accordingly.
- **Naming "the AI team" or "the platform team" as the owner.** The point of routing is specificity; generic ownership defeats the exercise.
- **Inventing NIST AI 600-1 categories.** The published GenAI Profile is a specific list; only the categories the chapter 02 mapping cites (or the Profile itself cites) may be named. A `follow-up-research` marker is acceptable; invented verbiage is not.
- **Omitting the recognition budget.** The budget is the operational signal that tells the on-call rotation whether the dashboarding or training has a gap. Without it, the catalog is a reading list.
- **Writing a two-paragraph narrative instead of the table.** The catalog *is* the table. Supplemental narrative is fine; replacing the table with prose is not — the operator needs a row they can scan at 3 AM.
- **Not calling out 2.10 as a composite.** It is listed tenth in the chapter because the composition is the operational handling, not because it is the tenth independent class.
- **Treating the external-source mappings as one-to-one.** They are often many-to-one or none-to-one. Chapter 02 is explicit; the catalog should be too.

## 6. References

- Lesson chapters (local):
  - `lessons/mod-105-non-deterministic-failure-diagnosis/01-ai-specific-failure-taxonomy.md` — the ten classes, their primary signals, suspect owners, and recognition budgets.
  - `lessons/mod-105-non-deterministic-failure-diagnosis/02-grounding-in-nist-aiid-oecd-mit.md` — the composite cross-walk table (section 5) and the worked retrospective passage (section 7).
  - `lessons/mod-105-non-deterministic-failure-diagnosis/README.md` — module-level learning objectives.
- Official standards and projects (authoritative):
  - NIST AI Risk Management Framework landing page — <https://www.nist.gov/itl/ai-risk-management-framework>.
  - NIST AI 600-1 (Generative AI Profile) knowledge-base entry — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>.
  - OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. (Adjacent coverage of ML-specific failure vocabulary; use as a secondary grounding source where the catalog touches input-attack-adjacent classes such as 2.3 prompt-injection surfaces.)
  - MITRE ATLAS — <https://atlas.mitre.org/>. (Adversary-view counterpart to the harm-view sources; useful where the catalog's 2.3 or 2.10 entries touch adversarial inputs.)
- Community research resources (cite as research, not regulatory authority):
  - AI Incident Database — <https://incidentdatabase.ai/>.
  - OECD.AI Incidents Monitor — <https://oecd.ai/en/incidents>; OECD AI Principles — <https://oecd.ai/en/ai-principles>.
  - MIT AI Risk Repository — <https://airisk.mit.edu/>.

<!-- follow-up-research: before shipping this catalog into a compliance-grade audit response, re-verify each NIST AI 600-1 category name, each MIT Domain name, and each AIID / CSET harm-axis name against the current versions of the respective publications. The chapter-02 text already flags these three cross-walks as requiring verification against the latest errata; the catalog here inherits those markers rather than resolving them. -->
