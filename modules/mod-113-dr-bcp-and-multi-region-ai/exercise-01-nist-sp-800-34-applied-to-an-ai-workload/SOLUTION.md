# Exercise 01 — NIST SP 800-34 Applied to an AI Workload — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-01-nist-sp-800-34-applied-to-an-ai-workload.md`.

> **Scope note.** The paired exercise prompt asks the learner to take a specific AI workload and walk it through the NIST SP 800-34 Rev. 1 contingency-planning process, producing the artifacts the standard names: a BIA, preventive-control inventory, recovery strategy, and an Information System Contingency Plan (ISCP) with the AI-specific additions (weights, caches, indices, judge pipeline). The reference solution works the process end-to-end against the same fabricated `chatcat` RAG-chat surface used across this module family. A learner who chose a different illustrative product should be scored against the rubric in §4, not against `chatcat` specifically.

## 1. Solution overview

NIST SP 800-34 Rev. 1 defines a seven-step contingency-planning process and an Information System Contingency Plan (ISCP) template. The process was written for general federal information systems, and most of it transfers cleanly to AI workloads. What **does not** transfer cleanly, and where learners most often shortcut, is the step where contingency assumptions meet the AI-specific stack:

1. **Model weights and indices are information-system components under SP 800-34**, with their own custody, their own backup frequency, and their own recovery criteria. A weights snapshot has an RPO; a retrieval index has an RPO; neither is captured by "database backups" on the general-purpose BIA template.
2. **"Recovery" is a two-stage concept**, not one. Serving-path recovery (route is up, requests get responses) and quality recovery (responses are grounded, judge metrics return to baseline) have different RTOs and different owners. The ISCP must name both.
3. **Hosted-model and provider dependencies are external dependencies**, which SP 800-34 §3.4.2 acknowledges but does not deeply expand on. They require the ISCP to cross-reference the vendor-risk register from `mod-107`.
4. **"Preventive controls" in SP 800-34 §3.3** include the controls the organisation can mechanically enforce (replication, snapshots, failover automation) *and* the AI-specific ones (weight-pinning, judge-pipeline snapshots, cache-warm-path documentation).

The deliverable set is five files committed to the contingency-planning repository, organised exactly as the standard expects the plan to be organised:

- `policy/isc-policy-chatcat.md` — Step 1 output (ISC policy).
- `bia/chatcat-bia.md` — Step 2 output (BIA).
- `controls/chatcat-preventive-controls.md` — Step 3 output (preventive-control inventory).
- `strategy/chatcat-recovery-strategy.md` — Step 4 output (recovery strategy).
- `plan/chatcat-iscp.md` — Step 5 output (the ISCP itself).

The two remaining SP 800-34 steps (test/train/exercise, maintenance) are owned by exercise 05 (rehearsal schedule) and exercise 06 (audit pack), and the plan deliberately cross-references them rather than inlining.

### 1.1 The seven steps of SP 800-34 Rev. 1 §3

| Step | Output | Where in this exercise | Where in module |
|------|--------|------------------------|-----------------|
| 1. Develop the contingency planning policy | Policy statement | `policy/isc-policy-chatcat.md` | §2.1 |
| 2. Conduct the business impact analysis (BIA) | BIA document | `bia/chatcat-bia.md` | §2.2 |
| 3. Identify preventive controls | Control inventory | `controls/chatcat-preventive-controls.md` | §2.3 |
| 4. Create contingency strategies | Strategy document | `strategy/chatcat-recovery-strategy.md` | §2.4 |
| 5. Develop information system contingency plan | ISCP | `plan/chatcat-iscp.md` | §2.5 |
| 6. Ensure plan testing, training, and exercises | Test / exercise programme | Cross-ref `exercise-05` | §2.5.7 |
| 7. Ensure plan maintenance | Review / update cadence | Cross-ref `exercise-06` and plan §9 | §2.5.8 |

### 1.2 The AI-specific additions to each step

| SP 800-34 step | Generic deliverable | AI-specific additions this solution adds |
|----------------|---------------------|------------------------------------------|
| Policy | Scope, roles, authorities | Model-weight custody, judge-pipeline ownership, provider-risk register linkage |
| BIA | MTD / RTO / RPO per critical system function | Separate serving-recovery and quality-recovery objectives; index RPO; weight RPO |
| Preventive controls | Backups, replication, UPS, environmental controls | Weight-pinning; stable-prefix hash CI check; judge-pipeline snapshotting; retrieval-index replication; hosted-model fallback contract |
| Recovery strategy | Alternate site, backup strategy | Active-active vs. warm-standby for serving; replicated-vs-reindexed decision for retrieval; weight-pinning cold-standby for the model |
| ISCP | Activation, notification, recovery, reconstitution | Quality reconstitution (judge validation), weight-provenance re-attestation, retrieval-freshness validation |
| Test / train / exercise | Tabletop, functional, full-scale | Adds "chaos drill" tier (dropped index shard, provider 429 flood, weight-pin break) — owned by exercise 05 |
| Maintenance | Annual review | Triggered reviews: model-version bump, provider swap, region addition — owned by exercise 06 |

## 2. Implementation — worked answer

### 2.1 Step 1 — ISC Policy (`policy/isc-policy-chatcat.md`)

```markdown
# chatcat Information System Contingency Planning Policy

**Owner:** VP, AI Platform. **Approver:** CISO.
**Policy version:** 2026-11-13. **Review cadence:** annual + triggered.

## Purpose

This policy establishes the minimum requirements for maintaining operational continuity of the
`chatcat` information system (hereafter "the system") in the event of a disruption. It is scoped
to the serving, retrieval, judge-pipeline, and provider-dependency components of the system.
It extends — it does not replace — the organisation's enterprise business-continuity policy.

## Scope

- In scope: `chatcat-serving`, `chatcat-retrieval`, `chatcat-eval-judge`, `chatcat-telemetry`.
- Out of scope: corporate IT services (handled by enterprise BCP), the research training
  cluster (handled by R&D DR policy).

## Roles and responsibilities

- **Contingency-Plan Coordinator (CPC):** AI SRE Director. Owns the ISCP document, the test
  schedule, and the maintenance log.
- **System Owner:** VP, AI Platform. Signs off on recovery-strategy choices and RTO/RPO
  commitments.
- **Information System Security Officer (ISSO):** AI Security Lead. Owns control inventory
  and reviews ISCP changes that touch security controls.
- **Oncall IC:** rotating AI SRE on call. Activates the ISCP; authority delegated per plan §4.
- **Vendor Liaison:** owns the hosted-model provider relationship (cross-ref `mod-107`).

## Authorities

- Plan activation authority: the oncall IC on confirmed regional outage, confirmed
  provider outage affecting primary, or confirmed data-corruption event affecting the
  retrieval index or weight store. Severity escalation per `mod-108` chapter 02.
- Spend authority during active recovery: pre-approved at $Y per hour via the CPC;
  anything above escalates to System Owner.

## Related plans

- Enterprise BCP: `enterprise-continuity/bcp-v2026.md`.
- Cyber Incident Response: `ir/cirp-v2026.md`.
- Occupant Emergency: `facilities/occupant-emergency.md`.
- Vendor risk register: `mod-107/registers/vendor-register.md`.
- Related exercises in this module: `mod-113/exercise-05` (test programme),
  `mod-113/exercise-06` (audit evidence).

## Training and testing

The system's ISCP is tested on the schedule authored in `mod-113/exercise-05`:
quarterly tabletop; semi-annual functional (one component); annual full failover.
All personnel named in the ISCP are trained annually on their recovery roles.

## Maintenance

The ISCP is reviewed annually. Triggered reviews are required on any of:
model-version bump (hosted or in-house), provider swap, region addition, retrieval-index
architecture change, judge-pipeline change.

## Policy exceptions

Any deviation from this policy requires written approval from the CISO and the System Owner,
logged in `policy/exceptions.md` with an expiration date.
```

**Why this shape.** SP 800-34 §3.1 requires the policy to establish *authority* to plan, *scope*,
*roles*, and *maintenance cadence*. The AI-specific additions are: naming the vendor liaison as
a role (because hosted-model dependencies are first-class), making model-version bump a
triggered-review event (because provider version drift is a silent risk per `mod-107`), and
cross-referencing the test programme instead of inlining it (keeps the policy stable while
the schedule rotates).

### 2.2 Step 2 — Business Impact Analysis (`bia/chatcat-bia.md`)

The BIA is the step where learners most often undercount AI-specific components. SP 800-34
§3.2 requires three outputs: (a) determine mission / business processes and recovery
criticality, (b) identify resource requirements, (c) identify recovery priorities for system
resources.

```markdown
# chatcat Business Impact Analysis

**BIA version:** 2026-11-13. **Prepared by:** AI SRE + Product Analytics.
**Approved by:** VP, AI Platform.

## 1. Mission / business processes supported

| Process ID | Process | Owning function | User-visible channel |
|------------|---------|-----------------|----------------------|
| P-01 | Interactive support chat | Customer Experience | web + mobile SDK |
| P-02 | In-product content summarisation | Product | web SDK |
| P-03 | Internal knowledge-base Q&A | IT / HR | intranet |

## 2. Recovery criticality

| Process | Impact of 1h downtime | Impact of 4h downtime | Impact of 24h downtime | FIPS 199 category |
|---------|-----------------------|-----------------------|------------------------|-------------------|
| P-01 | Moderate (deflection fallback to human agents possible, revenue loss bounded) | High (support-queue saturation, contractual SLA breach) | High (reputational + contractual) | Confidentiality: moderate; Integrity: high; Availability: high |
| P-02 | Low (feature degrades to no-summary) | Moderate (user complaints) | Moderate (feature off) | Availability: moderate |
| P-03 | Low | Low | Moderate | Availability: low |

## 3. Impact metrics

Per SP 800-34 §3.2.1, three metrics are established per process. For AI workloads this
template separates **serving recovery** (route is up) from **quality recovery** (judge
metrics are back at baseline), because "up and wrong" is a different failure mode than "down".

| Process | MTD | RTO (serving) | RTO (quality) | RPO (session state) | RPO (retrieval index) | RPO (weights) | WRT |
|---------|-----|----------------|----------------|----------------------|------------------------|-----------------|-----|
| P-01 | 4h | 30min | 2h | 5min | 1h | 24h | 15min |
| P-02 | 24h | 2h | 8h | 1h | 4h | 24h | 30min |
| P-03 | 72h | 4h | 24h | 4h | 24h | 7d | 1h |

Notes:
- **RTO (serving)** is the maximum tolerable interval before the request path accepts traffic
  from somewhere (primary or fallback) and returns responses.
- **RTO (quality)** is the maximum tolerable interval before the judge pipeline confirms
  groundedness, refusal-rate, and structured-output-validity have returned to within 2 sigma
  of baseline. This is the metric that matters for a customer-visible AI surface.
- **RPO (retrieval index)** is the maximum tolerable age of the last replicated snapshot of
  the index. A stale index does not cause a request to fail; it causes answers to be subtly
  wrong. The judge pipeline is the signal.
- **RPO (weights)** is the maximum tolerable age of the attested weight manifest in the cold
  store. The hosted-model path pins to a specific provider model version; the in-house path
  pins to a specific weight-SHA. A fresh pin is required for a clean cold-start.
- **WRT** is the recovery of staff workload after the system is up (per SP 800-34 §3.2.1).

## 4. Resource requirements

Per SP 800-34 §3.2.2:

### 4.1 Component inventory

| Component | Owner | Criticality | Replication strategy | External dependency |
|-----------|-------|-------------|----------------------|---------------------|
| `chatcat-serving` pods | AI SRE | High | Active-active across 2 regions | Hosted model API |
| `chatcat-retrieval` index | Data Platform | High | Snapshot every 1h; replicated | Vector DB provider |
| `chatcat-telemetry` sidecar | AI SRE | Moderate | Stateless | — |
| `chatcat-eval-judge` pipeline | ML Platform | High (for quality RTO) | Snapshot daily; weights pinned | Judge-model provider |
| `chatcat-prompts` repository | AI Engineering | High | Git-replicated | — |
| Hosted model (primary) | Vendor | External | Fallback contract | **Vendor**: see register |
| Hosted model (fallback) | Vendor | External | Standby, pre-attested | **Vendor**: see register |
| Session store (Redis) | Platform | Moderate | Replicated, 5min RPO | — |

### 4.2 Personnel

- IC pool: 6 engineers, rotation. On-call primary + secondary at all times.
- Vendor liaison: single owner + one named backup.
- Eval-judge subject-matter expert: ML Platform lead.

## 5. Recovery priority order

Per SP 800-34 §3.2.3, the order below is the sequence in which components must be recovered
to minimise total impact. The order is driven by the **dependency graph**, not alphabetical:

1. **Session store (Redis).** If lost, in-flight conversations are lost; downstream has no
   way to resume without re-prompting.
2. **Hosted model path (primary or fallback).** The route cannot respond without a model.
3. **`chatcat-serving`.** The request handler.
4. **`chatcat-retrieval` index (serving-ready replica).** Required for *grounded* answers;
   the serving path can degrade to no-RAG for a bounded window.
5. **`chatcat-prompts`.** Needed for correct prompting. Git clone is fast; this is late in
   the list because a cached prompt bundle can carry the service for a bounded window.
6. **`chatcat-telemetry`.** Required to *know* the service is healthy; blocks declaring
   recovery.
7. **`chatcat-eval-judge`.** Required to declare quality recovery; this is where
   RTO(quality) is validated, not where RTO(serving) is.

The recovery-strategy document (step 4) encodes this priority into automation; the ISCP
(step 5) encodes it into the operator-facing checklist.
```

**Why this shape.** The dual-RTO treatment is the single most important content gap in
most learner submissions. If RTO is a single number, the plan passes "the route is up"
and papers over regressions in groundedness. Index RPO and weight RPO are the two
AI-specific resource lines that generic BIA templates do not provide. The dependency-ordered
recovery priority is how SP 800-34 §3.2.3 wants priorities expressed; the rationale on
each line is what distinguishes a real plan from a reorder of the inventory.

### 2.3 Step 3 — Preventive Controls (`controls/chatcat-preventive-controls.md`)

```markdown
# chatcat Preventive Controls

**Controls version:** 2026-11-13. **Mapped to:** NIST SP 800-53 Rev. 5 CP family.

## 1. Environmental controls

- **UPS + generator for primary region.** CP-8 (Telecommunications Services), CP-9
  (System Backup). Evidence: facility certificates.
- **Fire suppression, access control.** Enterprise facility BCP; this plan references.

## 2. System-level preventive controls

| Control ID | Name | Covers | Evidence |
|------------|------|--------|----------|
| PC-01 | Active-active serving across 2 regions | Serving RTO | Deployment manifests; DNS failover config |
| PC-02 | Session-store cross-region replication (5min RPO) | Session RPO | Redis Enterprise active-active config |
| PC-03 | Retrieval-index snapshot every 1h + cross-region copy | Index RPO | Snapshot job logs + object-store bucket policy |
| PC-04 | Weight-manifest attestation on pin | Weights RPO + integrity | Sigstore attestation per weight version |
| PC-05 | Hosted-model fallback contract (primary + fallback, pre-flighted daily) | Provider dependency | Daily canary job: `scripts/vendor-canary.sh` |
| PC-06 | Stable-prefix-hash CI check on prompt repository | Prevents cache-cliff regressions | `chatcat-prompts/.github/workflows/stable-prefix.yml` |
| PC-07 | Judge-pipeline daily snapshot + weekly diff review | Quality-recovery baseline | Snapshot bucket + `eval-judge-weekly-diff` job |
| PC-08 | Immutable image + weight image digests | Supply-chain integrity | Image-digest allowlist in admission controller |
| PC-09 | Runbook library versioned and signed | Response readiness | `mod-108` runbook repo + CODEOWNERS |
| PC-10 | Chaos exercise programme | Verifies controls | `mod-113/exercise-05` schedule |

## 3. AI-specific preventive controls — rationale

Three controls do not appear on a generic preventive-controls list and warrant explicit rationale:

- **PC-04 (weight attestation).** A hosted-model provider silently updating a model behind
  the same name is a documented class of incident in the vendor-risk register. Pinning the
  provider to a specific dated version (e.g., `claude-3-5-haiku-20241022`) and re-attesting
  on every deploy is the control. For in-house weights, the attestation is a Sigstore signature
  over the weight SHA.
- **PC-06 (stable-prefix-hash check).** Prompt-template changes that invalidate the provider's
  prompt cache have caused observable TTFT regressions (see the worked post-mortem in
  `mod-108/exercise-03`). The control is a CI check that computes the hash of the stable
  prefix (everything up to the first variable token) before and after the PR, fails on silent
  moves, and requires a `cache-warmup-plan` label to proceed on intentional moves.
- **PC-07 (judge snapshot).** Judge-pipeline recovery requires a known-good snapshot to
  compare against post-recovery. Without it, "quality recovered" is unprovable.
```

### 2.4 Step 4 — Recovery Strategy (`strategy/chatcat-recovery-strategy.md`)

```markdown
# chatcat Recovery Strategy

**Strategy version:** 2026-11-13.

## 1. Chosen strategy summary

| Component | Strategy | Rationale |
|-----------|----------|-----------|
| Serving (`chatcat-serving`) | **Active-active** across region-A and region-B | RTO(serving) 30min cannot be met with warm-standby given the fleet size; active-active halves blast radius |
| Retrieval index | **Hot replica** in region-B, continuously replicated, plus 1h snapshot archive | Replica supports failover; snapshots support corruption recovery (replica alone does not protect against logical corruption) |
| Session store | **Active-active** Redis Enterprise | Session continuity across region failure |
| Hosted model path | **Fallback contract** with second provider, daily-pre-flighted | A 100% failover of a hosted-model path is the vendor's problem until the vendor fails; we need a different vendor on hand |
| Judge pipeline | **Warm-standby** in region-B; daily snapshot | RTO(quality) is 2h, warm-standby can be stood up within window |
| Prompt repository | **Git mirror** in region-B; local cache bundle on serving pods | Git clone is fast; the cache bundle is a belt-and-braces measure for full regional outage |
| Weights (in-house, where applicable) | **Cold store** + attestation; re-pin on recovery | Weight files are large and rarely change; cold store is enough |
| Weights (hosted) | **Version pin** + fallback pin | The vendor owns weight recovery; we own the pin |

## 2. Alternate-site considerations (SP 800-34 §3.4.2)

- **Primary site:** cloud region-A.
- **Alternate site:** cloud region-B (same provider, different region). This is a **hot site**
  for serving + session, a **warm site** for the judge pipeline.
- **Site diversity:** regions share the cloud provider. The provider outage scenario is
  mitigated by a **tertiary site** — a second cloud provider hosting only the fallback
  model path and a cold weight copy. This does not match the full chatcat stack; it covers
  the thin "degraded, hosted-model-only, no-RAG" mode as a last-resort continuity path.

## 3. Equipment / service / resource replacement

- Compute: pre-provisioned capacity in region-B (not "elastic on demand" alone; capacity
  reserved to the RTO).
- GPUs: reserved pool in region-B for the judge pipeline; hosted-model path does not
  require local GPU.
- Human: on-call rotation covers both regions.

## 4. Cost model

| Strategy | Estimated incremental cost vs. single-region | Covers |
|----------|---------------------------------------------|--------|
| Active-active serving | ~+80% of single-region serving | Regional outage; majority of fault domains |
| Hot replica retrieval + snapshots | ~+50% of index baseline | Regional + logical corruption |
| Tertiary site (hosted fallback) | ~+5% baseline | Provider outage |
| Total estimated DR overhead | ~+35% of baseline total | — |

Numbers are illustrative; actual commitment goes in the vendor-approved budget attached
to the ISCP.
```

### 2.5 Step 5 — Information System Contingency Plan (`plan/chatcat-iscp.md`)

The ISCP is the document the on-call reaches for. Its shape follows SP 800-34 Appendix A
(sections 1–9), with AI-specific additions inline:

```markdown
# chatcat Information System Contingency Plan

**ISCP version:** 2026-11-13. **Classification:** Internal. **Approver:** VP, AI Platform.

## Section 1 — Introduction

### 1.1 Background and purpose
This ISCP is the AI-specific contingency plan for the `chatcat` system. It extends the
enterprise BCP. It is activated on confirmed disruption of the serving, retrieval, judge,
or primary provider paths.

### 1.2 Scope
Per the policy. Lists the 8 components from the BIA.

### 1.3 Assumptions
- Cloud provider region-A is the primary site.
- Enterprise network / identity remains available (that is handled by enterprise BCP).
- The hosted-model provider will declare an outage in their status feed or will be
  inferred from the provider-error-rate alert.

## Section 2 — Concept of operations

### 2.1 System description
Links to the architecture doc; names the eight components.

### 2.2 Overview of recovery phases
Three phases per SP 800-34 §4:
- **Activation and notification** (declare incident, call IC, notify stakeholders)
- **Recovery** (execute strategy, bring services back)
- **Reconstitution** (return to normal operations in the primary site; declare recovery)

### 2.3 Roles and responsibilities
- Oncall IC: activation authority, recovery leadership.
- Vendor Liaison: communicates with provider; executes fallback pin.
- ML Platform lead: judge-pipeline reconstitution.
- Data Platform lead: retrieval-index recovery.

## Section 3 — Activation and notification

### 3.1 Activation criteria
The oncall IC activates the ISCP if any of:
- Regional outage declared by cloud provider affecting region-A **and** `chatcat-serving`
  error rate > SLO fast-burn threshold for > 5 min in region-A pods.
- Hosted-model provider primary error rate > 25% for > 10 min **and** fallback contract
  not auto-engaged.
- Logical corruption event on the retrieval index (incoming write failure cluster or a
  judge-pipeline groundedness drop > 5 sigma with no deploy correlate).
- Weight-manifest attestation failure on primary path.

### 3.2 Notification procedure
On activation, the IC:
1. Opens the incident channel per `mod-108` chapter 02.
2. Pages the Vendor Liaison and the Data Platform lead.
3. Notifies the status-page owner to publish an "investigating" update.
4. Notifies legal if the activation is caused by a confirmed security event.

### 3.3 Outage assessment
Within 20 min of activation, the IC determines:
- Scope: which components, which region.
- Likely duration: < 1h / 1-4h / > 4h — this drives whether to execute full failover.
- Rollback readiness: known-good pins available? Last green deploy?

## Section 4 — Recovery

The recovery procedures live in the runbook library (`mod-108/exercise-01`) and are
cross-referenced here. The ISCP's job is to sequence them.

### 4.1 Sequence for regional outage of region-A

1. `rb-regional-failover-dns` — flip DNS-weighted routing to 100% region-B. **Timer start.**
2. `rb-retrieval-index-promote-replica` — promote region-B replica to primary.
3. `rb-session-store-promote-replica` — promote region-B Redis replica.
4. `rb-vendor-fallback-pin` — if the regional outage also affects provider endpoints
   (and some do), pin to fallback provider.
5. Monitor: declare RTO(serving) met when `chatcat_route_availability{region="B"} >= 0.995`
   for 10 min.
6. `rb-judge-pipeline-warm-start` — stand up judge in region-B; prime from snapshot.
7. Declare RTO(quality) met when judge delta vs. baseline is within 2 sigma for 2 cycles.

### 4.2 Sequence for logical corruption of retrieval index

1. Freeze writes to the index (both regions).
2. Identify the last known-good snapshot (tagged by judge-groundedness green).
3. Restore from snapshot into region-B replica.
4. Diff region-A and region-B replicas; commit region-B as source of truth; re-replicate.
5. Resume writes; validate judge pipeline.

### 4.3 Sequence for primary-provider outage

1. Confirm via provider status + our provider-error-rate metric.
2. `rb-vendor-fallback-pin` — flip the router breaker.
3. Hold at fallback until provider confirms resolution **and** our own canary confirms
   recovered error rate for 15 min.
4. Un-pin gradually (10% → 50% → 100%) mirroring the exercise-01 canary pattern from
   `mod-111`.

## Section 5 — Reconstitution

### 5.1 Validation
- Serving: error rate, p95 TTFT per region at or below SLO.
- Quality: groundedness, refusal, structured-output validity within 2 sigma of baseline.
- Retrieval: index freshness metric within RPO.
- Weights: attestation verified against the pin.

### 5.2 Deactivation
The IC declares the incident resolved only when both RTOs are met. A separate
post-incident task owns the return to active-active primary — that happens outside the
ISCP activation window.

### 5.3 Post-incident actions
- Post-mortem per `mod-108/exercise-03`.
- Fixture back-feed per `mod-108/exercise-05`.
- ISCP update review (step 7) if any assumption was invalidated.

## Section 6 — Plan appendices

- A: contact list (role, primary, backup, escalation).
- B: dependency map (one-page diagram).
- C: runbook index with the full URL for each `rb-*` referenced above.
- D: vendor-risk register extract (`mod-107`).
- E: judge-pipeline snapshot catalogue (location, retention, last-validated date).

## Section 7 — Change log

| Date | Author | Change |
|------|--------|--------|
| 2026-11-13 | AI SRE | Initial publication |

## Section 8 — Approvals

- Prepared by: `<CPC>` — <date>.
- Reviewed by: `<ISSO>` — <date>.
- Approved by: `<System Owner>` — <date>.

## Section 9 — Testing and maintenance

Tested per `mod-113/exercise-05` schedule. Maintained per `mod-113/exercise-06`. The plan
is invalid as evidence for audit if §9's last-tested date is more than one quarter old
(see audit-pack definitions in `mod-113/exercise-06`).
```

## 3. Validation steps

A reviewer can mechanically validate the deliverable set in four passes:

1. **Coverage check.** For each of SP 800-34 §3's seven steps, confirm a corresponding
   artifact exists (or, for steps 6–7, a cross-reference). The table in §1.1 is the checklist.
2. **AI-specific additions.** Confirm the BIA separates RTO(serving) from RTO(quality);
   confirm RPO is given separately for index and for weights; confirm at least three
   AI-specific preventive controls (PC-04/06/07 or equivalent) appear in the control
   inventory with rationale.
3. **Cross-reference integrity.** Every `rb-*` the ISCP names must resolve to a runbook in
   `mod-108/exercise-01`; every "see `mod-107` vendor register" must resolve to a real
   register entry.
4. **Tabletop walkthrough.** Walk a reviewer through activation for one scenario (regional
   outage) using only the plan. If any step requires knowledge not in the plan, the plan
   is incomplete.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | SP 800-34 seven steps covered *(hard gate)* | Steps missing or renamed | All seven present but step 2/3 shallow | All seven present with per-step AI additions |
| 2 | BIA — dual RTO *(hard gate)* | Single RTO | Dual RTO named but same value | RTO(serving) and RTO(quality) separately justified |
| 3 | BIA — RPO per asset class *(hard gate)* | One RPO for everything | Separate RPO for data | Separate RPO for session, index, weights with rationale |
| 4 | Preventive controls — AI-specific *(hard gate)* | Generic only | 1-2 AI-specific | ≥ 3 AI-specific controls with rationale |
| 5 | Recovery strategy — alternate-site choice | Not stated | Stated | Stated with cost-vs-coverage justification |
| 6 | ISCP — activation criteria | Vague / qualitative only | Quantitative but single-signal | Quantitative multi-signal with named thresholds |
| 7 | ISCP — recovery sequence | Narrative only | Sequence listed | Sequence with timer, exit criteria per step, and runbook cross-refs |
| 8 | ISCP — reconstitution validation | "Looks good" | Signal-based | Dual-RTO validation + weight-attestation + judge snapshot comparison |
| 9 | Cross-references to vendor and runbook libraries | Missing | Named without resolving | Named and resolvable |
| 10 | Maintenance cadence | Annual only | Annual + one trigger | Annual + named triggered events (version bump, provider swap, region add) |

## 5. Common mistakes

- **Collapsing RTO.** Learners frequently publish a single RTO number and declare
  "recovered" when the route is up, which ignores the quality regression that is often the
  actual user-visible impact. SP 800-34's MTD/RTO are ceilings; splitting them for an AI
  workload is an interpretation, not an exception.
- **Silent provider assumptions.** The ISCP that assumes the hosted model "just works"
  until it doesn't and then names no fallback is the plan that fails on the first vendor
  incident. The fallback contract (PC-05) must be a real, pre-flighted contract.
- **No weight attestation.** A plan that recovers the system to an unspecified model
  version is not recovered to anything auditable. The attestation control is cheap and
  the omission is systemic.
- **Replica-as-backup.** A hot replica protects against regional loss; it does not protect
  against logical corruption of the index (the corruption replicates). The strategy must
  pair replica with snapshot.
- **Treating "test the plan" as owned by the plan itself.** SP 800-34 §3.6 wants the test
  programme owned explicitly. This solution offloads the schedule to `exercise-05`; the
  ISCP must cross-reference it, not duplicate it.
- **Policy that bakes in the schedule.** Baking the test schedule or the control list into
  the top-level policy makes the policy churn; keeping them as separate referenced
  documents keeps the policy stable and the operational documents live.

## 6. References

- NIST SP 800-34 Rev. 1, *Contingency Planning Guide for Federal Information Systems* —
  the authoritative seven-step process, ISCP template (Appendix A), BIA guidance (§3.2),
  and recovery-strategy guidance (§3.4) applied throughout this solution.
- NIST SP 800-53 Rev. 5, Contingency Planning (CP) control family — the control-ID
  references (CP-8, CP-9) in `controls/chatcat-preventive-controls.md`.
- FIPS 199, *Standards for Security Categorization of Federal Information and Information
  Systems* — the confidentiality / integrity / availability impact levels used in the BIA.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the MANAGE function ("MANAGE 4: AI risks and benefits are managed on an ongoing basis") grounds why BCP cadence for AI systems must include triggered reviews on model-version bump and provider swap.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML supply chain` motivates weight attestation (PC-04) and image-digest allowlisting (PC-08); `ML10: model poisoning` motivates the snapshot pairing for the retrieval index.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the ML supply-chain and ML-model-access technique classes justify PC-04 and PC-08 as adversary-facing controls, not only availability controls.
- `mod-107` vendor-and-provider-risk register (this repo) — the register the ISCP's §3.3 and §4.3 cross-reference for provider failover.
- `mod-108/exercise-01` runbook library — the runbooks the ISCP's §4 sequences.
