# Exercise 03 — Reliability Charter Authoring for an AI Team — Reference Solution

> Companion to `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-03-reliability-charter-authoring-for-an-ai-team.md` in the paired learning repo.

## 1. Solution overview

This is a **design exercise with a document deliverable**. The learner writes a **reliability charter** for an AI team — the one-page agreement that says *what we commit to on reliability, how we measure it, and what we will do when we breach it.* The charter is the document that future on-call rotations, release reviews, and incident reviews are tested against.

A charter is not an SLO document and not a runbook. It is the **governing agreement** that makes SLOs and runbooks legitimate: it says who signed up for what, under which conditions, and who can change it.

A strong charter has eight parts:

1. **Scope** — which service(s), which users, which failure classes.
2. **Reliability objectives** — the SLIs and SLOs, including separate **serving** and **quality** objectives.
3. **Error budget policy** — what happens when the budget is spent or burning fast.
4. **Roles and deferrals** — who owns what, who decides when AI SRE defers (see `[[exercise-01-role-scope-and-deferral-contract]]` in the paired repo).
5. **Risk register hooks** — how reliability events feed governance (NIST AI RMF).
6. **Security posture commitments** — the adversarial-ML and classical threats the team commits to detect/mitigate (MITRE ATLAS / OWASP ML Top 10).
7. **Change control** — how model, prompt, retrieval, and config changes are released and rolled back.
8. **Review cadence and amendment rule** — how the charter is kept alive.

The deliverable is one document, under ~1,500 words, signed by named role-holders.

## 2. Worked answer

Below is a **template** charter, worked through with a concrete fictional-but-representative service (`assist-api`, an internal RAG-backed assistant exposed over HTTP to employees).

### 2.1 Charter — `assist-api` Reliability Charter (template)

```
Reliability Charter — assist-api
Version: 1.0
Effective: <YYYY-MM-DD>
Owners: AI SRE lead <role>, ML lead <role>, Product lead <role>, Security lead <role>, Governance lead <role>
Review cadence: quarterly, or after any Sev-1 incident, or on material scope change.

1. Scope
  - In scope: the assist-api serving path (HTTP endpoint, inference server, retrieval
    service, guardrail layer), its model deployments, and its prompt/template surface.
  - Users: all employees on the corporate network.
  - Out of scope: downstream consumers' own UIs, user-authored prompts, and
    offline evaluation infrastructure (owned by ML).

2. Reliability objectives
  - Serving SLOs (28-day rolling window):
    * Availability: >= 99.5% (request returns a non-5xx within 30s).
    * TTFT p95:    <= 1.5s.
    * TTFT p99:    <= 4.0s.
    * End-to-end p95 latency: <= 8.0s for the standard-answer path.
  - Quality SLOs (weekly window):
    * Offline eval pass rate on the regression set: >= <target> for the primary
      regression slice.
    * User-feedback negative rate: <= <target>% of rated responses.
    * Guardrail-caught policy-violation rate: monitored, with alert on step change
      rather than a fixed budget.
  - Correctness-proxy SLIs that are tracked but NOT bound to a budget this cycle
    are listed in Appendix A and marked "observational."

3. Error budget policy
  - Serving budget burn:
    * >= 2x burn rate over 1h and >= 5% of the monthly budget consumed: page.
    * >= 50% of the monthly budget consumed: feature-flag freeze on non-reliability
      changes; ML and product co-approve exceptions.
    * >= 100% consumed: hard freeze on model, prompt, retrieval-index, and
      serving-config changes except rollbacks and reliability fixes, until the
      window resets or the error-budget committee approves a time-boxed exception.
  - Quality budget burn:
    * Any sustained quality SLO breach (two consecutive windows) triggers an
      ML-led root-cause investigation, with AI SRE providing serving-side evidence.
    * ML owns "fix it" for quality; AI SRE owns "contain user impact" (route to
      fallback model, degrade features, or pause risky prompts).

4. Roles and deferrals
  - AI SRE owns: serving-path reliability, incident command, release safety,
    capacity planning, runbooks, and the operational evidence pipeline.
  - ML owns: model selection, fine-tuning, eval design, quality root cause.
  - Security owns: adversarial-ML policy, threat model, and detection rules.
  - Trust & Safety / Policy owns: content-acceptability policy.
  - Governance owns: risk register, model card, NIST AI RMF artifacts.
  - Full deferral table lives in the deferral contract (companion doc).

5. Risk register hooks
  - All Sev-1 and Sev-2 incidents produce an entry or update in the risk register.
  - Any incident that touches a NIST AI RMF function (GOVERN / MAP / MEASURE /
    MANAGE) is tagged with the function and routed to Governance for artifact
    update (model card, impact assessment, risk-tier change).

6. Security posture commitments
  - We detect and respond to adversarial-ML techniques from MITRE ATLAS in scope
    for an HTTP-exposed RAG assistant (non-exhaustive, see Appendix B):
      * Prompt-injection via tool-callable retrieval sources.
      * Model-inference-API abuse (rate limits, extraction patterns).
      * Membership-inference via repeated crafted queries.
  - We align detection categories to the OWASP ML Top 10 and review alignment
    on charter review.
  - Classical AppSec posture (authn/z, secrets, patching) is inherited from
    platform SRE's charter and referenced, not re-committed here.

7. Change control
  - Model-weight changes: offline eval gate + 5% canary + automatic rollback on
    quality or serving SLI regression.
  - Prompt/template changes: versioned, reviewed by ML and AI SRE, canary same
    as above.
  - Retrieval-index refreshes: shadow-compare against prior index on a held-out
    query set before promotion.
  - Config/knob changes (temperature, batch size, concurrency): change-ticketed,
    reversible within on-call shift.

8. Review cadence and amendment rule
  - Quarterly review; mandatory re-review after any Sev-1 incident.
  - Amendments require sign-off from all named owners (section header).
  - A material scope change (new user class, new modality, new region) requires
    a new charter version, not a patch.

Appendix A: Observational SLIs (not budgeted this cycle)
Appendix B: In-scope ATLAS techniques and detection owners
Appendix C: Glossary and SLI definitions
```

### 2.2 Why each section is in the charter

- **Scope** keeps on-call from inheriting problems the team never agreed to own.
- **Separate serving and quality SLOs** make it impossible to hide a quality regression behind a green serving dashboard.
- **Budget policy** is what turns reliability from a dashboard into a decision rule; without it, SLOs are vanity numbers.
- **Roles and deferrals** keep handoffs explicit; the deferral contract from exercise 01 is the referenced document.
- **Risk register hook** is the operational link into NIST AI RMF `MEASURE` and `MANAGE`.
- **Security posture** names the adversarial-ML threats the team commits to, grounded in MITRE ATLAS and OWASP ML Top 10 so categories are shared vocabulary, not local jargon.
- **Change control** names every release surface (not just code) and its rollback story.
- **Review cadence** is what keeps the charter from going stale.

### 2.3 Sign-off block (required)

The charter is not in force until each named role-holder has signed. The sign-off block is the single most under-filled part of a charter and the single strongest indicator that the document is more than theater:

```
Signed:
  AI SRE lead     <name>, <date>
  ML lead         <name>, <date>
  Product lead    <name>, <date>
  Security lead   <name>, <date>
  Governance lead <name>, <date>
```

## Implementation

To take the charter template from §2 to a signed, in-force document:

1. **Replace every `<placeholder>` in §2.1 with a concrete value.** SLO targets (availability, TTFT p95/p99, end-to-end p95, eval pass rate, feedback negative rate) must be numbers derived from current measurements plus a stretch margin, not aspirational round numbers. If a signal is not yet measured, add it to Appendix A ("observational") rather than binding a budget to a number nobody can defend.
2. **Resolve each named role to a current holder.** The sign-off block in §2.3 is the gating artifact. Name AI SRE lead, ML lead, Product lead, Security lead, and Governance lead by role *and* current occupant; where a role does not exist, the charter is a draft until the role is filled or explicitly consolidated with another.
3. **Wire the burn-rate policy to real alerting.** The three serving-budget thresholds in §3 of the template (2× burn @ 1h, 50% consumed, 100% consumed) each need an alert rule, a paging target, and a documented approver for exceptions. A charter whose thresholds are not firing anywhere is theater.
4. **Point sections 4–6 at companion artifacts in the repo.** The deferral contract from exercise 01 is referenced from §4; the ATLAS technique list in Appendix B must point at the detection owners; the risk-register hook in §5 must name the ticket type or doc the on-call fills out when tagging a NIST AI RMF function.
5. **Publish, sign, and schedule the review.** The charter is in force only after every named owner has signed with a date. File the next review on the owners' calendars (quarterly) and add an auto-created review ticket on Sev-1 close. Unreviewed charters older than one quarter should be treated as expired, not authoritative.

## 3. Validation steps

1. **Scope test.** Point to a feature, a user class, and a failure class; can you say "in scope" or "out of scope" from the charter text alone? If not, scope is underspecified.
2. **SLO separation test.** Can you have a green serving dashboard and a red quality dashboard — and does the charter tell you what happens? If the charter only names one SLO class, it fails this test.
3. **Budget-action test.** For "50% burn" and "100% burn," is there a named action and a named approver? If not, the budget policy is decoration.
4. **Governance-hook test.** For a plausible Sev-1 (e.g., retrieval index pulled poisoned data, guardrail bypass succeeded), does the charter say how it feeds the risk register / model card / RMF artifact?
5. **Change-surface test.** List all release surfaces (image, model weights, prompt, retrieval index, config). Is each addressed? If any is missing, add a row.
6. **Signatures test.** Every named owner has signed. Unsigned charters are drafts.

## 4. Rubric / review checklist

- [ ] **Scope is enumerated**, not "everything AI." (2 pts)
- [ ] **At least one serving SLO AND at least one quality SLO**, both with window, target, and metric. (3 pts)
- [ ] **Error budget policy with at least three burn thresholds**, each with a named action and a named approver. (3 pts)
- [ ] **Roles section references a separate deferral contract** or includes one. (2 pts)
- [ ] **Risk-register hook is explicit**, including NIST AI RMF function tagging. (2 pts)
- [ ] **Security section references MITRE ATLAS and OWASP ML Top 10.** (2 pts)
- [ ] **Change-control section names ≥ 4 release surfaces** (image/model/prompt/retrieval/config). (2 pts)
- [ ] **Review cadence and amendment rule present.** (2 pts)
- [ ] **Sign-off block present with ≥ 4 named roles.** (2 pts)

Pass threshold: 16/20.

## 5. Common mistakes

- **Treating the charter as an SLO document.** SLOs live inside a charter; a charter also contains scope, roles, budget policy, change control, and governance hooks.
- **One blended SLO for "quality".** Fuse serving availability and output quality into a single number and you get a budget that gets spent for the wrong reasons.
- **Budget policy with no approver.** "We will freeze releases" with no approver means nothing freezes.
- **Role names, not owners.** A charter signed by "the SRE team" does not commit anyone. Name roles with a current holder.
- **Omitting model-provider risk.** If the service depends on hosted models, the charter must state the fallback and the detection rule for provider degradation.
- **Omitting governance.** A charter with no link to NIST AI RMF (or an equivalent risk framework) is not deployable for any regulated or high-risk use case.
- **No amendment rule.** Charters that can be edited silently by anyone are worthless; charters that require re-signing on amendment are load-bearing.

## 6. References

- **NIST AI RMF** — functions `GOVERN`, `MAP`, `MEASURE`, `MANAGE`; used for governance hooks. <https://www.nist.gov/itl/ai-risk-management-framework>
- **MITRE ATLAS** — adversarial-ML technique catalog used in the security section. <https://atlas.mitre.org/>
- **OWASP Machine Learning Security Top 10** — threat categorization used in the security section. <https://owasp.org/www-project-machine-learning-security-top-10/>
- Companion local artifacts: deferral contract from `[[exercise-01-role-scope-and-deferral-contract]]`, SRE vs AI SRE delta table from `[[exercise-02-classical-sre-vs-ai-sre-side-by-side]]`.
- Local exercise context: `lessons/mod-101-ai-sre-position-on-the-ladder/exercises/exercise-03-reliability-charter-authoring-for-an-ai-team.md`.
