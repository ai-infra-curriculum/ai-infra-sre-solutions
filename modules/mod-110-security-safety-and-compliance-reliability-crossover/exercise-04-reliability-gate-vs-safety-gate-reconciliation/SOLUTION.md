# Exercise 04 — Reliability Gate / Safety Gate Reconciliation — Reference Solution

Paired learning file: `lessons/mod-110-security-safety-and-compliance-reliability-crossover/exercises/exercise-04-reliability-gate-vs-safety-gate-reconciliation.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred
> below is derived from this module's chapter on reliability-and-safety gate
> reconciliation at release time. If the authored exercise narrows or widens scope on a
> subsequent content cycle, this solution should be regenerated.

> **Scope boundary.** This is the *release-time* reconciliation between the reliability
> gate (SLO-shaped) and the safety gate (eval / policy / red-team-shaped). The gates'
> *content* — which SLOs, which eval sets, which policy classes — is set elsewhere.
> This exercise authors the *reconciliation machinery*: how the gates interact, how
> conflicts are resolved, how overrides are controlled, how the audit trail is kept.

## 1. Solution overview

The deliverable is a committed design document `docs/release/gate-reconciliation.md`
plus three machine-readable artifacts:

1. A gate-reconciliation policy file (`config/release/gate-policy.yaml`) that encodes
   the gate taxonomy, the conflict-resolution matrix, the override policy, and the
   audit requirements.
2. A release-promotion pipeline step (`pipelines/release-gate.yaml`) that is called by
   the progressive-delivery system (`mod-111`) and emits structured gate decisions.
3. An audit-log schema (`schemas/release-gate-decision.schema.json`) for the gate
   decisions.

The reference design treats both gates as **block-by-default** with a narrow,
auditable override path. Reliability regressions are reversible (rollback is the
default remedy; `mod-111` owns the rollback machinery). Safety regressions are often
*not* fully reversible — once users receive unsafe output, the harm is done — and
their gate is correspondingly more conservative. The reconciliation recognises this
asymmetry explicitly.

## 2. Implementation — worked answer

### 2.1 Gate taxonomy

The release pipeline has two independent gates plus the reconciliation step:

```
          ┌──────────────┐
          │ Build / Tag  │
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │  Reliability │  SLO-shaped: canary pass-rate, burn-rate, provider-
          │     gate     │  health, performance envelope.
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │    Safety    │  Eval-shaped: safety eval pass-rate, red-team fixture
          │     gate     │  pass-rate, policy-class coverage, adversarial-eval
          │              │  regression.
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │ Reconciliation│  Combines gate verdicts with the conflict-resolution
          │               │  matrix; emits the single release decision + audit.
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │   Promote    │
          └──────────────┘
```

**Important property.** Both gates run on every candidate, even if one has already
failed. The reconciliation step consumes both verdicts. A safety-pass + reliability-fail
is operationally distinct from a safety-fail + reliability-fail (one is "safe but will
regress users' experience"; the other is "unsafe and broken").

### 2.2 Verdict shape

Each gate emits a verdict with four components:

```json
{
  "gate": "reliability | safety",
  "verdict": "pass | fail | warn | inconclusive",
  "signals": [
    {"name": "<signal-id>", "observed": ..., "threshold": ..., "state": "pass|fail|warn"},
    ...
  ],
  "sample_size": <int or object>,
  "advisory_notes": "<optional>"
}
```

- `pass` — all required signals are within tolerance.
- `warn` — advisory signals are out of tolerance, but no required signal failed. Does
  not block; feeds the reconciliation audit.
- `fail` — at least one required signal failed.
- `inconclusive` — the gate could not evaluate (e.g., eval set did not complete due to
  infra). Treated as `fail` for blocking purposes; the override policy for
  inconclusive-due-to-infra is narrower than for fail-due-to-regression.

### 2.3 Conflict-resolution matrix

The matrix maps the (`reliability_verdict`, `safety_verdict`) pair to a release
decision and an override-eligibility class.

| Reliability | Safety | Decision | Override class |
|-------------|--------|----------|----------------|
| pass | pass | **promote** | n/a |
| pass | warn | promote with digest | override not required |
| warn | pass | promote with digest | override not required |
| warn | warn | **hold-for-review** | reliability + safety peers; promote after sign-off; no exec needed |
| fail | pass | **block** (reliability-only) | reliability override (see §2.4); safety peer informed |
| pass | fail | **block** (safety-only) | safety override (see §2.4); narrow, exec-required for non-dark-launch |
| warn | fail | block | same as (pass, fail) |
| fail | warn | block | same as (fail, pass) |
| fail | fail | **block** (both) | both overrides required; exec required |
| inconclusive (any) | inconclusive (any) | block | narrowest override class of the two sides |

**Default stance.** If the pair is not in the matrix (vocabulary drift), the release is
blocked. The audit record names the un-matched pair.

### 2.4 Override policy

Overrides are the escape valve. They are narrow, auditable, time-bounded, and
asymmetric between the gates.

#### 2.4.1 Reliability override

Reliability failures are reversible. The override is engineering-owned.

- **Who can approve:** the AI SRE on-call lead + the service's engineering lead. A
  single approver is **not** sufficient.
- **Time-bounded:** the override expires in 24h. If the fix is not landed by then, the
  override does not auto-renew; a new override is required.
- **Scope-bounded:** the override is scoped to a specific build ID; it does not cover a
  future build.
- **Automatic rollback:** if the SLO burn-rate alert fires post-promote on the overridden
  build, `mod-111`'s rollback runbook executes automatically.
- **Audit:** the override record contains approvers, build ID, signals that failed,
  mitigations committed, expiry.

#### 2.4.2 Safety override

Safety failures are often *not* reversible. The override is deliberately narrow.

- **Who can approve:** the safety peer on-call lead + the AI risk engineer + an
  executive sponsor. Three approvers — not two — to match the asymmetric risk.
- **Time-bounded:** 72h maximum. The override does not auto-renew.
- **Scope-bounded:** a specific build + a specific exposure profile (dark-launch,
  internal-only, partial rollout at <=1% traffic). A full-promote safety override is
  not available in the standing policy; if required, it is treated as an incident,
  not an override.
- **Containment required:** the override lands with a documented containment plan
  (traffic shape, safety-signal monitoring cadence, escalation trigger).
- **Audit + legal / compliance notification:** the override record is also sent to the
  `legal-oncall` and `ai-risk-engineer-oncall` rotations at the time of approval;
  retention is governed by exercise-03's compliance store policy.

#### 2.4.3 What is never overridden

Independent of the gate verdict pair, the following signals block absolutely:

- `safety:cbrn-regression` > 0 (any regression on CBRN eval set).
- `safety:csam-regression` > 0.
- `safety:self-harm-regression` > 0 on surfaces not carrying an explicit safety-services
  label.
- `compliance:residency-breach-candidate` == true (a candidate observed in canary that
  suggests a data-residency breach under production conditions).
- `security:control-coverage-zero` on a required control class.

The non-overridable list is reviewed at the quarterly release-process review and
signed by the AI SRE lead, the safety peer lead, security-engineering lead, and the
`ai-risk-engineer`.

### 2.5 Gate-policy file — `config/release/gate-policy.yaml`

```yaml
version: 2026-11-01
owner: ai-sre + safety-engineering + security-engineering

gates:
  reliability:
    owner: ai-sre
    required_signals:
      - id: canary_pass_rate
        threshold: ">= 0.995"
      - id: canary_slo_burn_rate_1h
        threshold: "< 1.0"
      - id: canary_slo_burn_rate_6h
        threshold: "< 1.0"
      - id: performance_envelope_p95_delta
        threshold: "<= 0.10"
      - id: provider_health_canary
        threshold: "== green"
    advisory_signals:
      - id: cache_hit_ratio_delta
        threshold: ">= -0.05"
      - id: provider_fallback_rate_delta
        threshold: "<= 0.05"

  safety:
    owner: safety-engineering
    required_signals:
      - id: safety_eval_pass_rate
        threshold: ">= 0.985"
      - id: redteam_fixture_pass_rate
        threshold: "== 1.0"
      - id: policy_class_coverage
        threshold: "== 1.0"
      - id: adversarial_eval_regression
        threshold: "<= 0"
    advisory_signals:
      - id: eval_pass_rate_delta
        threshold: ">= -0.005"

non_overridable:
  - id: safety_cbrn_regression_count
    threshold: "== 0"
  - id: safety_csam_regression_count
    threshold: "== 0"
  - id: safety_self_harm_regression_count_non_safety_route
    threshold: "== 0"
  - id: compliance_residency_breach_candidate
    threshold: "== false"
  - id: security_control_coverage_zero
    threshold: "== false"

override_policy:
  reliability:
    approvers_required: 2
    approver_roles: [ai-sre-oncall-lead, service-engineering-lead]
    max_duration_hours: 24
    scope: build-id
    automatic_rollback_on_alert: true
  safety:
    approvers_required: 3
    approver_roles: [safety-engineering-oncall-lead, ai-risk-engineer-oncall, exec-sponsor]
    max_duration_hours: 72
    scope: build-id-and-exposure-profile
    exposure_profiles_allowed: [dark-launch, internal-only, partial-rollout-1pct]
    containment_plan_required: true
    legal_notify: true

resolution_matrix:
  - reliability: pass
    safety: pass
    decision: promote
  - reliability: pass
    safety: warn
    decision: promote-with-digest
  - reliability: warn
    safety: pass
    decision: promote-with-digest
  - reliability: warn
    safety: warn
    decision: hold-for-review
    review_roles: [ai-sre-oncall-lead, safety-engineering-oncall-lead]
  - reliability: fail
    safety: pass
    decision: block
    override_class: reliability
  - reliability: pass
    safety: fail
    decision: block
    override_class: safety
  - reliability: warn
    safety: fail
    decision: block
    override_class: safety
  - reliability: fail
    safety: warn
    decision: block
    override_class: reliability
  - reliability: fail
    safety: fail
    decision: block
    override_class: both

audit:
  store: compliance-log-store  # same store as exercise-03
  retention: 36 months
  roles_read: [ai-sre, safety-engineering, security-engineering, ai-risk-engineer, legal]
```

### 2.6 Release-gate pipeline step — `pipelines/release-gate.yaml`

```yaml
name: release-gate
description: |
  Called by the progressive-delivery system (mod-111) on every candidate build.
  Runs the reliability gate and the safety gate independently, reconciles them per
  the gate-policy file, and emits a structured decision + audit record.

inputs:
  build_id: string
  candidate_canary_report: artifact
  candidate_eval_report: artifact
  candidate_redteam_report: artifact
  gate_policy_version: string  # pinned

steps:
  - id: reliability-verdict
    run: release-gate-eval reliability --policy=${gate_policy_version} --report=${candidate_canary_report}
    emits: reliability-verdict.json

  - id: safety-verdict
    run: release-gate-eval safety --policy=${gate_policy_version} --eval=${candidate_eval_report} --redteam=${candidate_redteam_report}
    emits: safety-verdict.json

  - id: non-overridable-check
    run: release-gate-eval non-overridable --policy=${gate_policy_version} --report=${candidate_canary_report} --eval=${candidate_eval_report}
    emits: non-overridable.json

  - id: reconcile
    run: release-gate-reconcile --policy=${gate_policy_version} \
         --reliability=reliability-verdict.json \
         --safety=safety-verdict.json \
         --non-overridable=non-overridable.json
    emits: release-decision.json
    audit:
      sink: compliance-log-store
      schema: schemas/release-gate-decision.schema.json

outputs:
  decision: release-decision.json  # consumed by mod-111
```

### 2.7 Audit schema — `schemas/release-gate-decision.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ReleaseGateDecision",
  "type": "object",
  "required": [
    "build_id",
    "gate_policy_version",
    "reliability_verdict",
    "safety_verdict",
    "non_overridable_result",
    "decision",
    "decided_at_utc",
    "pipeline_run_id"
  ],
  "properties": {
    "build_id": { "type": "string" },
    "gate_policy_version": { "type": "string" },
    "reliability_verdict": {
      "type": "object",
      "required": ["verdict", "signals"],
      "properties": {
        "verdict": { "enum": ["pass", "warn", "fail", "inconclusive"] },
        "signals": { "type": "array" }
      }
    },
    "safety_verdict": {
      "type": "object",
      "required": ["verdict", "signals"],
      "properties": {
        "verdict": { "enum": ["pass", "warn", "fail", "inconclusive"] },
        "signals": { "type": "array" }
      }
    },
    "non_overridable_result": {
      "type": "object",
      "required": ["passed", "signals"],
      "properties": {
        "passed": { "type": "boolean" },
        "signals": { "type": "array" }
      }
    },
    "decision": {
      "enum": ["promote", "promote-with-digest", "hold-for-review", "block"]
    },
    "decided_at_utc": { "type": "string", "format": "date-time" },
    "pipeline_run_id": { "type": "string" },
    "override": {
      "type": "object",
      "required": ["override_class", "approvers", "expires_at_utc", "scope",
                   "containment_plan_ref", "legal_notify_sent"],
      "properties": {
        "override_class": { "enum": ["reliability", "safety", "both"] },
        "approvers": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["role", "name", "signed_at_utc"],
            "properties": {
              "role": { "type": "string" },
              "name": { "type": "string" },
              "signed_at_utc": { "type": "string", "format": "date-time" }
            }
          },
          "minItems": 2
        },
        "expires_at_utc": { "type": "string", "format": "date-time" },
        "scope": { "type": "string" },
        "containment_plan_ref": { "type": "string" },
        "legal_notify_sent": { "type": "boolean" }
      }
    }
  }
}
```

### 2.8 Worked example — a conflicted release decision

Illustrative. A candidate build improves eval-set accuracy by 0.3% but regresses the
canary p95 latency by 14%.

```json
{
  "build_id": "svc-advice-prod-2026-11-08-001",
  "gate_policy_version": "2026-11-01",
  "reliability_verdict": {
    "verdict": "fail",
    "signals": [
      {"name": "performance_envelope_p95_delta", "observed": 0.14, "threshold": "<= 0.10", "state": "fail"},
      {"name": "canary_pass_rate", "observed": 0.998, "threshold": ">= 0.995", "state": "pass"},
      {"name": "canary_slo_burn_rate_1h", "observed": 0.8, "threshold": "< 1.0", "state": "pass"},
      {"name": "canary_slo_burn_rate_6h", "observed": 0.4, "threshold": "< 1.0", "state": "pass"},
      {"name": "provider_health_canary", "observed": "green", "threshold": "== green", "state": "pass"}
    ]
  },
  "safety_verdict": {
    "verdict": "pass",
    "signals": [
      {"name": "safety_eval_pass_rate", "observed": 0.991, "threshold": ">= 0.985", "state": "pass"},
      {"name": "redteam_fixture_pass_rate", "observed": 1.0, "threshold": "== 1.0", "state": "pass"},
      {"name": "policy_class_coverage", "observed": 1.0, "threshold": "== 1.0", "state": "pass"},
      {"name": "adversarial_eval_regression", "observed": 0, "threshold": "<= 0", "state": "pass"}
    ]
  },
  "non_overridable_result": {"passed": true, "signals": []},
  "decision": "block",
  "decided_at_utc": "2026-11-08T14:12:33Z",
  "pipeline_run_id": "run-2026-11-08-001"
}
```

**Discussion.** The safety peer wants to promote (eval win). The AI SRE blocks (p95
regression). The matrix returns `block` with `override_class: reliability`.

- The safety peer **cannot** unilaterally override: the override class is
  *reliability*, which requires the reliability approver pair. The safety peer's role
  is to be *informed*.
- The reliability override, if sought, has a 24h window. The build is eligible for the
  override if the team commits to a mitigation (e.g., concurrency cap, pre-warm step,
  feature-flagged activation) and the automatic-rollback on burn-rate alert is active.
- If the override is not sought, the build returns to the engineering queue for a
  latency fix.

The inverse scenario — safety fail, reliability pass — would require the three-approver
safety override and a containment plan (dark-launch or internal-only). The asymmetry
between the two gates' override shapes is intentional.

## 3. Validation steps

1. **Matrix coverage.** Automated: the `resolution_matrix` covers all 16 cells of
   {pass, warn, fail, inconclusive} × {pass, warn, fail, inconclusive}, or has a named
   default that is `block`. Missing pairs default to block and surface in the audit.

   ```bash
   python3 - <<'EOF'
   import yaml, itertools
   cfg = yaml.safe_load(open('config/release/gate-policy.yaml'))
   states = ['pass','warn','fail','inconclusive']
   covered = {(r['reliability'], r['safety']) for r in cfg['resolution_matrix']}
   missing = set(itertools.product(states, states)) - covered
   if missing:
       print("DEFAULT BLOCK for:", sorted(missing))
   EOF
   ```

2. **Non-overridable integrity.** Automated: `non_overridable` list is signed off by
   four named roles and the signature set is current (within the last 90 days).

3. **Override-record completeness.** Automated: every audit record with
   `decision != promote*` and a subsequent `promote-after-override` record has an
   `override` sub-object with the required approvers, expiry, scope, and (for safety)
   containment-plan link.

4. **Override cardinality.** Automated: reliability overrides have >= 2 approvers;
   safety overrides have >= 3 approvers; both-class overrides have >= 5 (reliability
   pair + safety triple).

5. **Role-separation check.** A single human cannot satisfy two different approver
   roles on the same override. Automated: approver-role × approver-name is checked
   for duplicates.

6. **Expiry enforcement.** Automated: an override's `expires_at_utc` is enforced; a
   post-promote evaluation after expiry treats the build as un-overridden (a repeated
   failure becomes a block again at the next pipeline run).

7. **Automatic-rollback integration.** Drill semi-annually: inject a post-promote
   SLO burn-rate alert on a build released under a reliability override and verify
   that `mod-111`'s rollback runbook executes automatically.

8. **Quarterly review.** The AI SRE lead, safety peer lead, security-engineering lead,
   and `ai-risk-engineer` sign the current `non_overridable` list; the signatures land
   as attestations in the compliance store.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at >= 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Gates are independent and both evaluated *(hard gate)* | One gate shortcuts the other | Both evaluated but reconciliation implicit | Both always evaluated; reconciliation explicit |
| 2 | Conflict-resolution matrix coverage *(hard gate)* | Missing cells with no default | Covered but asymmetric where asymmetry is wrong | All 16 cells + explicit default |
| 3 | Override policy asymmetry *(hard gate)* | Same policy for both | Policy differs but shapes overlap | Reliability = engineering-owned, 24h, 2 approvers; safety = narrow, 72h, 3 approvers + containment + legal notify |
| 4 | Non-overridable list exists + signed *(hard gate)* | Missing | Present, unsigned | Present, signed by four named roles |
| 5 | Audit schema on every decision | None | Logged but un-schematised | JSON Schema + validation + compliance-store retention |
| 6 | Automatic rollback on post-promote burn | None | Manual | Automatic; interlocked with mod-111 |
| 7 | Role-separation enforced | Not checked | Policy-only | Policy + mechanical check in the pipeline |
| 8 | Vocabulary consistency with mod-108 / mod-111 | Divergent | Partial | Reuses mod-108 fixture-class vocab + mod-111 rollback vocab |
| 9 | Advisory signals feed the audit | Ignored | Logged but not surfaced | Logged + weekly digest to both leads |
| 10 | Quarterly review attested | Missing | Ad-hoc | Quarterly attestation stored in the compliance store |

## 5. Common mistakes

- **Letting one gate's pass count as the other's pass.** A safety eval win does not
  make a p95 regression safe to ship; a p95 win does not make a safety regression safe.
  The gates measure different things.
- **Single approver on safety override.** The three-approver rule is deliberate; safety
  failures are often not reversible. One person cannot carry the organisation's risk.
- **Full-traffic safety override.** Never available in the standing policy. If a
  full-traffic safety override is "needed," the correct framing is that it is an
  incident, not a release.
- **Short-circuiting the gate for "obvious" promotions.** Even "obvious wins" must
  pass the gates; otherwise the gates rot and the audit trail breaks.
- **Overrides without expiry.** An override without an expiry becomes a permanent state.
  The expiry forces a decision to fix or to re-override with fresh approvers.
- **Treating `inconclusive` as `pass`.** Inconclusive is treated as fail for blocking
  purposes. Giving the benefit of the doubt in the gate undermines the gate.
- **Audit-less overrides.** Every override is an auditable event; the compliance store
  retention matches exercise-03's policy.
- **Merging the gate-policy change with a build.** Policy changes are separate PRs,
  reviewed by both leads + security-engineering + `ai-risk-engineer`. A build PR that
  edits the gate policy is rejected on review.
- **Dropping the `non_overridable` list.** The list is the floor. Leaving it out
  removes the floor; the hard gate exists for a reason.
- **Allowing the exec-sponsor role to satisfy multiple approver slots.** Role-separation
  is enforced mechanically; an executive cannot stand in for the safety-engineering
  lead or the AI risk engineer.
- **Letting the reliability gate shadow the safety gate (or vice versa).** The gates
  run independently on every candidate. Hiding one behind the other is a reporting
  failure.

## 6. References

- Module chapter on reliability-and-safety gate reconciliation (this module) — the
  engineering framing for the matrix, the override asymmetry, and the
  `non_overridable` floor.
- `mod-111`, *Release engineering and progressive delivery* — the pipeline this gate
  plugs into, the rollback runbook, the canary telemetry.
- `mod-103`, *SLO engineering* — the burn-rate math used by the reliability gate's
  burn-rate signals.
- `mod-108` chapter 03 and chapter 05 — the fixture-class vocabulary used by the
  override containment-plan documents and the post-incident regeneration of this gate.
- `mod-108` chapter 06 — the trigger that fires when a safety override is granted
  (legal / compliance notification).
- `mod-110` exercise-01 — the guardrail sidecar latency budget whose breach is a
  reliability-gate signal and whose degraded-mode entry is auditable here.
- `mod-110` exercise-03 — the compliance log store used as the audit sink.
- OWASP Machine Learning Security Top 10,
  [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary used by the safety-eval and red-team signals.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — tactic / technique
  vocabulary used by the adversarial-eval signals and the control-coverage
  attestation referenced by the `non_overridable` floor.
- NIST AI Risk Management Framework,
  [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the Measure / Manage / Govern functions that the gate reconciliation implements
  at release time.
- NIST AI 600-1, *Generative AI Profile*,
  [nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
  — GAI-specific action vocabulary for the safety gate's required signals.
- EU AI Act, Reg. (EU) 2024/1689 — Art. 9 (risk management) and Art. 15 (accuracy,
  robustness, cybersecurity) inform the signals consumed by the safety gate for
  high-risk systems.
