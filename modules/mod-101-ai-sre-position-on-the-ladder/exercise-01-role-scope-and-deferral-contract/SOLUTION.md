# Exercise 01 — Role Scope and Deferral Contract — Reference Solution

> Companion to `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-01-role-scope-and-deferral-contract.md` in the paired learning repo.

## 1. Solution overview

This is a **design exercise**. The learner is asked to define, on paper, the boundary between the AI SRE role and adjacent roles (ML Engineering, Security, Data Engineering, Platform/Infra, Product), and to express that boundary as a **deferral contract**: an explicit, written rule for what the AI SRE *will not decide alone* and whom they will pull in.

A strong submission has three parts:

1. **An in-scope list** of concerns the AI SRE owns end-to-end.
2. **A deferral list** of concerns where the AI SRE is a *contributor or gate-holder*, not the decider, with the named decider role for each item.
3. **Trigger and handoff rules** — observable conditions that move an issue from "AI SRE handles" to "AI SRE escalates/defers," plus what the handoff artifact looks like.

The point is not to produce a maximalist list. It is to force explicit choices: concerns the learner lists as "in scope" become things they are on the hook for; concerns they defer must name a decider. Vague ownership is the failure mode.

## 2. Worked answer

### 2.1 In-scope for the AI SRE

The AI SRE owns the **reliability of a model-backed service in production**. Concretely:

| Concern | What AI SRE owns |
|---|---|
| Service SLIs / SLOs | Definition, instrumentation, error budget accounting for the serving path (latency, availability, correctness proxies). |
| Serving-stack health | Inference server (vLLM / SGLang / TRT-LLM / Triton / custom), GPU utilization, batching, queueing, autoscaling behavior. |
| Incident response | Primary on-call for the model-backed service: detection, mitigation, comms, follow-up. |
| Release safety | Canary, progressive rollout, rollback of model versions and serving-config changes; gating on reliability signals. |
| Capacity planning | GPU/accelerator demand forecasting, headroom for traffic + reload spikes, cost/perf tradeoffs. |
| Observability surface | Logs/metrics/traces for the request path, including token-level and prompt/response metadata where retention rules allow. |
| Vendor/provider reliability | Monitoring third-party model APIs as an *upstream dependency* (latency, error rate, rate limits, deprecation). |
| Runbooks and post-incident learning | Writing, drilling, and closing the loop on findings. |

### 2.2 Deferral list (contributor, not decider)

| Concern | AI SRE role | Deciding role |
|---|---|---|
| Model choice, architecture, fine-tuning, eval design | Provides reliability constraints (latency budget, cost ceiling, failure modes seen in prod). | ML / Applied Science. |
| Training-pipeline failures and data-quality incidents | Alerts and triages until scoped; defers to Data/ML for root cause of model-level regressions. | Data Engineering / ML. |
| Model-safety and content policy decisions (what outputs are acceptable) | Monitors and routes incidents; does not set policy. | Trust & Safety / Policy / Legal. |
| Adversarial ML risk acceptance (prompt injection, model extraction, evasion) | Operates the detections and mitigations once chosen; reports exposure. | Security / AI Red Team, informed by MITRE ATLAS tactic/technique mapping. |
| Regulatory posture and documentation (e.g., AI RMF mappings, model cards, risk tiering) | Contributes operational evidence (incidents, SLO history, drift events). | Governance / Compliance / Legal, framed by NIST AI RMF. |
| Compute purchase commitments and multi-year capacity contracts | Provides forecast inputs. | Finance / Infra leadership. |
| Underlying platform (K8s, networking, storage) beyond the serving path | Consumes as a dependency; raises issues. | Platform / Infra SRE. |
| Product scope and launch gating on non-reliability grounds | Flags reliability risk; does not block for other reasons. | Product. |

### 2.3 Trigger and handoff rules

A deferral contract is only useful if the triggers are observable. Example triggers:

- **Quality regression trigger.** If a correctness proxy SLI (e.g., eval pass-rate on a holdout, user thumbs-down rate, refusal rate) breaches its SLO for two consecutive windows **and** serving-path metrics are green, the AI SRE opens an incident **but routes root cause to ML**. Handoff artifact: incident doc with timeline, affected cohort, and the relevant eval slice.
- **Safety trigger.** If output filtering catches a step-change in policy violations, AI SRE pages the on-call Trust & Safety rep, continues to run the mitigation (e.g., stricter guardrail, route to smaller model, serve from cache), and defers *policy change* to T&S. Handoff artifact: sanitized sample set + detection rule diff.
- **Adversarial trigger.** A detection that maps to a MITRE ATLAS technique (e.g., `AML.T0051 LLM Prompt Injection`, `AML.T0040 ML Model Inference API Access`) escalates to Security within the agreed MTTA for that technique class. AI SRE preserves logs and request traces; Security leads containment.
- **Governance trigger.** Any incident that would require updating a regulated model card, a risk register entry, or an AI RMF `GOVERN`/`MAP`/`MEASURE`/`MANAGE` artifact is tagged as such in the postmortem, and Governance owns the downstream update.
- **Capacity trigger.** Forecast error on GPU demand exceeds ±X% for a quarter, or a vendor announces a capacity ceiling. AI SRE presents the number; Finance/Infra leadership owns the commit decision.

### 2.4 Deferral contract template (fill-in form)

```
Deferral Contract — <service name>
Owner: AI SRE team <name>
Version: <date>

In scope (we decide):
  - <concern 1>
  - <concern 2>
  ...

Deferred (we contribute, they decide):
  - <concern>  ->  <role>  (trigger: <observable condition>; handoff: <artifact>)
  ...

Review cadence: quarterly, or after any Sev-1/Sev-2 incident that invoked a deferral path.
```

## Implementation

To turn the worked answer above into a live deferral contract:

1. **Enumerate the concerns first, then classify.** Before writing the contract, list every reliability-adjacent concern the service team touches in a quarter (pull from the last ~8 incidents, the last 2 release reviews, and the current risk register). Only then place each concern in the in-scope or deferred list — this prevents the "everything is ours" default.
2. **Fill the template in §2.4 with concrete role names.** For each deferred item, open the org chart and name the role (not person) that currently holds the decision. Where the role does not exist yet, write `TBD — needs owner` and open a tracking issue; an unowned deferral is a known gap, not an acceptable state.
3. **Wire the triggers into existing signals.** Each observable trigger in §2.3 must map to an alert, a dashboard panel, or a ticket template. If the trigger has no mechanical surface (e.g., "ATLAS technique detection") the contract is aspirational — file the detection work before publishing.
4. **Commit the document into the service repo.** Store `deferral-contract.md` alongside the service's runbooks, link it from the on-call handbook, and add a `CODEOWNERS` entry so changes require sign-off from the AI SRE lead and each named deciding role.
5. **Review on the cadence declared at the bottom of the template** — quarterly, plus after any Sev-1/Sev-2 that invoked a deferral path. The review's job is to confirm triggers still fire, deciders still exist, and no new concern has drifted into "nobody owns it."

## 3. Validation steps

1. **Ownership closure check.** For every concern the service team treats as "reliability-adjacent" (list them before writing the contract), the contract places it in exactly one of the two lists. No concern is in both, none is missing.
2. **Named decider check.** Every deferred item names a role, not "the team" or "whoever is around."
3. **Observable trigger check.** Every deferral rule names an *observable* condition (SLI breach, alert firing, ATLAS technique hit, doc-change required), not a feeling.
4. **Handoff artifact check.** Every deferral names what the AI SRE hands over (incident doc, trace bundle, eval slice, sanitized samples).
5. **Dry-run.** Walk the last two incidents (real or hypothetical from the module exercises) through the contract. Does it route correctly? If not, the contract is wrong, not the incident.

## 4. Rubric / review checklist

Score each submission against:

- [ ] **Scope list is finite and specific.** Not "all reliability," but enumerated concerns. (2 pts)
- [ ] **Each deferred item names a role, not a person.** (2 pts)
- [ ] **Every deferral has an observable trigger.** (3 pts)
- [ ] **Every deferral names a handoff artifact.** (2 pts)
- [ ] **Adversarial-ML deferral is mapped to MITRE ATLAS** (at least one technique ID referenced). (2 pts)
- [ ] **Governance deferral is mapped to NIST AI RMF function** (GOVERN / MAP / MEASURE / MANAGE). (2 pts)
- [ ] **Safety/content-policy deferral is explicit and separate from reliability**, not folded into generic "quality." (2 pts)
- [ ] **Review cadence and version/owner metadata present.** (1 pt)
- [ ] **Dry-run against two incidents routes correctly.** (4 pts)

Pass threshold: 15/20. Below that, the contract is scaffolding-only and the learner should revise before moving on.

## 5. Common mistakes

- **Owning safety policy.** Writing "AI SRE decides what outputs are acceptable." Content policy is a Trust & Safety / Legal decision; AI SRE operates the detections and runs the incident, but does not set the rule.
- **Deferring with no trigger.** "We'd loop in Security if something seemed suspicious" is not a contract — it is wishful on-call behavior.
- **Owning model quality end-to-end.** Treating eval regressions as AI SRE's root-cause responsibility collapses the ML/SRE boundary and under-invests in model-side investigation.
- **Role-as-person.** Naming an individual rather than a role causes the contract to break on PTO, reorgs, and turnover.
- **One giant "other" bucket.** A single catch-all deferral line (e.g., "everything not above goes to Platform") is equivalent to not having a contract.
- **No review cadence.** Contracts drift. Without a review date and an incident-triggered review, the document ages out within a quarter.
- **No governance hook.** Omitting any link to NIST AI RMF or equivalent risk-management framework leaves regulated or high-risk deployments without an upstream owner.

## 6. References

- **NIST AI RMF** — the functions `GOVERN`, `MAP`, `MEASURE`, `MANAGE` are the vocabulary for the governance deferral path. <https://www.nist.gov/itl/ai-risk-management-framework>
- **MITRE ATLAS** — tactic/technique IDs are the vocabulary for the adversarial-ML deferral path. <https://atlas.mitre.org/>
- **OWASP Machine Learning Security Top 10** — threat classes informing what belongs on the security-deferral list. <https://owasp.org/www-project-machine-learning-security-top-10/>
- Local exercise context: `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-01-role-scope-and-deferral-contract.md` (paired learning repo).
