# Exercise 03 — Compliance Telemetry Hooks for EU AI Act Article 72 — Reference Solution

Paired learning file: `lessons/mod-110-security-safety-and-compliance-reliability-crossover/exercises/exercise-03-compliance-telemetry-hooks-eu-ai-act-article-72.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred
> below is derived from this module's chapter on compliance-as-a-reliability-surface
> and the Art. 72 touchpoint cited in `mod-108` chapter 06. If the authored exercise
> narrows or widens scope on a subsequent content cycle, this solution should be
> regenerated.

> **Scope boundary.** This is engineering telemetry that *supports* compliance under
> EU AI Act Reg. (EU) 2024/1689, Art. 72 (post-market monitoring system, "PMMS"). The
> AI SRE's deliverable is the telemetry and evidence pipeline; the *compliance
> determination* — whether the PMMS is sufficient, whether any Art. 73 serious-incident
> threshold is crossed, how the evidence is presented to a supervisory authority — is
> legal and the `ai-risk-engineer` function's responsibility. See `mod-108` chapter 06
> for the handoff. **This is not legal advice.**

## 1. Solution overview

The deliverable is a committed set of artifacts in `compliance/eu-ai-act-art-72/`:

1. A **mapping document** that maps Art. 72's PMMS obligations to concrete telemetry
   signals the organisation already produces (or should produce).
2. A **telemetry hooks specification** describing each signal: what emits it, what it
   contains, retention, access control, and how it ties back to the obligation.
3. A **evidence-pipeline spec** describing how signals are aggregated into the
   periodic PMMS evidence package the provider is expected to be able to produce on
   request.
4. A **retention and access policy** for the compliance log store.
5. A **worked example** PMMS evidence package for one illustrative reporting period.

The approach treats Art. 72 as a **systems engineering problem**: the PMMS is a
closed-loop monitoring system for the deployed high-risk AI system, and the AI SRE's
job is to make the loop legible and auditable. Most of the required telemetry is
already produced for reliability (see `mod-104`); Art. 72 adds (a) *intent* (the
signal is labelled as compliance evidence), (b) *retention under legal-hold-equivalent
rules*, and (c) *aggregation into a periodic evidence shape*.

## 2. Implementation — worked answer

### 2.1 Primary source — what Art. 72 says, in engineering terms

The following paraphrase is the engineering-facing summary; it is **not** a legal
interpretation. Where a specific clause matters for a filing, legal re-verifies against
the official text at [eur-lex.europa.eu/eli/reg/2024/1689/oj](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).

| Obligation (paraphrased from Art. 72) | Engineering implication |
|----------------------------------------|---------------------------|
| Providers of high-risk AI systems establish and document a post-market monitoring system (PMMS) proportionate to the nature and risks of the system. | The organisation has a documented monitoring plan; the engineering surface is a reference implementation of that plan. |
| The PMMS collects, documents, and analyses relevant data provided by deployers or collected through other sources on the performance of the system throughout its lifetime. | Telemetry is collected, retained, and analysed; the analysis is documented. |
| The PMMS enables the provider to evaluate continuous compliance with Chapter III, Section 2 requirements (data governance, technical documentation, record-keeping, transparency, human oversight, accuracy / robustness / cybersecurity). | Each Section 2 requirement has at least one telemetry hook that produces evidence. |
| The PMMS is based on a monitoring plan that is part of the technical documentation (Annex IV). | The monitoring plan is version-controlled and linked from the technical-documentation index. |
| The Commission adopts implementing acts specifying detailed provisions of the PMMS (originally planned under Art. 72(3); subsequently restructured by the Digital Omnibus on AI, Reg. (EU) 2026/1744, which replaces the implementing act with non-binding guidance due by 2027-09-02). | The organisation's monitoring plan is designed to be *compatible* with forthcoming Commission guidance; design for the plan, not the (still-unpublished) guidance. |
| For high-risk systems that are safety components of products already regulated under Union harmonisation legislation (Annex I), the PMMS is integrated with the sectoral post-market surveillance system. | For regulated products (e.g., medical devices), the AI PMMS aligns with the sectoral system (e.g., FDA PCCP; see `mod-108` chapter 06). |

<!-- Note: Art. 72(3) implementing-act status should be re-verified against the Commission's work programme at the time of a filing; the Digital Omnibus restructuring was adopted 2026-07-27 and the non-binding guidance deadline is 2027-09-02 per the chapter-06 citation in mod-108. -->

### 2.2 Mapping document — `compliance/eu-ai-act-art-72/mapping.md`

```markdown
# EU AI Act Art. 72 PMMS — engineering mapping — v2026-11-01

## 1. Scope

This document maps Art. 72 obligations to the telemetry that evidences them for the
in-scope high-risk AI systems. The in-scope systems are listed in the `high-risk-ai-
system-register.md` sibling document (owner: `ai-risk-engineer`).

## 2. Mapping — Chapter III Section 2 requirements to telemetry

| Section 2 requirement (Art. ref) | Engineering telemetry evidencing it | Signal owner |
|-----------------------------------|---------------------------------------|---------------|
| Risk management (Art. 9) | Risk-register change log + residual-risk reassessment events (see `compliance_risk_register_change_total`) | ai-risk-engineer |
| Data and data governance (Art. 10) | Dataset lineage records, dataset-quality metrics, dataset version pins at training and fine-tune time | data-engineering |
| Technical documentation (Art. 11) | Doc-index integrity: SHA pins on each Annex-IV section; update-log emits an event per edit | ai-risk-engineer |
| Record-keeping (Art. 12) | Audit-log emits (see §2.3 below) covering the categories enumerated in Annex IV's record-keeping expectations | platform-engineering + ai-sre |
| Transparency and provision of information (Art. 13) | Transparency-disclosure version pin on the user-facing surface; measurement of which users received which version | product-engineering |
| Human oversight (Art. 14) | Oversight-tool availability SLO, oversight-event counters (reviewer-intervened, reviewer-escalated), reviewer-training attestation | ai-sre + human-review-operations |
| Accuracy (Art. 15) | Eval-set pass-rate, canary-eval pass-rate, drift metrics on business-level accuracy signals | ai-engineering + ai-sre |
| Robustness (Art. 15) | Robustness-eval pass-rate, adversarial-eval results, redteam fixture coverage | ai-engineering + security-engineering |
| Cybersecurity (Art. 15) | Control-coverage attestation, vulnerability-scan results, SIEM detection coverage for ATLAS tactics | security-engineering |

## 3. Mapping — PMMS lifecycle events to telemetry

| PMMS lifecycle event | Engineering telemetry |
|-----------------------|-----------------------|
| System placed on the market | Release event (`release.ai_system`) with the model version ID, intended purpose, and Annex-IV doc pins. |
| Deployer feedback / complaint received | Complaint intake event (`compliance.complaint_received`) with a redacted summary, deployer / user cohort, received-at. |
| Serious incident candidate | Trigger per `mod-108` chapter 06 trigger card; emits `compliance.art73_candidate` with the artifact-package reference. (The determination itself is legal's.) |
| Monitoring plan update | `compliance.monitoring_plan.version` bump; linked to the technical-documentation index. |
| Periodic PMMS review | `compliance.pmms_review.completed` event with reviewer attestation. |
| Corrective action taken on the system | `release.ai_system.corrective_action` event with the action description and linked incident. |

## 4. Mapping — Art. 73 serious-incident reporting hooks

Art. 73 reporting is adjacent to Art. 72 but separate. The PMMS feeds the trigger
conditions; Art. 73 reporting is engaged per `mod-108` chapter 06. The hooks that
support the engagement are:

- `rb-art73-candidate` runbook link on the compliance dashboard.
- `compliance.art73_candidate` event (emitted once; de-duplicated by incident ID).
- Artifact-package ID (from `mod-108` chapter 06 §4) in the event's attributes.

The reporting windows (2 days / 10 days / 15 days depending on harm category) are the
operational deadlines that back-propagate into the on-call expectations. The AI SRE
does not file; the AI SRE enables legal to file on time.
```

### 2.3 Telemetry hooks — `compliance/eu-ai-act-art-72/hooks.yaml`

```yaml
version: 2026-11-01
scope: in-scope high-risk AI systems per high-risk-ai-system-register.md

schema:
  common_attributes:
    - key: system_id
      required: true
      description: Unique ID of the AI system from the system register.
    - key: model_version_id
      required: true
      description: Model version (provider ID + fine-tune hash if applicable).
    - key: monitoring_plan_version
      required: true
      description: Version of the monitoring plan this signal is emitted under.
    - key: annex_iv_section_ref
      required: false
      description: Optional link to the Annex-IV section the signal evidences.

hooks:
  - name: eval_canary_pass_rate
    evidences:
      - Art. 15 accuracy
      - Art. 15 robustness
    emit: per model promotion + hourly in steady-state
    content:
      - canary-eval set ID, version, SHA
      - pass count, fail count, by eval class
      - sample-size summary
    retention: 36 months
    access: ai-risk-engineer, ai-sre, legal
    system_of_record: evals-service
    pii: none

  - name: human_oversight_availability
    evidences:
      - Art. 14 human oversight
    emit: 1 min interval
    content:
      - oversight tool uptime
      - reviewer-queue depth
      - escalation-rate (count / interval)
    retention: 24 months
    access: ai-sre, human-review-operations, legal
    system_of_record: human-review-service
    pii: none

  - name: transparency_disclosure_delivery
    evidences:
      - Art. 13 transparency
    emit: per user session
    content:
      - disclosure version served (pinned SHA)
      - locale
      - user-cohort (not user ID)
    retention: 24 months
    access: ai-risk-engineer, legal
    system_of_record: edge-telemetry
    pii: none (cohort-aggregated)

  - name: data_governance_dataset_pin
    evidences:
      - Art. 10 data governance
    emit: per training run + per fine-tune run
    content:
      - dataset lineage (parent datasets)
      - dataset SHA, size, class-distribution summary
      - data-quality-scan results
    retention: 10 years (system-lifetime)
    access: data-engineering, ai-risk-engineer, legal
    system_of_record: data-lineage-service
    pii: dependent on dataset; records never contain raw records

  - name: record_keeping_audit_emit
    evidences:
      - Art. 12 record-keeping
    emit: per request (sampled per the sampling policy) + every event of record
    content: see audit-schema.yaml
    retention: 6 years
    access: ai-sre, security-engineering, legal
    system_of_record: audit-log
    pii: redacted; customer content is replaced with content-digest hashes and
      cohort-level attributes; raw content is in the controlled-access store (see
      exercise-05)

  - name: risk_register_change
    evidences:
      - Art. 9 risk management
    emit: per change to the risk register
    content:
      - risk ID, pre-change state, post-change state
      - author role
      - rationale link
    retention: 10 years (system-lifetime)
    access: ai-risk-engineer, legal
    system_of_record: risk-register-repo
    pii: none

  - name: incident_declared
    evidences:
      - Art. 72 lifecycle (serious incident candidate feeds Art. 73)
    emit: on incident declare
    content:
      - incident ID, severity, UTC timestamps
      - serious-incident-candidate flag (boolean)
      - artifact-package ID if flag is true
    retention: 10 years
    access: ai-sre, security-engineering, legal
    system_of_record: ir-service
    pii: none (artifact-package holds content)

  - name: control_coverage_attestation
    evidences:
      - Art. 15 cybersecurity
    emit: weekly
    content:
      - control-ID, attested-state
      - evidence-link (SIEM rule, test result, scan)
      - attesting role
    retention: 36 months
    access: security-engineering, ai-risk-engineer, legal
    system_of_record: control-plane-registry
    pii: none

  - name: monitoring_plan_version
    evidences:
      - Art. 72 PMMS definition
    emit: per monitoring plan change
    content:
      - version, SHA, date
      - approvers, change summary
    retention: 10 years
    access: ai-risk-engineer, legal
    system_of_record: compliance-docs-repo
    pii: none
```

### 2.4 Evidence pipeline — `compliance/eu-ai-act-art-72/evidence-pipeline.md`

```markdown
# PMMS evidence pipeline — v2026-11-01

## 1. Shape of a PMMS evidence package

A PMMS evidence package is produced on a cadence (quarterly by default; the monitoring
plan may set a different cadence per system). The package has:

1. **Cover.** System IDs in scope, reporting period (UTC), monitoring-plan version pin.
2. **Section-2 coverage table.** For each Section-2 requirement, the signals that
   evidence it in the period + the latest attestation state.
3. **Analysis narrative.** A short written analysis per requirement, referencing the
   signals. Not interpretive beyond "within tolerance / at threshold / out of
   tolerance."
4. **Incident digest.** Serious-incident candidates in the period (if any) + the
   handoff package IDs (per `mod-108` chapter 06).
5. **Corrective actions.** Actions taken on the system in the period.
6. **Appendix.** Dashboard links pinned to the window, dataset / release / audit event
   counts, SHA pins for the signals used.

The package is **generated** from the telemetry — not re-authored — and is attested
to by the `ai-risk-engineer` and the AI SRE lead; the signatures land in the package.

## 2. Pipeline

1. **Collection.** Signals from §2.3 land in the compliance log store (append-only,
   object-locked; see §2.5).
2. **Validation.** A nightly job validates every incoming signal against the schema;
   non-conforming signals are quarantined and alerted.
3. **Aggregation.** A per-period batch job aggregates signals into the Section-2
   coverage table. The job is deterministic and reproducible from the raw signals.
4. **Analysis.** The `ai-risk-engineer` reviews the aggregated tables and authors the
   analysis narrative within the first week of the following period.
5. **Attestation.** The `ai-risk-engineer` and AI SRE lead co-attest; the attestation is
   itself emitted as `compliance.pmms_review.completed`.
6. **Publication.** The package lands in the compliance repository under
   `packages/YYYY-QN/<system_id>.md`. External distribution (to a deployer, to a
   supervisory authority) is legal's decision; engineering does not distribute
   externally.

## 3. Freshness SLO

Signal freshness in the compliance log store has an SLO (99% of in-window signals
present within 24h of emit). The SLO has a burn-rate alert; the alert pages the AI SRE
compliance on-call.

## 4. Change management

A change to a hook schema requires:

- A monitoring-plan version bump.
- A migration plan for in-flight signals.
- An attestation that evidence for the affected Section-2 requirement remains derivable
  across the change (either via the old schema or the new).

Change-without-migration is a compliance incident.
```

### 2.5 Retention and access policy — `compliance/eu-ai-act-art-72/retention-access.md`

```markdown
# Compliance log store — retention and access — v2026-11-01

## 1. Store characteristics

- Append-only, object-locked for the retention window.
- Role-gated access; named access is logged per read.
- Geographically resident per the organisation's data-residency policy (EU-resident
  for EU-market systems).
- Encrypted at rest and in transit; keys managed per the security-engineering key
  policy.

## 2. Retention windows

- Signals evidencing Art. 10 (data governance) and Art. 9 (risk management): system
  lifetime, minimum 10 years.
- Audit log (Art. 12): 6 years.
- Human oversight and transparency: 24 months.
- Accuracy / robustness telemetry: 36 months.
- Incident declaration metadata: 10 years.
- Monitoring-plan version history: system lifetime.

Retention ends are superseded by legal hold at any time (see `mod-108` chapter 06).

## 3. Access

- `ai-risk-engineer`: full read.
- `ai-sre` (lead rotation): read for compliance-engineering purposes; writes are only
  via the hook pipeline.
- `security-engineering`: read for security / incident purposes.
- `legal`: read for compliance / regulatory purposes.
- All reads are logged.

## 4. Deletion

- Deletion is possible only after the retention end **and** legal clearance.
- The deletion operation itself emits an event to the audit log.
```

### 2.6 Worked example — PMMS evidence package

```markdown
---
system_id: svc-advice-prod
reporting_period: 2026-Q3 (2026-07-01 → 2026-09-30 UTC)
monitoring_plan_version: 2026-06-04
attestations:
  - role: ai-risk-engineer
    signed_at: 2026-10-06T14:12:00Z
  - role: ai-sre-lead
    signed_at: 2026-10-06T15:03:00Z
---

# PMMS evidence package — svc-advice-prod — 2026-Q3

## Cover

- **System ID:** `svc-advice-prod`
- **Reporting period:** 2026-07-01 → 2026-09-30 UTC (UTC-closed)
- **Monitoring plan version:** 2026-06-04 (link to repo tag)
- **Attestations:** ai-risk-engineer (2026-10-06 14:12 UTC), ai-sre-lead (2026-10-06 15:03 UTC)

## 1. Section-2 coverage

| Section-2 requirement | Signals evidencing | State | Notes |
|------------------------|---------------------|-------|-------|
| Risk management (Art. 9) | `risk_register_change` (14 events) | within tolerance | Two residual-risk reassessments. |
| Data governance (Art. 10) | `data_governance_dataset_pin` (2 training, 4 fine-tune events) | within tolerance | One fine-tune delayed pending data-quality scan. |
| Technical documentation (Art. 11) | Annex-IV doc-index SHA pins (quarterly check) | within tolerance | No doc drift. |
| Record-keeping (Art. 12) | `record_keeping_audit_emit` (3.1B events; sampling ratio documented) | within tolerance | One sampling-policy change mid-quarter; migration validated. |
| Transparency (Art. 13) | `transparency_disclosure_delivery` (411.2M sessions; disclosure-version coverage 100%) | within tolerance | — |
| Human oversight (Art. 14) | `human_oversight_availability` (99.94% availability; 7,120 escalations; 14 interventions) | within tolerance | Oversight-queue SLO burn 0.4%. |
| Accuracy (Art. 15) | `eval_canary_pass_rate` (28-day rolling pass rate 98.6%) | within tolerance | One canary-eval set expansion. |
| Robustness (Art. 15) | Robustness-eval pass rate 97.1%; redteam coverage of ATLAS tactics full | within tolerance | Three redteam fixtures added. |
| Cybersecurity (Art. 15) | `control_coverage_attestation` (weekly, 100% attested) | within tolerance | One vulnerability remediation within SLO. |

## 2. Analysis narrative

(Per-requirement one-paragraph write-ups; summarised above. Full text in appendix.)

## 3. Incident digest

- `inc-2026-08-22-latency-provider-regression` — `prompt-config` + `provider`
  category; no Art. 73 serious-incident candidate; no handoff package.
- `inc-2026-09-14-oversight-queue-backlog` — `scheduler` category; oversight-queue
  SLO burned 0.4%; corrective action AI-14 completed 2026-09-19.

## 4. Corrective actions

| ID | Description | Owner | Due | Status |
|----|-------------|-------|-----|--------|
| AI-14 | Autoscale rule for oversight queue | ai-sre | 2026-09-19 | done |
| AI-19 | Redteam fixture family for `persona`-retok | redteam | 2026-11-02 | in-progress (post-period) |

## 5. Appendix

- Grafana `compliance/svc-advice-prod` pinned to 2026-Q3.
- Audit emit counts by category.
- Dataset SHA pins used in the period.
- Monitoring-plan diff from 2026-06-04 → 2026-06-04 (none in period).
```

## 3. Validation steps

1. **Mapping completeness.** Automated: every Section-2 requirement (Art. 9–15) has at
   least one named hook in the mapping. A requirement without a hook fails the lint.

   ```bash
   python3 - <<'EOF'
   import yaml
   hooks = yaml.safe_load(open('compliance/eu-ai-act-art-72/hooks.yaml'))
   requirements = {'Art. 9 risk', 'Art. 10 data', 'Art. 11 technical',
                   'Art. 12 record-keeping', 'Art. 13 transparency',
                   'Art. 14 human oversight', 'Art. 15 accuracy',
                   'Art. 15 robustness', 'Art. 15 cybersecurity'}
   evidenced = set()
   for h in hooks['hooks']:
       for e in h.get('evidences', []):
           for r in requirements:
               if r.split()[0] + ' ' + r.split()[1] in e.lower() or r in e:
                   evidenced.add(r)
   missing = requirements - evidenced
   assert not missing, f"unmapped: {missing}"
   EOF
   ```

2. **Schema validation.** Nightly: validate every emitted signal against the hook
   schema; non-conforming signals are quarantined and alerted. The alert pages the AI
   SRE compliance on-call at any sustained quarantine rate > 0.01%.

3. **Freshness SLO.** The compliance log store's freshness SLO has a multi-window
   multi-burn-rate alert (per `mod-103`); the alert pages when freshness burns at the
   SLO's fast-burn rate.

4. **PMMS review on cadence.** Automated: the `compliance.pmms_review.completed` event
   must emit within the first week of each reporting period. A missing event is a
   compliance incident; the IC follows `rb-pmms-review-overdue`.

5. **Attestation coverage.** Every published PMMS evidence package has both
   attestations in its front-matter. Automated check.

6. **Access-log integrity.** Weekly: the access log on the compliance store has no gaps
   > 5 minutes (the monitoring probe emits every 60s; gaps > 5m suggest tampering or
   outage). Integrity-check failure is a compliance incident.

7. **Change-management check.** Any hook schema change must land with a matching
   monitoring-plan version bump and a migration-plan note. PRs that touch
   `hooks.yaml` without touching the monitoring-plan version fail CI.

8. **Semi-annual tabletop.** Legal and the `ai-risk-engineer` walk a scripted
   supervisory-authority request through the evidence pipeline; the AI SRE's
   deliverable is the package within the operational SLA.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at >= 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Section-2 coverage *(hard gate)* | One or more requirement without a hook | All requirements mapped but some thin | Every Section-2 requirement mapped to >= 1 hook with owner |
| 2 | Hook schema completeness *(hard gate)* | Missing retention / access / PII fields on a hook | Partial | Every hook has emit cadence, content, retention, access, SoR, PII posture |
| 3 | Evidence pipeline reproducible | Narrative-only pipeline | Pipeline but non-reproducible | Deterministic aggregation + attested sign-off |
| 4 | Retention policy mapped to obligation *(hard gate)* | Single retention window across | Mixed | Per-signal retention aligned to the obligation it evidences; legal-hold-override noted |
| 5 | Access policy role-gated + logged *(hard gate)* | Open or unlogged | Role-gated or logged | Role-gated and read-logged |
| 6 | Change management interlocked with monitoring plan | Changes untracked | Tracked | Hook changes gated on monitoring-plan version bump |
| 7 | Incident-reporting hook present (Art. 73 adjacency) | Missing | Mentioned | Emitted `art73_candidate` with handoff-package ID; defers determination to legal |
| 8 | PMMS review on cadence | Missing | Ad-hoc | Emitted event + cadence SLO + overdue runbook |
| 9 | Freshness SLO | Missing | Static threshold | Multi-window multi-burn-rate per mod-103 |
| 10 | Scope disclaimer ("not legal advice") | Missing | Implicit | Explicit; defers compliance determination to legal / ai-risk-engineer |

## 5. Common mistakes

- **Treating the exercise as a filing exercise.** The AI SRE's deliverable is the
  telemetry and evidence pipeline, not a submission to a supervisory authority. Legal
  owns submission.
- **Collapsing all signals into a single retention window.** Each signal retention
  should be mapped to the obligation it evidences; Art. 10 (data governance) typically
  requires system-lifetime retention while Art. 13 (transparency) does not.
- **Emitting raw customer content as "audit."** The audit emit should carry
  content-digest hashes and cohort attributes; raw content lives in the
  controlled-access store per exercise-05's discipline.
- **Treating the (not-yet-published) Commission guidance as a hard constraint.** The
  guidance under the Digital Omnibus restructuring is non-binding and due in 2027. The
  monitoring plan should be *compatible* with forthcoming guidance, not *waiting* for
  it. Design for the obligation under Art. 72 as written.
- **Dependency on human attestation for continuous evidence.** The PMMS is a
  monitoring system; most evidence should be machine-emitted. Attestation is for the
  periodic review, not for every signal.
- **Mixing PMMS evidence with product analytics.** The compliance store is not a
  product analytics store; the schema, retention, and access are compliance-specific.
  Analytics has its own store; some signals are emitted to both.
- **Failing the freshness SLO silently.** If the compliance log store is behind by
  more than 24h, the PMMS is not operating. The freshness SLO is as load-bearing as the
  user-facing SLO.
- **Over-aggregation on a single dashboard.** The evidence package is the
  deliverable, not a dashboard. Dashboards are internal; the evidence package is the
  artifact legal / the authority consumes.
- **Hard-coding Article numbers in code.** Hook schemas reference the obligation by a
  stable slug (e.g., `art-10-data-governance`) that is defined in a vocabulary file;
  Article renumbering under the Act's future amendments would then land as a single
  vocabulary change, not a schema-wide refactor.
- **Including `art73_candidate` determinations in the engineering emit.** The hook
  signals a *candidate*; the determination is legal's.

## 6. References

- **Primary source.** EU AI Act — Regulation (EU) 2024/1689 of the European Parliament
  and of the Council laying down harmonised rules on artificial intelligence
  (Artificial Intelligence Act). Art. 72 (post-market monitoring) and Art. 73 (serious-
  incident reporting). Official text at
  [eur-lex.europa.eu/eli/reg/2024/1689/oj](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- **Related.** Digital Omnibus on AI, Regulation (EU) 2026/1744 — restructured the
  implementing-act expectation of Art. 72(3) into non-binding guidance due by
  2027-09-02 (per the `mod-108` chapter 06 citation; re-verify against the Commission
  work programme at the time of a filing).
- Module chapter on compliance-as-a-reliability-surface (this module) — the engineering
  framing used here.
- `mod-108` chapter 06, *Legal / Compliance / Regulator-Facing Handoff for AI-Safety
  Incidents* — the Art. 73 trigger and handoff that the Art. 72 `incident_declared`
  hook feeds into.
- `mod-104`, *Observability for LLM and GPU workloads* — the telemetry conventions
  that the hooks reuse.
- `mod-103`, *SLO engineering* — the multi-window multi-burn-rate alert math used for
  the freshness SLO.
- NIST AI Risk Management Framework,
  [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the Measure / Govern functions and the risk-management vocabulary that aligns with
  Art. 9's requirement.
- NIST AI 600-1, *Generative AI Profile*,
  [nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
  — the GAI-specific risk vocabulary that overlaps with Art. 15 robustness /
  cybersecurity hooks.
- OWASP Machine Learning Security Top 10,
  [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary for Art. 15 cybersecurity signals.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — tactic / technique
  vocabulary for the control-coverage attestation.
- ISO/IEC 42001:2023, *Information technology — Artificial intelligence — Management
  system* — adjacent management-system standard; Annex A controls may be referenced
  alongside Art. 72 for organisations that adopt it.
