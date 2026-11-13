# Exercise 06 — BCP Artifact Pack for Regulated-Industry Audit — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-06-bcp-artifact-pack-for-regulated-industry-audit.md`.

> **Scope note.** The paired exercise prompt asks the learner to assemble the business-continuity artifact pack an external auditor expects for a regulated-industry assessment of an AI workload — the bundle that proves the plans from exercises 01–05 exist, are current, and are exercised. The reference solution works the pack against the same fabricated `chatcat` product used across this module family, assuming the audit framework is a representative mix of ISO 22301 (business continuity), the SOC 2 CC9 (business continuity) common criteria, and a sector-specific module that calls out the AI-specific additions. A learner whose scenario assumes a different audit framework (HIPAA, PCI DSS, EU AI Act, DORA) should be scored against the rubric in §4 for how well they mapped the generic shape to that framework.

## 1. Solution overview

A BCP audit artifact pack is a **curated package for an external reader**, not a
dump of every file in the DR repository. It should let an auditor, with no insider
knowledge, verify five things in a bounded reading session:

1. **Plans exist** and were approved by the right roles.
2. **Plans are current** (within the review cadence the policy declares).
3. **Plans are exercised** (drills happen, scores exist, action items are closed).
4. **Controls are in place** that make the plans executable (the preventive controls
   from exercise 01 §3 and the residency controls from exercise 04).
5. **AI-specific controls are in place** — the three that generic BCP audits
   miss and that AI-sector audits increasingly demand: weight attestation, judge-
   pipeline snapshotting, and stable-prefix-hash CI check.

The pack is **evidence-first**: each claim has a pointer to a dated artifact in the
repository, and each artifact has a review cadence and an owner. The pack is
**read-order sensitive**: the auditor opens the top-level letter first and the
dependency flows from there.

The deliverable is a committed directory `audits/pack-YYYY-MM-DD/` containing:

- `00-cover-letter.md` — the auditor-facing executive summary.
- `01-control-matrix.csv` — the matrix mapping audit control IDs to internal controls
  to evidence files.
- `02-plan-register.md` — the list of plans in scope with versions, owners, and last
  review dates.
- `03-exercise-history.md` — a 12-month drill history (populated from `mod-113/
  exercise-05`'s `programmes/history/`).
- `04-attestation-index.md` — links to the attestations (residency attestations,
  provider contracts, weight attestations).
- `05-ai-specific-addendum.md` — the AI-specific controls and evidence pointers.
- `06-sampling-guidance.md` — if the auditor wants to sample, which samples to pull
  and how (e.g., "pick any 3 of the last 12 tabletops and request the scribe notes").
- `07-change-log.md` — append-only change log of the pack itself.

### 1.1 Design principles

- **Evidence over narrative.** Every claim has a pointer. "We test the plan annually"
  is not evidence; `programmes/history/full-interruption-2026-06-15.md` is.
- **Fixed structure, versioned pack.** The directory structure does not change
  between quarters; what changes is the contents. The audit team can diff pack-N
  against pack-N-1 to see what moved.
- **AI-specific addendum, not retrofit.** Attempting to fit AI-specific controls
  into legacy audit control IDs produces strained mappings. The addendum lets the
  pack be clean against both generic and AI-aware frameworks.
- **Honest gaps.** Where a plan or control is in flight, the pack says so, with a
  closing date and owner. Concealed gaps become audit findings.

## 2. Implementation — worked answer

### 2.1 Cover letter (`00-cover-letter.md`)

```markdown
# chatcat — Business Continuity Audit Artifact Pack

**Pack version:** 2026-Q4 (2026-11-13). **Pack owner:** AI SRE Director.
**Attested by:** VP, AI Platform + CISO. **Audience:** external audit team.

## Purpose of this pack
This pack is the evidence set for the business-continuity portion of the chatcat
audit. It contains the plans in scope, the register of controls, the drill history,
and the AI-specific addendum. It assumes no prior access; every claim is backed
by a dated artifact.

## Scope
- The chatcat production workload across its two residency scopes (eu-gdpr, us-general).
- The plans in `mod-113/` exercises 01-05 (ISCP, regional-outage playbook, state-
  reconstitution plan, residency-failover design, exercise programme).
- The controls in `mod-113/exercise-01/controls/`, `mod-113/exercise-04/controls/`.
- The exercise history from 2025-11-01 to 2026-11-01 (one year).

## Out of scope
- Enterprise-level BCP (covered under the parent organisation's audit).
- Research training infrastructure (separate audit scope).
- Corporate IT services (parent audit).

## Reading order
1. `00-cover-letter.md` (this file).
2. `02-plan-register.md` — confirms the plans exist and are current.
3. `01-control-matrix.csv` — maps audit controls to our internal controls to evidence
   files.
4. `03-exercise-history.md` — confirms the plans are exercised.
5. `04-attestation-index.md` — confirms the external attestations.
6. `05-ai-specific-addendum.md` — the AI-specific controls.
7. `06-sampling-guidance.md` — suggestions if the auditor wants to sample.

## Known in-flight items (disclosed)
- Judge-pipeline snapshot retention is currently 30 days; policy is being updated
  to 90 days per audit-cycle feedback from 2026-Q2. Change ticket `CHG-2026-1134`,
  closing `2026-12-01`. Owner: ML Platform lead.
- Residency scope `apac-sg` was added 2026-09; attestations exist but the first
  quarterly walkthrough covering this scope is `2026-12-18`.

## Point of contact
- Pack owner: `<name>`, `<email>`, `<phone>`.
- Escalation: `<CISO name>`, `<email>`.

## Attestation
I attest that the artifacts referenced in this pack were, to the best of my
knowledge, accurate as of the pack version date.

Signed: `<pack owner>`, `<date>`.
```

### 2.2 Plan register (`02-plan-register.md`)

```markdown
# Plan register

| Plan ID | Title | Version | Owner | Last review | Next review | Location |
|---------|-------|---------|-------|-------------|-------------|----------|
| POL-ISC-01 | ISC Policy | 2026-11-13 | VP AI Platform | 2026-11-13 | 2027-11-13 | `policy/isc-policy-chatcat.md` |
| BIA-01 | Business Impact Analysis | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-11-13 | `bia/chatcat-bia.md` |
| CTL-01 | Preventive Controls Inventory | 2026-11-13 | AI Security Lead | 2026-11-13 | 2027-05-13 | `controls/chatcat-preventive-controls.md` |
| STR-01 | Recovery Strategy | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-05-13 | `strategy/chatcat-recovery-strategy.md` |
| ISCP-01 | Information System Contingency Plan | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-02-13 | `plan/chatcat-iscp.md` |
| PB-REG-01 | Regional Outage Playbook | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-02-13 | `playbooks/regional-outage.md` |
| SRP-01 | State Reconstitution Plan | 2026-11-13 | Agent Platform lead | 2026-11-13 | 2027-05-13 | `plans/scoutkit-state-reconstitution.md` |
| RES-FO-01 | Residency Failover Design | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-02-13 | `designs/chatcat-residency-failover.md` |
| EXE-PGM-01 | DR Exercise Programme | 2026-11-13 | AI SRE Director | 2026-11-13 | 2027-02-13 | `programmes/dr-exercise-programme.md` |

**Review cadence notes.** The ISCP, playbooks, and exercise programme have a 90-day
review cadence; strategy, controls, and SRP have a 180-day cadence; the policy and
BIA have an annual cadence. These cadences are declared in POL-ISC-01 §Maintenance
and are more aggressive than the minimum the enterprise BCP requires.

**Triggered review events (POL-ISC-01 §Maintenance).** Any of: model-version bump,
provider swap, region addition, retrieval-index architecture change, judge-pipeline
change. Each event opens a triggered-review ticket; see `audits/change-register.md`
for the list.
```

### 2.3 Control matrix (`01-control-matrix.csv`)

```csv
audit_framework,audit_control_id,audit_control_text,internal_control_id,internal_control_name,enforcement_mechanism,evidence_file,last_evidence_date,review_cadence,owner
ISO-22301,8.3.1,Business continuity strategies,STR-01,Recovery Strategy,Document + approval,strategy/chatcat-recovery-strategy.md,2026-11-13,180d,AI SRE Director
ISO-22301,8.4.2,Warning and communication,PB-REG-01 §5,Communications cadence in playbook,Document,playbooks/regional-outage.md,2026-11-13,90d,AI SRE Director
ISO-22301,8.5,Exercise programme,EXE-PGM-01,DR Exercise Programme,Scheduled drills,programmes/dr-exercise-programme.md,2026-11-13,90d,AI SRE Director
ISO-22301,9.1,Monitoring, measurement, analysis, evaluation,PGM-HEALTH-01,Programme health metrics,Alerting,dashboards/dr-programme.json,continuous,continuous,AI SRE Director
SOC-2,CC9.1,Identifies, selects, and develops risk mitigation activities for disruptions,CTL-01,Preventive Controls Inventory,Document + enforcement,controls/chatcat-preventive-controls.md,2026-11-13,180d,AI Security Lead
SOC-2,CC9.1,Identifies, selects, and develops risk mitigation activities for disruptions,PC-05,Hosted-model fallback contract,Daily canary job + contract,audits/residency-attestations.md,2026-11-13,90d,Vendor Liaison
SOC-2,CC9.2,Assesses and manages risks associated with vendors,PC-05,Hosted-model fallback contract,Register entry,mod-107/registers/vendor-register.md,2026-11-01,90d,Vendor Liaison
ISO-27001,A.17.1.2,Implementing information security continuity,ISCP-01,Information System Contingency Plan,Document + role training,plan/chatcat-iscp.md,2026-11-13,90d,AI SRE Director
ISO-27001,A.17.1.3,Verify information security continuity,EXE-PGM-01,DR Exercise Programme,Drill cadence,programmes/history/,continuous,per tier,AI SRE Director
GDPR,Art 32,Security of processing,RC-11,In-scope backup,KMS policy + bucket policy,audits/residency-attestations.md,2026-11-13,90d,AI Security Lead
GDPR,Art 32,Security of processing,RC-12,Audited cross-scope access,Access log,audits/access-logs/,continuous,continuous,AI Security Lead
AI-SPECIFIC,AI-WEIGHT-ATT,Weight attestation on model pin,PC-04,Weight-manifest attestation,Sigstore attestation,audits/weight-attestations/,2026-11-13,per-deploy,ML Platform lead
AI-SPECIFIC,AI-JUDGE-SNAP,Judge-pipeline snapshotting,PC-07,Judge-pipeline daily snapshot,Scheduled job,audits/judge-snapshots/,continuous,daily,ML Platform lead
AI-SPECIFIC,AI-PROMPT-CACHE,Stable-prefix hash CI check,PC-06,Stable-prefix-hash CI check,CI job on prompt repo,chatcat-prompts/.github/workflows/stable-prefix.yml,continuous,per-PR,AI Engineering lead
AI-SPECIFIC,AI-PROVIDER-FB,Fallback provider contract + daily canary,PC-05,Hosted-model fallback contract,Canary job + contract,scripts/vendor-canary.sh + audits/residency-attestations.md,2026-11-13,daily,Vendor Liaison
AI-SPECIFIC,AI-RESIDENCY,Scope-bound serving and state,RES-FO-01 + RC-02..07,Residency controls,Router + KMS + ledger,controls/residency-controls.md,2026-11-13,per-drill,AI SRE Director
```

The matrix is intentionally in CSV. Auditors read it in Excel; the shape is stable,
the columns are fixed, and the matrix can be diffed between pack versions.

### 2.4 Exercise history (`03-exercise-history.md`)

```markdown
# Exercise history (2025-11-01 — 2026-11-01)

Twelve months of drills. Linked to `programmes/history/` where the full scribe notes
and rubric scores live. The rubric scores below are the headline pass/fail per drill.

## Tabletop (monthly, 12 runs)

| Date | Scenario | IC (role) | Rubric score | Action items | Closed |
|------|----------|-----------|--------------|---------------|--------|
| 2025-12-04 | s-regional-outage-eu | oncall-ic-alice | 17 / 20 | 3 | 3/3 |
| 2026-01-08 | s-provider-primary-degraded | oncall-ic-bob | 15 / 20 | 4 | 4/4 |
| 2026-02-05 | s-retrieval-index-corruption | oncall-ic-chandra | 18 / 20 | 2 | 2/2 |
| 2026-03-05 | s-weights-attestation-fail | oncall-ic-alice | 14 / 20 | 5 | 4/5 (1 in-flight CHG-...) |
| 2026-04-02 | s-provider-silent-version-bump | oncall-ic-dara | 16 / 20 | 3 | 3/3 |
| 2026-05-07 | s-both-regions-impaired | oncall-ic-bob | 13 / 20 (fail) | 6 | 6/6 (one re-drill scheduled) |
| 2026-05-21 | s-both-regions-impaired (re-drill) | oncall-ic-bob | 17 / 20 | 1 | 1/1 |
| 2026-06-04 | s-agent-state-loss-partial | oncall-ic-ethan | 15 / 20 | 4 | 3/4 (1 in-flight) |
| 2026-07-02 | s-residency-cross-scope-attempted | oncall-ic-chandra | 18 / 20 | 2 | 2/2 |
| 2026-08-06 | s-judge-pipeline-outage | oncall-ic-alice | 16 / 20 | 3 | 3/3 |
| 2026-09-03 | s-cost-blowup-during-failover | oncall-ic-dara | 15 / 20 | 4 | 4/4 |
| 2026-10-01 | s-cyber-incident-triggers-isc | oncall-ic-bob | 17 / 20 | 3 | 3/3 |

Note: 2026-05 drill failed and was re-drilled 2026-05-21. That cycle produced the
most action items (6), five of which hardened the plan; one was rejected (see history
file for rationale).

## Structured walkthrough (quarterly, 4 runs)

| Date | Focus | Attendees | Result | Action items |
|------|-------|-----------|--------|---------------|
| 2025-12-15 | ISCP + playbook currency | 8 (all roles present) | pass | 2 (closed) |
| 2026-03-18 | Residency + SRP (new) | 9 | pass | 3 (closed) |
| 2026-06-17 | Full plan set | 10 | pass | 2 (closed) |
| 2026-09-16 | Full plan set + apac-sg scope prep | 10 | pass | 4 (3 closed, 1 in-flight) |

## Simulation (quarterly, 4 runs, rotating component)

| Date | Component | RTO(serving) met | RTO(quality) met | Surprises |
|------|-----------|------------------|------------------|-----------|
| 2026-01-22 | Retrieval primary drop | yes (18 min) | yes (84 min) | 1 (runbook step missing) |
| 2026-04-23 | Session store primary drop | yes (12 min) | n/a | 0 |
| 2026-07-23 | Judge worker pool kill | n/a | yes (48 min) | 1 (snapshot older than policy) |
| 2026-10-22 | Provider 429 forced | yes (14 min) | yes (36 min) | 0 |

## Parallel test (semi-annual, 2 runs)

| Date | Shadow percentage peak | Divergence rate | Capacity result | Verdict |
|------|------------------------|-----------------|------------------|---------|
| 2026-02-20 | 100% us-general anonymous | 3.4% | within headroom | pass |
| 2026-08-20 | 100% eu-gdpr anonymous | 4.1% | within headroom | pass |

## Full-interruption (annual, 1 run)

| Date | Duration | RTO(serving) | RTO(quality) | Return-to-primary | Customer complaints | Verdict |
|------|----------|---------------|---------------|-------------------|----------------------|---------|
| 2026-06-15 | 2h announced | 22 min | 68 min | within window | 0 | pass |

Attached to each row is a link to the `programmes/history/` file; redacted scribe
notes available on request to the audit contact.
```

### 2.5 Attestation index (`04-attestation-index.md`)

```markdown
# Attestation index

## Provider attestations
- `audits/residency-attestations.md#provider-a-eu` — provider-A EU endpoint, last
  reviewed 2026-11-01, next review 2027-02-01, owner Vendor Liaison.
- `audits/residency-attestations.md#provider-a-us` — provider-A US endpoint, last
  reviewed 2026-11-01, next review 2027-02-01, owner Vendor Liaison.
- `audits/residency-attestations.md#provider-b-eu` — fallback provider EU, last
  reviewed 2026-11-01.
- `audits/residency-attestations.md#provider-b-us` — fallback provider US, last
  reviewed 2026-11-01.

## Weight attestations
- `audits/weight-attestations/` — Sigstore attestations per in-house weight version.
  Last attested: `chatcat-rerank-v4.2` on `2026-11-10`.
- For hosted models: the provider contract pins (e.g., `claude-3-5-haiku-20241022`)
  are recorded in `audits/model-pin-history.md` alongside the dated change tickets
  that moved the pin.

## Judge-pipeline snapshots
- `audits/judge-snapshots/` — daily snapshots retained 30 days (in-flight extension
  to 90 days per cover-letter disclosure). Weekly diff reports in
  `audits/judge-snapshots/weekly-diff-*.md`.

## In-scope backup bucket policies
- `audits/bucket-policies/` — one file per scope + component. Policies deny
  cross-scope replication at the KMS and bucket layers.

## Cross-scope access logs
- `audits/access-logs/` — append-only. Exception-path jump-host access only;
  review cadence is monthly by AI Security Lead.

## Vendor-risk register
- `mod-107/registers/vendor-register.md` — contains the register this pack's
  exposures refer to.

## Change register
- `audits/change-register.md` — triggered-review events (model-version bumps,
  provider swaps, region additions) and the review tickets they opened.
```

### 2.6 AI-specific addendum (`05-ai-specific-addendum.md`)

```markdown
# AI-specific controls addendum

This addendum is an AI-sector overlay on the standard BCP audit. It names the
controls that generic frameworks do not yet demand but that an AI-aware audit
expects.

## AI-WEIGHT-ATT — Weight attestation on model pin

**Why.** A hosted-model provider silently updating a model behind the same identifier
is an observed class of incident (documented in the vendor-risk register under
"silent version drift"). An audited AI system pins to a specific dated model version
or in-house weight SHA, attests on each deploy, and re-attests on a schedule.

**How.** PC-04 in `controls/chatcat-preventive-controls.md`.
- Hosted: the model identifier in the request body (e.g., `claude-3-5-haiku-20241022`)
  is asserted by an admission control; the admission rejects requests without an
  attested pin.
- In-house: Sigstore signatures over the weight SHA. The signature set is enumerated
  in `audits/weight-attestations/`.

**Evidence.** Last attestation date per pin; the attestation signature; the pin
history over the audit window.

## AI-JUDGE-SNAP — Judge-pipeline snapshotting

**Why.** The quality-recovery contract (RTO(quality)) depends on having a
reproducible "baseline" to compare against post-recovery. Without a snapshot, the
on-call cannot declare quality recovered; they can only declare "quality looks right
to me".

**How.** PC-07. Daily snapshot of the judge-pipeline state (sample set, judge model
pin, thresholds). Weekly diff review highlights drift.

**Evidence.** Snapshot catalogue; weekly diff reports; the last diff reviewed by the
ML Platform lead.

## AI-PROMPT-CACHE — Stable-prefix hash CI check

**Why.** Prompt-template changes that invalidate the provider's prompt cache cause
observable TTFT regressions. The control is cheap; the omission is systemic. See
the worked post-mortem in `mod-108/exercise-03` for the fault pattern.

**How.** PC-06. CI job on `chatcat-prompts` computes the stable-prefix hash before
and after each PR. A move requires a `cache-warmup-plan` label and reviewer
sign-off.

**Evidence.** CI history on the prompt repository; sample PRs that were blocked.

## AI-PROVIDER-FB — Fallback provider contract + daily canary

**Why.** Hosted-model providers are external dependencies with their own outage
history; the only way to carry traffic through a provider outage is a second
provider, pre-attested and daily-exercised.

**How.** PC-05. Fallback-provider contract in place; daily `scripts/vendor-canary.sh`
runs a representative workload against the fallback endpoint; failure pages SRE.

**Evidence.** Last 90 days of canary job runs; the contract clause committing to
a fallback-provider pin.

## AI-RESIDENCY — Scope-bound serving and state

**Why.** Residency-aware workloads cannot silently cross scopes. Exercise 04
describes the controls in detail; this addendum links to them for the audit reader.

**How.** RES-FO-01 and RC-01 through RC-13 in `mod-113/exercise-04/controls/
residency-controls.md`.

**Evidence.** Router config, KMS policies, scope-tag telemetry, quarterly drill
log (August 2026 scenario).

## Mapping to AI-specific frameworks (when applicable)

| If the audit framework is | Then also reference |
|---------------------------|---------------------|
| EU AI Act (high-risk provider) | Articles 10 (data), 15 (accuracy / robustness / cybersecurity), 72 (post-market monitoring). This pack's dual-RTO + judge-pipeline + weight-attestation set maps directly to Article 15 and 72. |
| DORA (financial services) | ICT risk management framework (Art. 6-16) and ICT business continuity (Art. 11). The exercise programme in `mod-113/exercise-05` maps to Art. 11.6. |
| NIST AI RMF (voluntary) | GOVERN 1, 4, 5, 6; MANAGE 1, 4; MEASURE 1. |
| HIPAA Security Rule | §164.308(a)(7) contingency plan standard and its implementation specifications (data backup, disaster recovery, emergency mode operation, testing and revision). |

The pack's structure is framework-neutral; the §1 cover letter declares the primary
framework, and the matrix in §2.3 maps the control IDs explicitly.
```

### 2.7 Sampling guidance (`06-sampling-guidance.md`)

```markdown
# Sampling guidance for auditors

If the audit methodology calls for sampling rather than exhaustive review, the
suggestions below are the shortest path to coverage. The samples are intended to
be representative, not minimising.

## Sample A — Exercise history
Pick 3 of the 12 monthly tabletops in `03-exercise-history.md` and request:
- the full scribe notes,
- the rubric score sheet,
- the action-item closure records.

Also request 1 of the 4 simulations and the 1 full-interruption.

## Sample B — Controls
Pick 3 of the 16 rows in `01-control-matrix.csv` and request the evidence file
dated within the review cadence.

Suggested pick: at least one row from each of AI-SPECIFIC, ISO-22301, and GDPR to
cover the overlay. The AI-SPECIFIC rows are the most likely to surface a weak-
evidence control.

## Sample C — Triggered reviews
Pick the latest triggered-review event from `audits/change-register.md` and
request the triggered-review ticket + the plan revision it produced.

Suggested pick: most recent model-version bump (common trigger).

## Sample D — Cross-scope access (if residency is in scope)
Pick the most recent 10 entries in `audits/access-logs/cross-scope/` and request
the corresponding tickets, step-up MFA records, and post-access review.

## Sample E — Weight attestation
Pick the most recent in-house weight pin and verify the Sigstore attestation
signature and the pin ticket.
```

### 2.8 Pack change log (`07-change-log.md`)

```markdown
# Pack change log

| Date | Author | Change |
|------|--------|--------|
| 2026-11-13 | AI SRE Director | Initial pack version 2026-Q4 |
| 2026-08-13 | AI SRE Director | Pack 2026-Q3 — added apac-sg scope prep note; refreshed exercise history |
| 2026-05-13 | AI SRE Director | Pack 2026-Q2 — added judge-pipeline snapshot retention to in-flight items |
| 2026-02-13 | AI SRE Director | Pack 2026-Q1 — first full pack after annual full-interruption 2026-06 — correction to history: full-interruption was 2026-06-15, pack publication is quarterly |
```

(Correction: in the fabricated history the quarterly pack is produced on the 13th of
each cycle-ending month; the pack change log here is illustrative.)

## 3. Validation steps

1. **Cold-reader walk.** Hand the pack to an engineer who has no context. Within 30
   minutes they should be able to answer, with pointers: "Do the plans exist? Are
   they current? Are they exercised? Are the AI-specific controls in place?"
2. **Pointer integrity.** Every filename referenced in the pack resolves. A broken
   pointer is a defect.
3. **Freshness check.** Every "last reviewed" date in `02-plan-register.md` and the
   control matrix is within the declared cadence. A date beyond cadence is either
   a disclosed gap (in the cover letter) or a defect.
4. **Framework coverage.** Every audit control ID named in the cover letter's
   declared framework has at least one row in `01-control-matrix.csv`.
5. **Matrix diff vs. prior pack.** The pack diff against the prior version should
   be small and explicable: dates moved forward, new controls added for new scopes,
   closed action items.
6. **Sampling can succeed.** For each sample in §2.7, the referenced artifacts are
   actually present and complete.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Pack structure *(hard gate)* | Unstructured dump | Named files, inconsistent | Fixed 8-file structure, stable across versions |
| 2 | Cover letter *(hard gate)* | Missing | Narrative but no in-flight disclosure | Full: purpose, scope, reading order, in-flight items, POC |
| 3 | Plan register *(hard gate)* | Missing | Plans listed | Plans with version, owner, last review, next review |
| 4 | Control matrix *(hard gate)* | Missing | Narrative only | CSV with audit-control-to-internal-control-to-evidence mapping |
| 5 | Exercise history | Missing | List | Per-tier, with scores + action-item closure |
| 6 | Attestation index | Missing | Listed | Dated, owned, with review cadence |
| 7 | AI-specific addendum *(hard gate)* | None | Mentioned | ≥ 4 AI-specific controls with why/how/evidence |
| 8 | Sampling guidance | None | Narrative | Specific samples with suggested picks + what to request |
| 9 | Framework coverage | One only | Named | Primary framework + AI-sector overlay with explicit mapping |
| 10 | Pack change log + diff story | None | Change log | Change log + "what diffs between versions" explanation |

## 5. Common mistakes

- **Narrative instead of evidence.** "We exercise the plan quarterly" as the only
  evidence is a finding waiting to happen. The dated history table is the evidence.
- **Pack as dump.** Including every file in `mod-113/` as the "pack" overwhelms the
  auditor and makes gaps hard to see. Curation is the point.
- **Framework-shoehorning.** Trying to map every AI-specific control to a generic
  ISO 22301 control ID stretches language and leaves AI-sector auditors unconvinced.
  The AI-specific addendum is the right place for them.
- **Hidden gaps.** Leaving out the fact that judge-pipeline retention is below
  policy produces a worse audit outcome than disclosing it in the cover letter with
  a closure date and owner.
- **Attestation drift.** Attestations with "last reviewed 2023-04" are worse than
  no attestations because they claim current evidence that is stale.
- **No sampling guidance.** Auditors will sample anyway; without guidance they may
  pull samples that happen to cover the strongest areas while missing the weak
  ones, producing a false sense of confidence. Explicit guidance is both a kindness
  and a form of transparency.
- **Non-versioned pack.** Without a versioned pack and a diff story, the auditor
  cannot see progress between cycles. The quarterly cadence + change log makes the
  pack an ongoing artifact, not a one-time deliverable.
- **Weight-attestation-as-afterthought.** Weight attestation is the control that
  distinguishes a modern AI BCP posture from a web-service BCP posture; folding it
  into "supply-chain generic" undersells it.

## 6. References

- NIST SP 800-34 Rev. 1 §3.7 — plan maintenance; this pack is the evidence that
  maintenance happened.
- ISO 22301:2019, *Security and resilience — Business continuity management systems
  — Requirements* — the clauses 8 and 9 the control matrix maps against.
- AICPA SOC 2 Trust Services Criteria — the CC9 common-criteria (business
  continuity) the matrix also maps to.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the GOVERN and MANAGE functions grounds the AI-specific addendum's control set; in particular GOVERN 1 ("policies, processes, procedures, and practices across the organization related to AI risks are in place") maps to the plan-register freshness and MEASURE 2 ("AI systems are evaluated for trustworthy characteristics") maps to the judge-pipeline snapshot control.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML supply chain` grounds AI-WEIGHT-ATT and AI-PROVIDER-FB; `ML10: model poisoning` grounds the retrieval-index snapshot pairing cited in the matrix.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "Resource development" and "Supply chain compromise" technique classes justify why AI-WEIGHT-ATT and AI-PROVIDER-FB are evidence-first controls in the audit, not qualitative narrative.
- EU AI Act (Regulation 2024/1689) — Articles 15 and 72, referenced for high-risk AI provider scope mapping in `05-ai-specific-addendum.md`.
- Digital Operational Resilience Act (DORA, Regulation 2022/2554) — Articles 6-16 and Article 11, referenced for financial-services scope mapping.
- `mod-107` vendor-and-provider-risk register — the primary source for provider-related evidence in §2.5.
- `mod-108/exercise-06` legal-compliance handoff — the handoff pattern the cross-scope access log (§2.5) and November tabletop scenario (`s-cyber-incident-triggers-isc`) exercise.
- `mod-110/exercise-03` compliance-telemetry hooks — the telemetry that feeds `01-control-matrix.csv` for continuous-monitoring rows.
- `mod-113/exercise-01` through `exercise-05` — the plans and evidence sources this pack curates.
