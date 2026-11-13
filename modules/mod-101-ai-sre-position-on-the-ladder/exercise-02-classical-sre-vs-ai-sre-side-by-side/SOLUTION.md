# Exercise 02 — Classical SRE vs AI SRE Side-by-Side — Reference Solution

> Companion to `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-02-classical-sre-vs-ai-sre-side-by-side.md` in the paired learning repo.

## 1. Solution overview

This is a **comparison exercise**. The learner has to take each classical SRE practice (as established by Google's SRE book and the broader SRE tradition) and ask: *does it transfer unchanged, transfer with modification, or need a new practice entirely for AI/model-backed services?*

The failure mode for this exercise is a symmetrical table that reads like marketing copy — "classical: X, AI: X + AI" — without naming what actually changes. A strong submission names the **mechanism of change**: non-determinism, data-dependent failure, accelerator economics, external model-provider risk, or governance exposure.

Three categories of outcome per row:

- **Transfers unchanged.** The practice applies as-is. Example: service-level availability as a percentage of successful HTTP responses on the request path.
- **Transfers with modification.** The practice applies, but the SLI/SLO, error class, or operational rhythm changes. Example: correctness SLIs require eval harnesses and sampled human review, not just HTTP 5xx counting.
- **New or expanded practice.** AI introduces a concern that classical SRE does not address. Example: prompt-injection containment, model-drift monitoring, accelerator thermal/MIG planning.

## 2. Worked answer

### 2.1 Side-by-side table

| Practice area | Classical SRE | AI SRE | Delta category | Why the delta |
|---|---|---|---|---|
| **SLIs** | Request success, latency percentiles, availability. | Same request-path SLIs **plus** correctness/quality proxies (eval pass rate, refusal rate, user feedback rate), **plus** token-level SLIs (TTFT, tokens/sec, context-fit rate). | Modified | Output correctness is not derivable from HTTP status; quality is a first-class reliability signal. |
| **SLOs & error budgets** | Chosen against user-visible success. | Same, but split: a **serving SLO** (request-path) and a **quality SLO** (output correctness) with separate budgets because they fail independently. | Modified | A serving stack can be green while output quality regresses after a model swap. |
| **Monitoring & observability** | Logs, metrics, traces. | Logs, metrics, traces **plus** prompt/response sampling, drift metrics, GPU telemetry (utilization, memory, SM efficiency, NVLink), batch-queue depth. | Expanded | Failure modes live in the data/model/GPU layer, not just the request path. |
| **Incident response** | Page → mitigate → postmortem. | Same ritual; mitigations expanded to include **model rollback**, **route-around to a smaller/fallback model**, **prompt/guardrail hotpatch**, **temperature/sampling knob changes**. | Modified | Mitigation surface is larger and includes model-config changes, not just binary rollback. |
| **Postmortems / RCA** | Blameless, action items. | Same discipline; add **data-RCA** (did input distribution shift?), **model-RCA** (did an eval regress?), and **governance-RCA** (does this need a NIST AI RMF `MANAGE` update or a model-card change?). | Modified | Root causes are often data- or model-level, not code-level. |
| **Testing & release** | Unit, integration, canary, staged rollout. | Same **plus** offline eval suite as a gate, shadow traffic against candidate model, automatic rollback on correctness-SLI regression. | Expanded | Non-determinism makes classic diff-testing insufficient; eval-based gating is the new practice. |
| **Capacity planning** | CPU/RAM/disk/network vs forecast traffic. | Accelerator-first: GPU/TPU inventory, MIG partitions, KV-cache memory per request, batch-size planning, warm-pool for model reload, provider rate-limit capacity for hosted models. | Modified | Unit of capacity is a GPU-second with KV-cache constraints, not a CPU-second. |
| **Cost** | Usually finance-adjacent for SRE. | Cost is a **reliability constraint** because accelerator cost can force latency/quality tradeoffs under load (e.g., spill to smaller model). | Expanded | Cost directly shapes user-visible behavior during contention. |
| **Dependency management** | Upstream services, databases, caches. | All of the above **plus** third-party model providers (OpenAI, Anthropic, Bedrock, Vertex, etc.) with their own SLAs, rate limits, deprecation timelines, and model-version lifecycle. | Expanded | Model providers behave like a database vendor that may silently change the engine under you. |
| **Security** | AuthN/Z, secrets, patching, network. | Classical surface **plus** adversarial ML: prompt injection, model extraction, membership inference, data poisoning, evasion — mapped to MITRE ATLAS; broader risk mapped to OWASP ML Top 10. | Expanded | New attack surface not covered by classical AppSec threat models. |
| **Chaos engineering** | Kill pods, inject latency, drop packets. | Same **plus** model-level chaos: swap to a weaker model, inject high-latency token streaming, corrupt a retrieval source, poison a config-driven prompt. | Expanded | Reliability assumptions depend on model behavior, which has its own failure modes. |
| **Change management** | Code + config review and rollout. | Same **plus** model-weight changes, prompt/template changes, retrieval-index refreshes, tool definition changes — each is a release surface with its own rollback. | Expanded | "Release" is no longer only a container image. |
| **Governance & compliance** | Often a separate function. | Reliability events feed governance artifacts (risk register, model cards, AI RMF mappings). AI SRE contributes operational evidence. | New | Regulatory frameworks (NIST AI RMF and equivalents) make operational signals part of governance. |
| **Toil reduction** | Automate paging, runbooks, rollout. | Same **plus** automated eval gates, drift alerts, prompt-regression harnesses. | Modified | Toil surface includes "manually re-running evals" and "diffing prompts by hand." |

### 2.2 Transfers-unchanged shortlist

These transfer with essentially no modification:

- Error budgets as a policy device (freeze on burn).
- Postmortem culture (blameless, written, actioned).
- On-call shift/handoff discipline.
- SLO-driven prioritization.
- Request-path tracing and log correlation IDs.
- Toil budget and the Google 50% rule as a planning heuristic.
- Capacity-planning *process* (demand model → supply plan → review), even though the units change.

### 2.3 New-practice shortlist

Practices that have no classical equivalent and must be invented:

- **Offline eval as a release gate** with versioned datasets and acceptance thresholds.
- **Drift and distribution monitoring** on inputs, outputs, and user feedback.
- **Guardrail / safety-filter operations**: tuning, versioning, incident triage.
- **Adversarial-ML detection and response** keyed to MITRE ATLAS techniques.
- **Model-provider lifecycle management**: tracking model deprecations, A/B over versions, dual-sourcing.
- **Operational-evidence pipeline to governance** (NIST AI RMF `MEASURE`/`MANAGE` outputs).

## Implementation

To turn the side-by-side table into something the team actually uses:

1. **Start from the learner's current service, not an abstract one.** Open the service's runbooks, SLO doc, and release checklist, and for each practice listed there, locate the matching row in §2.1. The table is only useful as a gap finder for a specific service.
2. **Fill the table in a shared doc with three columns, not two.** The third column — *delta category* (`unchanged` / `modified` / `expanded` / `new`) — is what forces the mechanism-of-change discipline in §2 and prevents marketing-copy symmetry. Reject any row filled in without naming a mechanism.
3. **Cross-link each `new` row to a concrete backlog item.** The six entries in the "new practice shortlist" (§2.3) each imply work: eval-as-gate pipeline, drift monitors, guardrail ops, ATLAS detection coverage, model-provider lifecycle, governance evidence pipeline. For each, file or link an existing issue so the table is wired to execution rather than analysis.
4. **Attach the table to the reliability charter.** The charter from exercise 03 cites this delta analysis as the source for which practices need custom commitments (serving + quality SLOs, change control over ≥ 4 release surfaces, ATLAS/OWASP ML Top 10 security section). Keeping the two documents side by side in the repo prevents the charter from drifting away from the practice reality.
5. **Re-run the comparison on the same cadence as the charter** — quarterly, or after any Sev-1 that revealed a missing row. Rows migrate between delta categories over time (e.g., "expanded" practices become "transfers with modification" as tooling matures); the document is only useful if it reflects the current state.

## 3. Validation steps

1. **Delta-category check.** For every row in the learner's table, is the category (`unchanged` / `modified` / `expanded` / `new`) justified by a named mechanism?
2. **Over-claim check.** No row should claim "new" for a practice that is actually classical SRE applied to a new substrate (e.g., GPU capacity planning is *modified* capacity planning, not a new practice).
3. **Under-claim check.** No row should claim "unchanged" for a practice whose SLI, mitigation, or release surface demonstrably differs (e.g., incident response is modified, not unchanged, because model-rollback and guardrail-hotpatch are new mitigations).
4. **Governance hook.** Does at least one row link to NIST AI RMF?
5. **Security hook.** Does at least one row link to MITRE ATLAS or OWASP ML Top 10?
6. **Dry-run.** Pick two historical incidents (one classical, one AI-specific); walk them through the learner's table and confirm the practices invoked match the delta categories.

## 4. Rubric / review checklist

- [ ] **≥ 10 practice areas covered.** (2 pts)
- [ ] **Each row names the mechanism of change**, not just that it's different. (4 pts)
- [ ] **Delta categories used consistently.** (2 pts)
- [ ] **At least 3 rows in "unchanged" / "modified" / "expanded or new."** No one-sided tables. (2 pts)
- [ ] **Capacity planning row is accelerator-aware** (GPU/TPU, KV-cache, batch-size, provider rate-limit). (2 pts)
- [ ] **Security row references MITRE ATLAS or OWASP ML Top 10.** (2 pts)
- [ ] **Governance row references NIST AI RMF function(s).** (2 pts)
- [ ] **Quality SLO is distinct from serving SLO.** (2 pts)
- [ ] **Model providers appear as a dependency class.** (2 pts)
- [ ] **No row is marketing-copy symmetry** ("classical: X, AI: X + AI"). (reviewer judgment, up to −4 pts)

Pass threshold: 15/20.

## 5. Common mistakes

- **Marketing-copy symmetry.** "Classical: monitoring. AI: AI monitoring." The row carries no information.
- **Collapsing serving and quality into one SLO.** This hides the most common AI incident mode: serving green, quality red.
- **Treating model providers as just another vendor.** Model providers ship silent behavior changes; they belong in a dedicated row.
- **Over-classifying as "new."** Most AI SRE practice is modified classical SRE; genuinely new practices are a short list (evals-as-gate, drift ops, adversarial-ML response, governance evidence).
- **Missing the data layer.** Data pipelines are a reliability surface in AI; omitting them from the table understates the on-call burden.
- **No cost column.** In AI, cost is a reliability constraint, not just a finance concern.
- **No governance row.** Leaves the reliability function with no answer to "how do we feed what we see to the risk register?"

## 6. References

- **Google SRE book** — canonical source for the classical practices this exercise is comparing against. (General reference; see `https://sre.google/books/` for the book family.)
- **NIST AI RMF** — the governance vocabulary (`GOVERN`, `MAP`, `MEASURE`, `MANAGE`). <https://www.nist.gov/itl/ai-risk-management-framework>
- **MITRE ATLAS** — adversarial-ML tactic/technique catalog. <https://atlas.mitre.org/>
- **OWASP Machine Learning Security Top 10** — threat categories. <https://owasp.org/www-project-machine-learning-security-top-10/>
- Local exercise context: `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-02-classical-sre-vs-ai-sre-side-by-side.md`.
