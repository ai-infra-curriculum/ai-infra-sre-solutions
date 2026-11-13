# Exercise 06 — Legal / Compliance Handoff for AI Safety Incidents — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-06-legal-compliance-handoff-for-ai-safety-incidents.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below is
> derived from chapter 06 of the module (which the exercise is explicitly paired to) and chapter
> 06 §8's worksheet. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

> **Scope boundary.** This is operational guidance for the AI SRE's handoff to legal /
> compliance. It is **not** legal advice. Determinations about whether a regulatory
> threshold is crossed are made by legal, the `ai-risk-engineer` function, and executive
> leadership — never by the SRE. Chapter 06 §1 is explicit; the exercise's deliverable is
> the structured handoff package, not a filing.

## 1. Solution overview

The exercise asks the learner to produce the organisation-shaped version of chapter 06's
handoff artifacts for AI safety-adjacent incidents. The deliverable is a bundle in a
`docs/incident-command/legal-compliance-handoff/` directory containing:

1. A **trigger reference card** — the pre-committed conditions that page legal / compliance
   immediately (chapter 06 §3).
2. The **artifact package template** — the structured bundle the AI SRE prepares for legal
   when a trigger fires (chapter 06 §4).
3. A **preservation and legal-hold procedure** (chapter 06 §5).
4. A **framework-mapping reference** — the three frameworks chapter 06 §2 names (EU AI Act,
   FDA PCCP, SR 11-7 / SR 26-2) and the fields of the technical narrative that map to each.
5. A **communication protocol** with legal and compliance (chapter 06 §6).
6. A **worked example** — the artifact package for an illustrative safety-adjacent event.

All language in the worked example is factual-register and avoids speculation; chapter 06
§1 and §6 govern. Framework-specific dates and thresholds are cited from chapter 06 which in
turn cites the official texts; where a specific paragraph is required for a filing, the
retro owner and legal consult the primary source directly.

## 2. Implementation — worked answer

### 2.1 Trigger reference card — `docs/incident-command/legal-compliance-handoff/triggers.md`

The card is designed to be readable in-incident. Any trigger below pages legal and
compliance *immediately*, regardless of the engineering diagnosis. The AI SRE's duty is
engagement, not determination.

```markdown
# Legal / compliance page triggers — v2026-11-01

## Safety triggers

- **Toxic-output sustained.** `rb-toxic-output-blast` firing on a user-facing route for
  any sustained duration (configuration: >= 5 min on a user-facing route, >= 3× baseline
  on the safety-classifier output rate).
- **Toxic-output single egregious event.** Any user-reported safety flag with an
  egregious sample, reported via support, user channels, or detected by a monitor.
- **Model output materially contributes to harm.** Health, safety, or financial harm
  reported by a user, support, or monitor.
- **Regulated-domain incorrect answer.** Medical, legal, or financial advice emitted on
  a regulated route (per contract or jurisdiction).
- **Suspected or confirmed jailbreak.** A safety control defeated in production.

## Compliance triggers

- **Data-residency violation.** An EU-origin request is observed to have traversed a
  non-EU route against contract.
- **ZDR / no-training commitment potentially violated.** Customer data potentially
  reached a provider path not covered by the ZDR commitment.
- **Potential exfiltration of user interactions.** Through a provider or vendor path.
- **Loss of audit-log integrity.** On a regulated path.

## Reporting triggers (framework-specific)

- **EU AI Act Art. 73 serious-incident candidate.** An incident that *may* meet the Act's
  serious-incident threshold — serious harm; serious and irreversible disruption of
  critical infrastructure; serious violation of fundamental-rights obligations; serious
  damage to property or the environment. Legal determines whether the threshold is
  crossed; the AI SRE engages legal at the candidate stage and prepares the
  shortest-window-compatible artifact package (chapter 06 §2.1 — 2 days / 10 days /
  15 days depending on harm category).
- **FDA PCCP deviation.** On an FDA-cleared AI device: a deviation from the stated
  performance envelope, an out-of-PCCP modification, or a serious adverse event
  associated with the device output.
- **SR 26-2 / SR 11-7 material model regression.** A regression that materially shifts
  the model's risk profile relative to its documented posture.
- **Regulator inquiry.** Any inquiry opened in connection with the incident.

## Who gets paged

| Trigger class | Page |
|----------------|------|
| Safety | `safety-oncall`, `legal-oncall`, `pr-oncall` |
| Safety at scale / sustained | add `exec-oncall` |
| Compliance | `legal-oncall`, `privacy-oncall`, `ai-risk-engineer-oncall` |
| Reporting | `legal-oncall`, `ai-risk-engineer-oncall`, `exec-oncall` |
| Regulator inquiry | `legal-oncall`, `exec-oncall`, `pr-oncall` |

## What NOT to do

- Do not call a regulator.
- Do not draft a public statement.
- Do not reply to press.
- Do not delete artifacts, logs, traces, or configurations even after resolution until
  legal clears the retention end.
- Do not paste customer content or model outputs into the open incident channel.
```

### 2.2 Artifact package template — `templates/legal-handoff-package.md`

The package is the AI SRE's output to legal. It is structured, factual, and separates
observation from inference. The five sub-documents come from chapter 06 §4.

```markdown
---
package_id: <pkg-YYYY-MM-DD-rb-slug-surface>
incident_id: <inc-YYYY-MM-DD-rb-slug-surface>
trigger_fired: <trigger-id from triggers.md>
prepared_by: ai-sre-ic-oncall
prepared_at: <UTC>
distribution: [legal-oncall, ai-risk-engineer-oncall]
retention: <legal-hold-id; retention end TBD by legal>
---

# Legal / compliance handoff package — <pkg-id>

## 1. Technical narrative

### Incident identifiers

- **Internal incident ID:** <inc-...>
- **Incident channel:** `#inc-...`
- **Model version IDs:** <primary and fallback model IDs with version dates>
- **Prompt / config / weights hashes:** <pinned hashes of the artifacts active in the window>
- **Related runbook:** `rb-<slug>`
- **Internal classification (chapter 03 §3):** `<category>`

### Chronology (UTC)

- First signal: <HH:MM UTC> — <signal name> on <system>
- Declare: <HH:MM UTC> — SEV-<N>
- Mitigation steps: <timestamped list with owner-role and effect>
- Stable: <HH:MM UTC>
- Resolve: <HH:MM UTC>

### Scope

- **User cohorts:** <count if known; cohort identifiers>
- **Regions:** <regions / data-residency flags>
- **Product surfaces:** <routes>
- **Data classes touched:** <PII / PHI / PCI / customer-content / none>

### Technical root cause

- **Category (chapter 03 §3):** `<category>`
- **Specific instance:** <one sentence>

### Detection signal

- **Signal name + system of record:** <metric / query / dashboard / trace query>
- **First-fire time (UTC):** <HH:MM>

### Mitigation

- **Mitigation(s) applied and their effect:** <list with UTC timestamps>

### Residual exposure

- **After mitigation, what remains:** <explicit — e.g., affected content may remain in
  customer caches; a specific route is still on the fallback provider; some sessions
  completed before mitigation and the output is retained under the preservation list.>

## 2. Impact assessment

Quantitative where possible; qualitative where not.

- **Number of affected users / requests / sessions:** <count>
- **Duration of exposure:** <HH:MM>
- **Categories of impact:**
  - Availability (SLO burn): <SLO name and percent-of-budget burned>
  - Quality (eval regression): <metric and delta>
  - Safety (classifier events): <count by category>
  - Privacy (data exposure): <categories; count of records>
  - Financial (cost or contractual): <quantified where measurable>
- **Known or suspected third-party dependency involvement:** <provider / CDN / other>

## 3. Artifact inventory

Every artifact has a path, access-control note, and retention date. Nothing on this list
is deleted without legal's clearance.

| Artifact | Path / location | Access control | Retention end | Chain-of-custody notes |
|----------|-----------------|-----------------|----------------|------------------------|
| Timeline source | <...> | <role gated> | <TBD by legal> | <who-read-what-when log> |
| Telemetry snapshots | <Grafana / Honeycomb pins> | <role gated> | <TBD> | <...> |
| Sample outputs (if customer content at issue) | <controlled-access store> | <role gated, named access> | <TBD> | <...> |
| Request / response logs | <log store, pinned window> | <role gated> | <TBD> | <...> |
| Model artifacts | <registry pointer, version-id, weights-hash> | <role gated> | <permanent> | <...> |
| Vendor correspondence | <ticket IDs, status-page snapshots> | <role gated> | <TBD> | <...> |

## 4. Classification map (framework-specific)

Legal selects the applicable framework(s) and populates the table. The table is a drafting
aid; legal rewrites into the actual filing shape.

### 4a. EU AI Act Art. 73 drafting aid (if applicable)

| Art. 73 element | Fact from the narrative |
|------------------|--------------------------|
| Date and place of incident | <UTC; affected regions> |
| Serious harm category | <none identified / under-review by safety; health / fundamental-rights / critical-infrastructure / property-environment> |
| Information identifying the system | <model version ID, product surface, intended purpose, CE-mark reference if applicable> |
| Any corrective action taken | <mitigation summary with UTC> |

### 4b. FDA PCCP drafting aid (if applicable)

| PCCP element | Fact from the narrative |
|---------------|--------------------------|
| Modification specification (covered / out-of-PCCP) | <...> |
| Modification protocol applied | <...> |
| Impact on safety and effectiveness | <...> |
| Serious adverse-event association | <...> |

### 4c. SR 26-2 / SR 11-7 drafting aid (if applicable)

| MRM element | Fact from the narrative |
|--------------|--------------------------|
| Model-inventory entry | <model id, owner, intended use> |
| Documentation update required | <...> |
| Validation status | <in-period / expired / re-validation triggered> |
| Governance decision needed | <threshold crossed / escalation to MRM committee> |

## 5. Data-handling attestation

Factual; not interpretive. Legal is the consumer.

- **Data processed during the window:** <classes; systems that touched it>
- **Data retained for incident analysis:** <classes; where; access control>
- **Data shared with third parties:** <recipient; legal basis; what was shared>
```

### 2.3 Preservation and legal-hold procedure

```markdown
# Preservation and legal-hold procedure — v2026-11-01

## When a trigger fires

1. **Page legal and compliance immediately.** Per the trigger card.
2. **Pin the incident window.** The IC records the UTC window on the channel topic. All
   subsequent preservation keys off this window.
3. **Open the preservation ticket.** `legal-hold-<pkg-id>` in the hold tracker. Legal
   assigns the retention end; engineering does not.
4. **Freeze the deletion schedule.** For every artifact class that lives in a store with
   automatic deletion, engineering issues a hold:
   - Logs: tag the window with a `legal_hold` label; the retention policy's
     age-based prune rule skips labelled entries.
   - Traces / metrics: export snapshots to the immutable store.
   - Sample outputs: already in the controlled-access store (chapter 02 §5.3 forbids
     in-channel); confirm the store's retention for the hold-tagged window.
   - Model artifacts: pin the model-registry version; mark the version as under hold.
   - Vendor correspondence: export tickets and status-page snapshots to the immutable
     store.
5. **Document who accessed what.** The chain-of-custody log in the artifact inventory is
   updated as responders read artifacts. Access during the live incident is logged
   automatically; access after resolve is logged by role and name.

## Immutable storage

- Preserved artifacts land in an append-only / object-locked store.
- Legal determines retention end; engineering defaults are superseded.
- Access is role-gated; named access is logged.

## Chain of custody

- For artifacts that may be presented externally, the access log is preserved alongside
  the artifact.
- No modifications to preserved artifacts; derivatives are stored separately with a
  reference to the source.

## Release from hold

- Legal clears the hold end; only legal clears the hold end.
- Upon clearance, the normal retention policy resumes from that point.
```

### 2.4 Framework-mapping reference

The AI SRE's working knowledge is sufficient to know *when* to hand off — not to interpret.
The table below is the engineering-facing summary; legal and the `ai-risk-engineer`
function interpret.

| Framework | Scope test | AI SRE touchpoint | Where chapter 06 cites the source |
|-----------|-------------|-------------------|------------------------------------|
| **EU AI Act, Reg. (EU) 2024/1689, Art. 72 (post-market monitoring)** | Provider of a high-risk AI system placed on the EU market | Telemetry we already produce (`mod-104`) is a compliance deliverable — evidence that monitoring exists and is operating | Chapter 06 §2.1; the Digital Omnibus on AI (Reg. (EU) 2026/1744, in force 2026-07-27) replaces Art. 72's originally-planned Commission implementing act with non-binding guidance due by 2027-09-02 |
| **EU AI Act, Reg. (EU) 2024/1689, Art. 73 (serious-incident reporting)** | Serious incident on a high-risk AI system — harm, critical-infrastructure disruption, fundamental-rights violation, property / environment damage | Shortest-window-compatible artifact package prepared at trigger; legal determines whether the threshold is crossed | Chapter 06 §2.1; windows: 2 days (critical infrastructure / widespread infringement), 10 days (death), 15 days (general) |
| **FDA PCCP for AI-enabled Device Software Functions** | AI-enabled medical device in the US | Trigger on any deviation from the stated performance envelope, any out-of-PCCP modification, any serious adverse event | Chapter 06 §2.2; guidance *Marketing Submission Recommendations for a Predetermined Change Control Plan for Artificial Intelligence-Enabled Device Software Functions* issued 2024-12-04 |
| **SR 26-2 (US bank model risk management)** | US bank model in scope of the interagency guidance | Trigger on material model-performance deviation, risk-profile shift, or audit / validation entitlement | Chapter 06 §2.3; SR 26-2 / OCC Bulletin 2026-13 / FDIC FIL-16-2026 issued 2026-04-17, supersedes SR 11-7 (2011) |
| **NIST AI RMF and Generative AI Profile (NIST-AI-600-1)** | Voluntary; many organisations use it as the internal risk vocabulary | Internal documentation shape; some customer contracts align with it | Chapter 06 §2.4; see [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) |
| **ISO/IEC 42001:2023** | AI management system standard | Clauses 8–10 and Annex A controls; incidents may trigger a non-conformance | Chapter 06 §2.4 |
| **State AI laws (e.g., Colorado AI Act, SB 24-205)** | Deployers of high-risk AI systems per the jurisdiction's scope | Colorado: AG notification duty within 90 days of discovering algorithmic discrimination; effective 2026-06-30 (delayed from the original 2026-02-01 date) | Chapter 06 §2.4 |
| **Data-breach notification laws (GDPR Art. 33 / state laws)** | AI incident involving unauthorised disclosure of personal data | Short windows (e.g., GDPR Art. 33: 72 hours to the supervisory authority) | Chapter 06 §2.4 |
| **CIRCIA (US critical-infrastructure incident reporting)** | Covered entity in a critical-infrastructure sector | Proposed windows: 72 hours for covered cyber incidents; 24 hours for ransomware payments. As of late 2026, the final rule remained under OMB / White House review and no obligation was yet in force | Chapter 06 §2.4 |

### 2.5 Communication protocol with legal and compliance

```markdown
# Communication protocol — v2026-11-01

## Four conventions

1. **Named routing.** Page `legal-oncall` and `ai-risk-engineer-oncall` at trigger — not
   an individual by name. The rotation is visible in the IR tooling.
2. **Facts first, implications later.** The initial brief is five bullets: what happened,
   when, scope, detection, mitigation. Implications are discussed after the fact base.
3. **Written primary, verbal secondary.** The written record (channel, package) is the
   source of truth. Verbal briefings supplement.
4. **Clarify unknowns explicitly.** "We do not yet know whether EU customer data was
   affected" is correct. "We believe EU customers were not affected" is a professional
   failure — it overstates certainty.

## Initial-brief template (five bullets)

```
- **What happened:** <one line>
- **When (UTC):** <HH:MM declare; HH:MM mitigation; HH:MM stable>
- **Scope:** <routes, regions, user count, data classes>
- **Detection:** <signal; system of record>
- **Mitigation:** <applied; effect>
```

## What legal / compliance commit back

- Response within the handoff window defined in function doctrine (matched to the
  applicable framework's shortest deadline).
- Determination of whether a reportable threshold is crossed; communication back to the IC.
- Ownership of external communication end-to-end.
```

### 2.6 Worked example — artifact package for an illustrative safety-adjacent event

A worked package demonstrating the template against an illustrative incident. The scenario
is fictional; all names are role-only; the package is written in the voice the real
submission would use.

```markdown
---
package_id: pkg-2026-10-28-toxic-output-blast-advice
incident_id: inc-2026-10-28-toxic-output-blast-advice
trigger_fired: toxic-output-sustained
prepared_by: ai-sre-ic-oncall
prepared_at: 2026-10-28T19:40:00Z
distribution: [legal-oncall, ai-risk-engineer-oncall, safety-oncall]
retention: legal-hold-2026-10-28-01; retention end TBD by legal
---

# Legal / compliance handoff package — pkg-2026-10-28-toxic-output-blast-advice

## 1. Technical narrative

### Incident identifiers

- **Internal incident ID:** inc-2026-10-28-toxic-output-blast-advice
- **Incident channel:** `#inc-2026-10-28-toxic-output-blast-advice`
- **Model version IDs:** primary `gpt-4o-mini-2024-07-18`; fallback
  `claude-3-5-haiku-20241022`
- **Prompt / config / weights hashes:** prompt `pt-advice@3f9a2c1`, safety-filter config
  `sf-cfg@7b0e551`
- **Related runbook:** `rb-toxic-output-blast`
- **Internal classification (chapter 03 §3):** `prompt-config` (primary); `provider`
  (contributing — silent model-side moderation change)

### Chronology (UTC)

- 2026-10-28 17:12 — First signal: `chatcat_safety_classifier_rate{category="self-harm"}`
  crossed 3× 7-day baseline on the `/advice` route.
- 17:14 — Second signal: five user-reported safety flags within a 2-minute window on
  `/advice`.
- 17:16 — Incident declared SEV-1 (trigger `toxic-output-sustained` + egregious single-
  sample reports); `safety-oncall`, `legal-oncall`, `pr-oncall`, `exec-oncall` paged.
- 17:18 — Moderation strictness raised on `/advice`; filter configuration reverted to
  `sf-cfg@7b0e551-prev`.
- 17:21 — `/advice` route traffic halted via feature flag `advice.enabled=false`.
- 17:24 — Sample outputs exported to the controlled-access store; channel reference ID
  `art-pkg-2026-10-28-01`.
- 17:48 — Root cause confirmed: a safety-filter config change at 16:55 UTC inadvertently
  disabled the `self-harm` category classifier. Reverted and verified.
- 18:04 — Stable.
- 19:30 — Resolved (feature flag kept off pending full safety review).

### Scope

- **User cohorts:** ~3,200 sessions touched the `/advice` route in the 17:12–17:21 window.
  Flagged outputs: 47. User-reported events: 11.
- **Regions:** global; `/advice` is not geo-fenced.
- **Product surfaces:** `/advice` (health and well-being coaching feature).
- **Data classes touched:** user-generated content; no PHI (the surface does not collect
  health identifiers per the privacy review on file).

### Technical root cause

- **Category:** `prompt-config` (safety-filter configuration change)
- **Specific instance:** The 16:55 UTC deploy of `sf-cfg@7b0e551` refactored the filter
  configuration schema and, due to a mis-mapped key, disabled the `self-harm` category
  classifier on the `/advice` route.

### Detection signal

- **Signal name + system of record:**
  `chatcat_safety_classifier_rate{category="self-harm", route="/advice"}` (Prometheus,
  dashboard `chatcat/quality-health` panel `Safety classifier rate per route`).
- **First-fire time (UTC):** 2026-10-28 17:12.

### Mitigation

- 17:18 UTC — Raised moderation strictness; reverted filter configuration. Effect:
  classifier output resumed within 2 minutes.
- 17:21 UTC — Halted `/advice` feature via flag `advice.enabled=false`. Effect: no further
  user traffic.

### Residual exposure

- 47 flagged outputs were delivered to 47 user sessions between 17:12 and 17:18. The
  outputs are preserved under `legal-hold-2026-10-28-01` in the controlled-access store.
- Four user sessions reported self-harm-adjacent content specifically; those sessions are
  individually tagged in the artifact store for the safety review.
- No downstream auto-publish (email / social / support) exists on this route; no onward
  distribution of the content beyond the user sessions themselves.

## 2. Impact assessment

- **Number of affected users / requests / sessions:** 47 flagged outputs; 3,200 sessions
  on the route in the window.
- **Duration of exposure:** 9 minutes (17:12 → 17:21).
- **Categories of impact:**
  - Availability: negligible (route remained available, serving a classifier-degraded
    experience).
  - Quality (eval regression): not the primary concern here.
  - Safety (classifier events): 47 flagged outputs (`self-harm` category); 11 user-
    reported events.
  - Privacy (data exposure): none identified; no PHI on this route per privacy review.
  - Financial / contractual: pending customer-contract review for the enterprise tier.
- **Third-party dependency involvement:** none identified; both the primary and fallback
  model providers were operating nominally in the window.

## 3. Artifact inventory

| Artifact | Path / location | Access control | Retention end | Chain-of-custody notes |
|----------|-----------------|-----------------|----------------|------------------------|
| Timeline source | `/incidents/inc-2026-10-28-toxic-output-blast-advice/timeline.md` | ai-sre-leadership, legal | TBD | Scribe + 3 reviewers logged |
| Telemetry snapshots | Grafana `chatcat/quality-health` pinned to window; Honeycomb saved query `art-pkg-2026-10-28-telemetry` | ai-sre-leadership, legal | TBD | Access logged via SSO |
| Sample outputs | Controlled-access store `art-pkg-2026-10-28-01` (47 items) | safety-oncall, legal-oncall, exec-oncall | TBD | Store enforces per-access log |
| Request / response logs | `logs://chatcat-prod/advice` tagged `legal_hold=pkg-2026-10-28-01` window 17:00–17:30 UTC | ai-sre-leadership, legal | TBD | Tag freezes auto-prune |
| Model artifacts | Registry entries: `pt-advice@3f9a2c1`, `sf-cfg@7b0e551`, `sf-cfg@7b0e551-prev`, `gpt-4o-mini-2024-07-18`, `claude-3-5-haiku-20241022` | ai-sre-leadership | permanent | Registry versions pinned |
| Vendor correspondence | no vendor tickets opened; provider status-page snapshots pinned in the channel | ai-sre-leadership, legal | TBD | n/a |

## 4. Classification map (drafting aid)

### 4a. EU AI Act Art. 73 drafting aid

| Art. 73 element | Fact |
|------------------|------|
| Date and place of incident | 2026-10-28 17:12–17:21 UTC; global route |
| Serious harm category | Under safety-team review; four user sessions flagged self-harm-adjacent content specifically |
| Information identifying the system | `/advice` route, `gpt-4o-mini-2024-07-18`, prompt `pt-advice@3f9a2c1` |
| Any corrective action taken | Filter configuration reverted; feature flag halted route |

### 4b. FDA PCCP drafting aid

Not applicable — `/advice` is not an FDA-cleared device.

### 4c. SR 26-2 drafting aid

Not applicable — the model is not in scope of the US bank MRM guidance for this product.

## 5. Data-handling attestation

- **Data processed during the window:** user-generated queries and model outputs on
  the `/advice` route; no PHI by product design.
- **Data retained for incident analysis:** the 47 flagged outputs and the 3,200 session
  request / response logs, preserved under `legal-hold-2026-10-28-01`.
- **Data shared with third parties:** none as of handoff. Any sharing with the model
  provider would be a legal decision, not an engineering decision.
```

## 3. Validation steps

1. **Trigger card completeness.** The card lists all nine triggers from chapter 06 §3 (five
   safety, four compliance, four reporting; the overlap is intentional). Automated check:
   grep the card against the chapter 06 list and fail if any trigger is missing.

2. **Package template compliance.** A package validates against the five-section structure:
   technical narrative, impact assessment, artifact inventory, classification map,
   data-handling attestation.

   ```bash
   REQUIRED=('## 1. Technical narrative' '## 2. Impact assessment' \
             '## 3. Artifact inventory' '## 4. Classification map' \
             '## 5. Data-handling attestation')
   for f in packages/*.md; do
     for h in "${REQUIRED[@]}"; do
       grep -qF -- "$h" "$f" || { echo "FAIL: $f missing $h"; exit 1; }
     done
   done
   ```

3. **Preservation discipline check.** Every artifact entry has: path, access control, and
   retention end. Automated: parse the inventory table and fail if any row is incomplete.

4. **Role-only language.** Grep the package body for `@[a-z]+` handles or human first-names
   outside the author-list front-matter. Any hit is a reviewer flag.

5. **Observation vs. inference separated.** Human review: the retro / package avoids
   "we believe" / "we think" where a fact statement is possible; where uncertainty is
   genuine, "we do not yet know X" is the correct form (chapter 06 §6).

6. **Framework-table correctness.** Any dated claim (Reg. numbers, effective dates,
   reporting windows) that the retro repeats inline from chapter 06 should cite chapter 06
   or the primary source directly, not be re-stated from memory. If a specific fact matters
   for a filing, legal re-verifies it against the official text.

7. **Trigger drill.** Semi-annual tabletop: the IC team reads a scripted incident scenario
   and should page the correct roles within 2 minutes based only on the trigger card.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at ≥ 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Trigger card coverage *(hard gate)* | Missing safety / compliance / reporting bucket | Partial coverage | All nine triggers with paged roles |
| 2 | Package five-section structure *(hard gate)* | One or more sections missing | Present but thin | All sections populated with specific content |
| 3 | Framework mapping | Missing | One framework | EU AI Act Art. 72 & 73, FDA PCCP, SR 26-2 all present |
| 4 | Preservation procedure | Missing | Freeze policy only | Freeze policy + immutable storage + chain-of-custody + release |
| 5 | Observation vs. inference *(hard gate)* | Mixed | Partial | Factual register throughout; uncertainty explicit |
| 6 | Role-only language *(hard gate)* | Named humans in package | Mix | Role-only |
| 7 | Classification map drafting aid | Missing | One framework's table | At least one framework table + "not applicable" where not applicable |
| 8 | Data-handling attestation | Missing or interpretive | Factual but incomplete | Factual, complete, three sub-fields populated |
| 9 | Communication protocol | Missing | Partial | Four conventions + initial-brief template + commit-back expectations |
| 10 | Scope boundary ("not legal advice") | Missing | Implicit | Explicit disclaimer; defers framework determination to legal |

## 5. Common mistakes

- **Drafting a regulator submission.** Not in scope. The AI SRE's output is the technical
  narrative; legal rewrites.
- **Calling a regulator.** Never. Chapter 06 §1 is explicit.
- **Naming the vendor publicly.** Chapter 02 §5.3; chapter 06 §6 "written primary, verbal
  secondary."
- **Deleting logs after resolve.** Not without legal clearance. The preservation procedure
  overrides the engineering retention default.
- **Pasting customer content into the open incident channel.** Chapter 02 §5.3 and chapter 06
  §4.3. The channel references artifacts by ID; the content lives in the controlled-access
  store.
- **Overstating certainty about EU customers.** "We believe EU customers were not affected"
  is a professional failure. The correct form is "we do not yet know."
- **Omitting the residual-exposure field.** The window closes at mitigation, but the exposure
  often persists (cached content, retained outputs). Legal needs this explicitly.
- **Blanket redaction on legal request.** Negotiate specific redactions (chapter 03 §7
  "over-redacted retro"); blanket redaction destroys the technical narrative.
- **Combining the retro and the handoff package into one document.** They serve different
  audiences. The retro is internal; the package is prepared for legal with legal's
  specificity requirements. They share facts but not form.
- **Reproducing regulatory texts without citation to the primary source.** Chapter 06 is a
  practitioner's guide; when a specific threshold or deadline matters for a filing, legal
  and the retro owner re-verify against the official text.

## 6. References

- Module chapter 06, *Legal / Compliance / Regulator-Facing Handoff for AI-Safety Incidents*
  — the handoff premise, the three frameworks (EU AI Act Art. 72 & 73, FDA PCCP, SR 26-2
  / SR 11-7), the trigger taxonomy, the artifact package structure, the preservation
  discipline, the communication protocol, and the post-handoff follow-through that this
  exercise implements.
- Module chapter 02, *Incident Command for AI Systems* — the safety-adjacent decision tree
  the trigger card codifies, and the channel discipline that keeps customer content out of
  the open channel.
- Module chapter 03, *Blameless Post-Mortems for AI Systems* — the technical narrative the
  package is derived from.
- Module chapter 05, *Regression-Fixture Back-Feed Discipline* — the fixtures that may land
  as regulatory-required commitments.
- EU AI Act (Regulation (EU) 2024/1689) and the Digital Omnibus on AI (Regulation (EU)
  2026/1744) — the primary sources chapter 06 §2.1 cites; legal consults directly.
- US FDA, *Marketing Submission Recommendations for a Predetermined Change Control Plan for
  Artificial Intelligence-Enabled Device Software Functions* (issued 2024-12-04) —
  [www.fda.gov/media/187905/download](https://www.fda.gov/media/187905/download). Chapter 06
  §2.2.
- Federal Reserve / OCC / FDIC interagency guidance SR 26-2 / OCC Bulletin 2026-13 /
  FDIC FIL-16-2026 (issued 2026-04-17), superseding SR 11-7 (2011) — chapter 06 §2.3.
- NIST AI Risk Management Framework and Generative AI Profile (NIST-AI-600-1),
  [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- ISO/IEC 42001:2023, *Information technology — Artificial intelligence — Management
  system* — chapter 06 §2.4.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — technique vocabulary for
  safety / security-adjacent handoff contexts (jailbreak, weight tampering, eval-set
  poisoning).
- OWASP Machine Learning Security Top 10,
  [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary for the safety / compliance triggers that align with ML top-10
  categories (ML02, ML06).
