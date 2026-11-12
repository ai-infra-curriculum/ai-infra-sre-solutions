# Exercise 04 — Dickerson's Hierarchy Applied to AI Workloads — Reference Solution

> Companion to `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-04-dickerson-hierarchy-applied-to-ai-workloads.md` in the paired learning repo.

## 1. Solution overview

Mikey Dickerson's **Hierarchy of Reliability** (popularized through Google's SRE book as "Dickerson's Hierarchy of Service Reliability Needs") lists reliability investments in order of leverage, bottom-up: **Monitoring → Incident Response → Postmortem/Root-cause Analysis → Testing & Release → Capacity Planning → Development → Product**. Each layer assumes the layer below is solid; investing higher in the stack while the base is weak wastes effort.

This exercise asks the learner to **apply each layer to an AI/model-backed workload** and to name, per layer:

1. What the classical practice is.
2. What AI workloads add or change (and *why* — the mechanism).
3. One concrete artifact or signal the learner would produce at that layer this quarter.

The failure mode is to restate the classical layer with "...for AI" appended. A strong submission names the AI-specific mechanism of change and produces an artifact.

## 2. Worked answer — layer by layer

### 2.1 Layer 1 — Monitoring (base of the pyramid)

- **Classical.** Logs, metrics, traces for the request path; coverage of all critical SLIs; actionable alerts tied to user impact.
- **AI delta.** Three additional signal classes:
  1. **Model/output telemetry** — sampled prompts/responses (with retention and PII rules), eval pass rate on a continuous regression slice, user-feedback rate, guardrail-catch rate.
  2. **Data/drift telemetry** — input-distribution monitors, retrieval-corpus freshness and hit-rate, embedding-space drift against a reference set.
  3. **Accelerator telemetry** — GPU/TPU utilization, memory, SM/tensor-core efficiency, KV-cache occupancy, batch-queue depth, tokens/sec, TTFT distributions, cross-GPU link health.
- **Mechanism of change.** AI failures live in model behavior, data, and accelerator state — none of which are visible from HTTP-level metrics.
- **Quarter-one artifact.** A single dashboard with three rows (request-path SLIs, quality SLIs, accelerator SLIs) and a one-page alert catalog where every alert names a runbook.

### 2.2 Layer 2 — Incident response

- **Classical.** Clear on-call roster, paging, incident command, mitigate-first discipline, comms.
- **AI delta.** The mitigation toolbox grows substantially:
  - Model rollback to the previous weight / tag.
  - Route-around to a smaller or alternative provider's model (dual-source).
  - Prompt / guardrail hotpatch with its own small change-control.
  - Sampling knob changes (temperature, top-p) as a short-term degrade.
  - Serve-from-cache for a degraded quality mode.
- **Mechanism of change.** "Reliability" has a quality axis; mitigation can trade quality for availability, and on-call needs explicit permission to do so.
- **Quarter-one artifact.** A mitigation menu per Sev level, listing the five mitigations above, each with a named owner and an authorized-without-approval line.

### 2.3 Layer 3 — Postmortem / RCA

- **Classical.** Blameless, written, timeline + root cause + action items, tracked to close.
- **AI delta.** Root cause is split across three columns: **code/infra RCA**, **data RCA**, **model RCA**. For every Sev-1 and many Sev-2s, all three must be checked, not just the first that looks plausible.
- **Mechanism of change.** A model regression can look exactly like a serving regression on the user side; without a three-column discipline, the wrong root cause gets filed.
- **Quarter-one artifact.** A postmortem template with three required "root cause" sub-sections (code/infra, data, model), each with a "not applicable" option that must be justified in writing.

### 2.4 Layer 4 — Testing and release

- **Classical.** Unit/integration/load tests; canary and progressive rollout; feature flags and kill switches.
- **AI delta.**
  - **Offline eval as a release gate.** Each release candidate (model weight, prompt template, retrieval index) passes a versioned eval suite before promotion.
  - **Shadow traffic.** New model/prompt runs against a mirror of live traffic; outputs are compared offline before promotion.
  - **Automatic rollback on quality SLI regression**, not just error-rate regression.
  - **Multiple release surfaces** — image, weights, prompts, retrieval index, config — each with its own rollback.
- **Mechanism of change.** Non-determinism means classical diff-testing cannot certify output correctness; evals and shadowing fill that gap.
- **Quarter-one artifact.** An eval-gate pipeline: a versioned dataset, a pass-rate threshold agreed with ML, and a CI step that blocks promotion on regression.

### 2.5 Layer 5 — Capacity planning

- **Classical.** Forecast demand, plan supply (CPU/RAM/disk/network), measure headroom.
- **AI delta.**
  - Unit of capacity is **GPU-seconds subject to KV-cache memory constraints**, not CPU-seconds.
  - Batch-size and concurrency choices trade throughput for TTFT/p99 latency.
  - **Model-provider rate limits** are a capacity constraint for any hosted-model dependency.
  - Warm-pool capacity for model reload is its own line item.
  - MIG / partitioning choices are a capacity decision, not a config decision.
- **Mechanism of change.** Accelerator economics, KV-cache memory, and provider quota are new constraint classes without classical equivalents.
- **Quarter-one artifact.** A capacity model that outputs required GPU-seconds (and provider QPM/TPM) per 10% increment of forecast traffic, reviewed quarterly.

### 2.6 Layer 6 — Development

- **Classical.** Reliable services depend on well-written, reviewable, testable code; SRE contributes patterns and sometimes code.
- **AI delta.** The "development" surface expands to include:
  - **Prompt authoring and review** as a code-review-equivalent discipline.
  - **Retrieval-corpus curation** as a development activity with its own CI (freshness, duplication, PII checks).
  - **Tool/function definitions** for tool-using models become a public API surface with versioning.
- **Mechanism of change.** The artifacts that shape behavior extend beyond source code; they need the same review and versioning hygiene.
- **Quarter-one artifact.** A "prompts and tool defs" repo with code-review ownership, CI, and semantic versioning.

### 2.7 Layer 7 — Product (apex)

- **Classical.** Reliability at the top layer is about product decisions: features launched, user promises made, failure modes acceptable to users.
- **AI delta.**
  - Product must decide which **failure modes are acceptable and visible**: wrong answers, refusals, latency spikes, hallucinated citations, disclaimers.
  - Product owns the **UX of degraded modes** (e.g., "we are temporarily using a smaller model" messaging).
  - Product makes risk-tier decisions that feed governance (NIST AI RMF `MAP`), which in turn constrain the layers below.
- **Mechanism of change.** User-visible failure modes are new (hallucination, refusal, inconsistency); product must pick which to tolerate and which to spend budget on.
- **Quarter-one artifact.** A "user-visible failure modes" table agreed with product, with each mode tagged as *suppress*, *disclose*, or *degrade-gracefully*, and linked to the corresponding SLI/SLO.

### 2.8 Layering check — don't invest above a weak layer

Dickerson's point is that layers are ordered by leverage. The AI-specific implication:

- Investing in evals and shadow traffic (layer 4) while the quality SLI is not even monitored (layer 1) is wasted work — you cannot detect the regressions your pipeline is designed to catch.
- Standing up a chaos program for model swaps (layer 4+) while incident response (layer 2) has no mitigation menu for model rollback is wasted work — you can discover failures you cannot act on.
- Writing a reliability charter (apex, layer 7-adjacent) while monitoring is absent gives you a governance surface over a service you cannot see.

The rule is the same as classical SRE: **fix the lowest weak layer first**.

## Implementation

To sequence the per-layer work from §2 into a quarter of execution rather than a wish list:

1. **Audit the base layer first.** Before scheduling any work above layer 1, score the current monitoring stack against the three signal classes in §2.1 (model/output, data/drift, accelerator). Any class with less than one actionable alert is a gap that blocks investment above it. This is the operationalization of the "fix lowest weak layer first" rule from §2.8.
2. **Pick exactly one artifact per layer this quarter.** §2.1–§2.7 each propose a concrete quarter-one artifact (dashboard + alert catalog, mitigation menu, three-column postmortem template, eval-gate pipeline, GPU-seconds capacity model, prompts/tools repo, user-visible failure-modes table). Resist adding a second — layered investment outperforms horizontal sprawl.
3. **Order the seven artifacts by Dickerson's pyramid.** Do not start the eval-gate pipeline (layer 4) in the same sprint as the monitoring dashboard (layer 1); the eval gate consumes the quality SLI the dashboard produces. Sequence: monitoring → incident-response mitigation menu → postmortem template → eval gate → capacity model → prompts repo → failure-modes table.
4. **Bind each artifact to a named owner and a review signal.** The artifact is not complete until it has a `CODEOWNERS` entry, a dashboard panel or doc URL linked from the on-call handbook, and a review event (incident, release, quarterly) that forces it to be opened. Artifacts that no one opens are not reliability work.
5. **Re-audit the pyramid at quarter end.** Score each layer again against §4's rubric before planning the next quarter. If any lower layer has regressed, pause higher-layer investment and repair the base — this is the operational meaning of the hierarchy, not a metaphor.

## 3. Validation steps

1. **Per-layer mechanism check.** For every layer, does the submission name an AI-specific *mechanism* of change (non-determinism, data dependency, accelerator economics, provider risk, governance exposure)?
2. **Per-layer artifact check.** For every layer, is there one concrete artifact the learner commits to this quarter? "A dashboard," "a template," "a pipeline" — each with enough specificity to be built.
3. **Ordering check.** Does the submission respect the base-first ordering, or is it inverted (e.g., proposing chaos work while monitoring is absent)?
4. **No-symmetry check.** Reject layers where the AI delta reads as "...for AI" without a mechanism.
5. **Governance tie-in.** Does at least the product/apex layer connect to NIST AI RMF?
6. **Security tie-in.** Does the submission acknowledge that security monitoring (MITRE ATLAS, OWASP ML Top 10) is part of layer 1, not an orthogonal pillar?

## 4. Rubric / review checklist

- [ ] **All 7 layers addressed.** (2 pts)
- [ ] **Each layer names the AI-specific mechanism, not just the practice.** (5 pts, ~0.7/layer)
- [ ] **Each layer has a concrete, buildable artifact.** (5 pts, ~0.7/layer)
- [ ] **Monitoring layer separates serving, quality, and accelerator signal classes.** (2 pts)
- [ ] **Incident-response layer lists ≥ 3 AI-specific mitigations with authorization rules.** (2 pts)
- [ ] **Postmortem layer requires three-column RCA (code/infra, data, model).** (2 pts)
- [ ] **Testing/release layer names ≥ 3 release surfaces with their own rollbacks.** (2 pts)
- [ ] **Capacity layer uses GPU-seconds and KV-cache memory as units, and names provider rate limits.** (2 pts)
- [ ] **Product/apex layer connects to NIST AI RMF or equivalent risk-tier decision.** (2 pts)
- [ ] **Submission explicitly states the "fix lowest weak layer first" principle** or applies it in sequencing recommendations. (2 pts)

Pass threshold: 18/26.

## 5. Common mistakes

- **Symmetry restatements.** "Monitoring, for AI." No mechanism, no artifact.
- **Investing above a weak layer.** Proposing chaos drills while monitoring is incomplete.
- **Collapsing "testing" into one bucket.** AI testing is at least three distinct surfaces (offline eval, shadow traffic, production canary with automatic rollback).
- **Capacity planning in CPU units.** Reveals that the learner has not internalized that accelerator-second is the unit.
- **Omitting the data layer.** Most learners discuss code and model; data-RCA and retrieval-index freshness are commonly missing.
- **Treating product as out of scope.** The apex layer is where failure-mode tolerance is decided; SRE needs it to size budgets.
- **Missing the governance hook.** NIST AI RMF `MAP` tier decisions bind the layers below; without the link, reliability work floats without a risk framing.
- **Treating security as a sidebar.** Adversarial-ML monitoring belongs *in* layer 1; a separate "security pillar" misses that it is reliability signal.

## 6. References

- **Mikey Dickerson's Hierarchy of Service Reliability Needs**, as described in the Google SRE book, chapter "Embracing Risk" and surrounding chapters. (General reference; see `https://sre.google/books/` for the book family.)
- **NIST AI RMF** — the `MAP` function is the risk-tier decision the apex layer connects to. <https://www.nist.gov/itl/ai-risk-management-framework>
- **MITRE ATLAS** — adversarial-ML techniques, consumed by the monitoring and incident-response layers. <https://atlas.mitre.org/>
- **OWASP Machine Learning Security Top 10** — threat categories feeding the monitoring layer's security signals. <https://owasp.org/www-project-machine-learning-security-top-10/>
- Companion local artifacts: deferral contract (`[[exercise-01-role-scope-and-deferral-contract]]`), SRE vs AI SRE delta table (`[[exercise-02-classical-sre-vs-ai-sre-side-by-side]]`), reliability charter (`[[exercise-03-reliability-charter-authoring-for-an-ai-team]]`).
- Local exercise context: `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-04-dickerson-hierarchy-applied-to-ai-workloads.md`.
