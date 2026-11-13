# Exercise 05 — Incident Sensitive-Data Handling Discipline — Reference Solution

Paired learning file: `lessons/mod-110-security-safety-and-compliance-reliability-crossover/exercises/exercise-05-incident-sensitive-data-handling-discipline.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred
> below is derived from this module's chapter on handling sensitive data during AI
> incident response, and from `mod-108` chapter 02 §5.3 (channel discipline) and
> chapter 06 §5 (preservation and legal-hold). If the authored exercise narrows or
> widens scope on a subsequent content cycle, this solution should be regenerated.

> **Scope boundary.** This is operational discipline for the AI SRE and the incident
> responders. It describes **how** to handle sensitive data mid-incident and after.
> Classifications of the data itself (what is PII, what is PHI, what is customer-
> content) are the privacy / legal / `ai-risk-engineer` function's authority. This
> document operationalises those classifications; it does not make them.

## 1. Solution overview

The deliverable is a committed set of artifacts in `docs/incident-command/sensitive-
data-discipline/`:

1. A **data-class reference card** for in-incident use: short, legible on a screen,
   the on-call reads it while deciding whether to paste something into the channel.
2. A **channel-discipline standard** that codifies what is and is not permitted in the
   incident channel.
3. A **controlled-access store standard** — where sensitive artifacts actually live
   during and after an incident.
4. A **minimum-necessary access policy** for responders.
5. A **chain-of-custody procedure** for artifacts that may be shared with legal or
   third parties.
6. A **worked example** — a short IR transcript annotated with "right" and "wrong"
   responder behaviours, drawn against an illustrative incident.

The approach rests on three invariants:

1. **Classification precedes handling.** Before any artifact enters an incident
   workflow, its data class determines its routing. Responders do not re-classify
   on the fly.
2. **The channel is a coordination surface, not an evidence surface.** Customer
   content, model outputs containing PII, raw prompts, raw completions — none of
   these live in the channel. They live in the controlled-access store, and the
   channel references them by artifact ID.
3. **Minimum necessary.** Responders have access to what they need to resolve the
   incident; broader access is granted on request, by role, with an audit entry.

## 2. Implementation — worked answer

### 2.1 Data-class reference card — `docs/incident-command/sensitive-data-discipline/data-class-card.md`

Designed to be readable at a glance in-incident. Posted as a pinned message in every
incident channel on open.

```markdown
# Sensitive data — in-incident reference card — v2026-11-01

## Can I paste this in the channel?

| Thing | In channel? | Where it goes |
|-------|-------------|----------------|
| An **error message** or stack trace with no customer content | Yes | Channel |
| A **log line** that has been tested and contains no customer / model content | Yes | Channel |
| An **artifact ID**, e.g. `art-pkg-2026-11-13-01` | Yes | Channel |
| A **sample of customer prompt or output** | **No** | Controlled-access store; link by ID |
| A **classifier label** on a customer output (e.g., `self-harm=true`) | No | Controlled-access store; link by ID |
| A **customer email / name / handle** | No | Controlled-access store; never in channel |
| **Raw PII** (SSN, DoB, address, card number) | No, ever | Controlled-access store (encrypted); access logged |
| A **screenshot of a dashboard** with aggregated numbers | Yes | Channel (verify no PII in aggregations) |
| A **screenshot of a trace** that includes request bodies | No | Controlled-access store |
| A **vendor incident reference** (their incident ID) | Yes | Channel |
| A **customer-identifying trace header** (tenant ID) | Yes | Channel (tenant ID is not PII in our system) |
| A **customer-identifying ID that is a user ID** | No | Controlled-access store |

## If you are unsure — rule of thumb

If you would hesitate to paste this into a company-wide chat, it does not belong in
the incident channel. If you would hesitate to show it to your own manager, it does
not belong in the controlled-access store either without classification. In both cases,
ask the IC; the IC pages `privacy-oncall` if the answer is not obvious.

## What to do if something sensitive was pasted by accident

- Do **not** delete the message. Deletion is destruction of an artifact that may be
  under preservation. Flag the IC immediately.
- The IC follows `rb-sensitive-data-in-channel`: the content is copied to the
  controlled-access store under preservation, the channel message is tombstoned (not
  deleted) with a redacted replacement, and legal / privacy on-call is paged.
```

### 2.2 Channel discipline — `docs/incident-command/sensitive-data-discipline/channel-standard.md`

```markdown
# Incident channel — sensitive-data standard — v2026-11-01

## 1. Channel setup (on open)

1. Channel is role-gated: `#inc-<id>` is invite-only to the responder rotations
   enumerated in the IR tooling. Broader membership is granted by the IC on request.
2. The pinned top message is the data-class reference card.
3. The IC posts the controlled-access store ID for the incident
   (`art-pkg-<incident-id>`) and the preservation ticket ID.
4. Scribe is named; the timeline is the source of truth for sequencing.

## 2. In-channel behaviour

- Reference artifacts by ID, not by content. "See `art-pkg-2026-11-13-01` item 14"
  is correct; pasting the content of item 14 is not.
- Keep interpretive language in the retro, not in the channel. The channel is a
  coordination surface during the incident; the retro is where inference lives.
- Decisions get a write-up in the channel at the time they are made (scribe re-posts
  them structured). Verbal-only decisions do not exist.
- Do not screenshot private DMs into the channel. If someone briefed you in DM, ask
  them to re-brief in the channel, or ask the IC to prompt them.

## 3. Membership management

- New responder joins: the IC acknowledges in channel ("welcome, data-class card
  pinned above"). Access to the controlled-access store is granted on request, role-
  gated, with an audit entry.
- Responder leaves: the IC posts the handoff. Access to the store is not revoked at
  handoff (the audit log is needed for the retro); it is reviewed at the quarterly
  access review.

## 4. Closure

- On resolve, the channel is archived, not deleted. The channel transcript is pinned
  to the incident folder and the retention is inherited from the incident folder's
  retention (per `mod-108` chapter 06 preservation).
- Any in-channel pastes that turned out to be sensitive are migrated to the
  controlled-access store and tombstoned in the channel.
```

### 2.3 Controlled-access store standard — `docs/incident-command/sensitive-data-discipline/store-standard.md`

```markdown
# Controlled-access store — v2026-11-01

## 1. Store properties

- **Append-only / object-locked** for the retention window (legal hold extends).
- **Per-access logged**: every read is logged with role, name, timestamp, and the
  specific artifact key.
- **Role-gated**: entries have a base-access role set; named access is granted on
  top with an audit entry.
- **Encrypted at rest with a per-artifact key**; keys rotate on the organisation's
  schedule.
- **Geographically resident** per the organisation's data-residency policy. An EU-
  origin artifact lives in an EU-resident store; a cross-region replica is not
  established without a documented legal basis.

## 2. Artifact entry

Every entry has:

- `artifact_id` (globally unique; the channel references this).
- `incident_id`.
- `data_class` (one of: `customer-content`, `customer-pii`, `model-output`, `prompt`,
  `trace-with-body`, `provider-ticket`, `redacted-summary`, `other`).
- `ingested_at_utc`.
- `ingested_by` (role + name).
- `source` (where it came from — log tag, trace ID, bridge upload).
- `retention_end_utc` (default per class; superseded by legal hold).
- `legal_hold_ref` (if under hold).
- `access_log` (append-only).

## 3. Minimum access

Base access by data class:

| Data class | Base access (always) | Extended access (by request + audit) |
|-------------|-----------------------|---------------------------------------|
| customer-content | privacy-oncall, legal-oncall, safety-oncall-lead | IC; named responders |
| customer-pii | privacy-oncall, legal-oncall | IC; named responders (minimum necessary) |
| model-output | privacy-oncall, legal-oncall, safety-oncall-lead | IC; AI SRE; eval-engineering |
| prompt | privacy-oncall, legal-oncall | IC; AI SRE; policy-engineering |
| trace-with-body | privacy-oncall, security-engineering-oncall | IC; AI SRE |
| provider-ticket | privacy-oncall, security-engineering-oncall | IC; AI SRE; vendor-management |
| redacted-summary | responder rotation | — |
| other | privacy-oncall | — |

## 4. Retention by class

| Data class | Default retention |
|-------------|-------------------|
| customer-content | 90 days (overridden by legal hold) |
| customer-pii | 30 days (overridden by legal hold) |
| model-output | 180 days |
| prompt | 90 days |
| trace-with-body | 90 days |
| provider-ticket | 24 months |
| redacted-summary | matches the retro's retention (10 years) |

Legal hold supersedes the default; engineering never defaults deletion ahead of a
legal-hold expiry date.

## 5. Export / share

- Shares with third parties (vendor, regulator, customer) are legal's decision, not
  engineering's.
- Every export is logged with the recipient, the legal basis, the artifacts shared,
  and the approver.
- Exports are via the organisation's secure-share channel; email is not a share
  channel.
```

### 2.4 Minimum-necessary access — `docs/incident-command/sensitive-data-discipline/minimum-necessary.md`

```markdown
# Minimum-necessary access during an incident — v2026-11-01

## 1. Principle

Responders have access only to the data they need to resolve the incident. "Need"
is scoped to the role, the data class, and the artifact.

## 2. In-incident grant

- The IC can grant extended access to a named responder at the time of the incident.
- The grant is for the specific artifact(s), not a blanket store access.
- The grant is logged automatically by the store; the IC does not need to file
  separately.
- The grant expires at incident resolve plus 48 hours, unless the responder is the
  retro author or a named reviewer on the retro.

## 3. Retro-time access

- The retro's authors and named reviewers retain access for the retro's authoring
  window.
- Beyond the retro's authoring window, access reverts to base-access per class.
- The retro's own body references artifacts by ID; the retro is not itself in the
  highest-sensitivity class.

## 4. Quarterly access review

- The AI SRE lead, `ai-risk-engineer`, and privacy-oncall jointly review the access
  log and base-access roles each quarter.
- Access anomalies (one role reading many unrelated artifacts, named-responder
  retention past the policy) surface on a weekly digest to the privacy-oncall.
```

### 2.5 Chain-of-custody — `docs/incident-command/sensitive-data-discipline/chain-of-custody.md`

```markdown
# Chain-of-custody — v2026-11-01

## 1. When chain-of-custody matters

Chain-of-custody is activated when:

- The artifact may be shared with a third party (vendor, regulator, customer).
- The artifact is pinned to a legal hold.
- The incident has a serious-incident candidate flag (per `mod-108` chapter 06).

## 2. What chain-of-custody requires

- **Immutable at ingest.** The artifact in the store is byte-identical to the
  source; a cryptographic digest is stored at ingest and verified on every read.
- **No modification.** Derivatives (e.g., a redacted version) are new artifacts with
  a reference to the source. The source is never modified.
- **Logged access.** Every read is logged with role, name, timestamp, and purpose
  (the purpose field is a short free-text; the role + name are mechanical).
- **No deletion.** During the chain-of-custody window, no entry is deleted. Legal
  clears deletion.

## 3. Chain-of-custody header in the retro

When the retro references an artifact under chain-of-custody, the artifact inventory
entry includes:

- Ingest digest (SHA-256).
- Ingest actor (role + name).
- Who read the artifact between ingest and retro close (summary count by role; full
  log in the store).
- Any derivatives made (e.g., redacted-summary for external publication).

## 4. Breaking chain-of-custody

Breaking chain is a reviewable incident in its own right. An accidental modification
of an artifact is reported immediately to the IC; the IC pages legal-oncall.
```

### 2.6 Worked example — IR transcript with annotations

Illustrative. The scenario is a mid-incident on `/advice` where a responder nearly
pastes a customer prompt into the channel. Role-only; all names are role-identifiers.

```markdown
# inc-2026-11-13-advice-classifier-regression — annotated transcript

Channel: `#inc-2026-11-13-advice-classifier-regression`
Opened: 2026-11-13 15:02 UTC
Pinned: data-class card; `art-pkg-2026-11-13-01`; preservation ticket
  `legal-hold-2026-11-13-01`.

---

[15:02] ai-sre-ic-oncall: incident declared SEV-2; safety-oncall paged. Data-class
  card pinned.
  [annotation: correct — IC opens with the pinned reference card.]

[15:03] ai-sre-ic-oncall: controlled-access store is `art-pkg-2026-11-13-01`; preservation
  ticket `legal-hold-2026-11-13-01`. All sample outputs go to the store.
  [annotation: correct — the store ID is in the channel; the content isn't.]

[15:06] safety-engineering-oncall: pulling a sample of the 23 flagged outputs; uploading
  to `art-pkg-2026-11-13-01` now.
  [annotation: correct — safety peer uses the store; names the artifact; does not
   paste content.]

[15:08] ai-engineering-oncall: look at this response — "You should consider
  [paraphrased sensitive health content] ..."
  [annotation: WRONG — model output with health-adjacent content pasted into channel.]

[15:08] ai-sre-ic-oncall: @ai-engineering-oncall — please do not paste model output in
  channel. Move it to the store; reference by ID. I'll flag the paste for rb-sensitive-
  data-in-channel.
  [annotation: correct — IC catches it in-channel, names the behaviour, invokes the
   runbook.]

[15:09] ai-sre-ic-oncall: following rb-sensitive-data-in-channel. Content copied to
  store `art-pkg-2026-11-13-01` item 07; channel message tombstoned; privacy-oncall
  paged.
  [annotation: correct — the message is not deleted; it is tombstoned, the content is
   preserved under the hold, and privacy-oncall is engaged.]

[15:10] privacy-oncall: acknowledged; reviewing the tombstoned entry + item 07.
  [annotation: correct — privacy is in the bridge.]

[15:15] ai-sre-ic-oncall: safety-oncall-lead has extended access to
  art-pkg-2026-11-13-01 items 01-23. Access logged.
  [annotation: correct — minimum-necessary; extended access granted at the IC level,
   logged by the store.]

[15:22] ai-engineering-oncall: user `user-7a3f...` reported to support with a
  screenshot.
  [annotation: BORDERLINE — the user-ID should not be in channel; replaced in the
   tombstone by `cohort-advice-health-adjacent`. See rb-sensitive-data-in-channel.]

[15:25] ai-sre-ic-oncall: mitigation A deployed; moderation strictness raised; see
  art-pkg-2026-11-13-01 item 24 for the before/after.
  [annotation: correct — before/after samples live in the store.]

[16:02] ai-sre-ic-oncall: resolved. Channel archived to the incident folder;
  art-pkg-2026-11-13-01 under preservation; retro assigned to ai-sre-ic-oncall +
  safety-engineering-oncall as co-authors (joint retro per exercise-02). Minimum-
  necessary access reverts to base-access at 18:02 UTC per policy.

---

## Retro back-feed (fixtures)

- **Alert on `#inc-*` channel pastes matching `[A-Z]{2,}\d{3,}` or known PII patterns**
  — automatic tombstone proposal to the IC. Owner: `platform-engineering`.
- **Pre-write hook in the incident bot that reminds responders of the data-class
  card on first post** — gentle nudge, not a block. Owner: `platform-engineering`.
- **Add `user-id-in-channel` to the quarterly access-review digest** — a small signal
  that the pattern is leaking into channels. Owner: `privacy-oncall`.
```

## 3. Validation steps

1. **Channel-paste guardrail.** Deploy a bot that scans incident channels for
   patterns matching likely PII (e.g., card numbers, SSN shapes, email addresses) and
   for substrings matching tenant-prefixed user IDs. The bot proposes tombstones to
   the IC (does not auto-delete). Validation: quarterly, inject ten synthetic sensitive
   lines into a scripted channel and confirm the bot proposes a tombstone within 30s.

2. **Store ingest + hash.** Every store ingest records a SHA-256; every read verifies
   it. Automated: a nightly job samples 1% of stored artifacts and verifies hashes.
   Any mismatch is an incident.

3. **Access-log completeness.** Weekly: a monitor compares the store's audit entries
   to the role-and-name attempts at the store's API. Any API call without an audit
   entry is an incident.

4. **Minimum-necessary review.** Quarterly: the AI SRE lead + `ai-risk-engineer` +
   privacy-oncall review the access log; anomalies escalate to the quarterly access
   review.

5. **Chain-of-custody drill.** Semi-annual: an auditor requests a specific artifact
   under chain-of-custody. The AI SRE can demonstrate (a) the ingest digest, (b) the
   read log, (c) the derivative lineage, (d) the retention / hold status.

6. **Runbook `rb-sensitive-data-in-channel` drill.** Semi-annual: a scripted paste
   drill in a staging channel. The IC should invoke the runbook within 60s; the paste
   should be tombstoned and migrated within 5 minutes.

7. **Retro cross-check with exercise-02.** Every joint retro is reviewed by the
   composition reviewer for channel-paste incidents in the transcript; any
   un-tombstoned paste is flagged.

8. **Data-class card freshness.** Reviewed quarterly by privacy, legal, AI SRE,
   security-engineering. Expired card (older than 180 days without an attested review)
   is a compliance ticket.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at >= 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Data-class reference card exists + pinned *(hard gate)* | Missing | Exists, not pinned | Exists, pinned on every incident channel open |
| 2 | Channel vs. store discipline *(hard gate)* | Customer content pasted in channels | Occasionally pasted | Channel references artifacts by ID; content is in the store |
| 3 | Controlled-access store with per-access log *(hard gate)* | None | Role-gated only | Role-gated + per-access log + object-lock |
| 4 | Minimum-necessary policy enforced *(hard gate)* | Blanket responder access | Role-based | Role-based + named grants + expiry |
| 5 | Chain-of-custody procedure | Missing | Present as prose | Present + digest + derivative lineage + drilled |
| 6 | Retention mapped to data class | Single window | Partially mapped | Per-class retention + legal-hold override |
| 7 | Accidental-paste runbook | Missing | Exists but delete-first | Tombstone-first + migrate + legal / privacy page |
| 8 | Guardrail bot | Missing | Logging only | Proposes tombstones; quarterly drill |
| 9 | Vocabulary consistent with mod-108 chapter 02 / 06 | Divergent | Partial | Reuses mod-108 vocabulary and the chapter-06 preservation shape |
| 10 | Quarterly access review attested | Missing | Ad-hoc | Quarterly attestation stored in the compliance store (see exercise-03) |

## 5. Common mistakes

- **Pasting a "safe-looking" prompt in channel.** Prompts can contain PII the user
  included (names, addresses, health details). Default to the store.
- **Deleting a channel message that contained sensitive content.** Deletion is
  destruction. Tombstone and migrate; preserve under the legal-hold policy.
- **Treating the controlled-access store as a dumping ground.** Classify on ingest;
  retention and base access depend on the class.
- **Granting blanket responder access to the store at IC-declare time.** That is not
  minimum necessary. Grant per-artifact on request.
- **Logging the paste but not the tombstone.** Both are evidence; both are needed
  for the retro's chain-of-custody.
- **Sharing a sample with the vendor via a chat DM.** Not a share channel. Use the
  organisation's secure-share; log the export with the legal basis.
- **Skipping the privacy page.** Any accidental in-channel exposure of sensitive data
  pages privacy-oncall, independent of incident severity. The IC does not decide
  whether to page privacy; the runbook does.
- **Letting the retro author operate on raw artifacts without named-reviewer
  co-access.** Single-reviewer on sensitive-content retros is a concentration risk;
  the joint retro template (exercise-02) requires co-authorship.
- **Keeping extended access past the policy.** Access that was legitimate during the
  incident is not legitimate at retro-plus-N. Automated expiry prevents this.
- **Pasting classifier labels (e.g., `self-harm=true`) in channel.** The label plus
  the user ID is a safety-flag-plus-identifier combination; keep it in the store.
- **Assuming the vendor's incident ID is sensitive.** It is a provider-side
  identifier, not customer content. It can be in channel; the correspondence itself
  (with sample content) is in the store under `provider-ticket` class.

## 6. References

- Module chapter on sensitive-data handling during AI incidents (this module) — the
  classification-first discipline, the channel standard, the controlled-access
  shape, the minimum-necessary policy, and the chain-of-custody procedure that this
  exercise implements.
- `mod-108` chapter 02 §5.3, *Incident Command for AI Systems* — the channel-
  discipline rule (no customer content in-channel) that this exercise extends.
- `mod-108` chapter 06 §5, *Legal / Compliance Handoff* — the preservation and
  legal-hold procedure that this exercise aligns with.
- `mod-108` chapter 03 §7, *Blameless Post-Mortems* — the "no deletion to clean up"
  discipline that applies to tombstoned channel content.
- `mod-110` exercise-02, *Joint Post-Mortem with the Security Peer* — the joint
  retro's sensitive-data preservation requirements cross-reference this exercise.
- `mod-110` exercise-03, *Compliance Telemetry Hooks* — the compliance log store
  where access-log attestations and quarterly review attestations land.
- NIST AI Risk Management Framework,
  [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the Govern / Measure functions that align with the minimum-necessary and
  chain-of-custody disciplines.
- NIST AI 600-1, *Generative AI Profile*,
  [nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
  — GAI-specific risks (data exposure, confidentiality) that the store's retention
  and class policy operationalises.
- OWASP Machine Learning Security Top 10,
  [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — ML03 (model inversion) and ML04 (membership inference) classes inform why
  response content is kept under access control even post-incident.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — the exfil tactic
  vocabulary used by the store's security-engineering signals.
- EU AI Act, Reg. (EU) 2024/1689 — Art. 12 (record-keeping) and Art. 10 (data
  governance) inform the retention and access shape here for in-scope high-risk
  systems.
- GDPR (Regulation (EU) 2016/679) — Art. 32 (security of processing) informs the
  encryption-at-rest, access-logging, and minimum-necessary posture for any
  incident data that contains personal data under GDPR's scope.
