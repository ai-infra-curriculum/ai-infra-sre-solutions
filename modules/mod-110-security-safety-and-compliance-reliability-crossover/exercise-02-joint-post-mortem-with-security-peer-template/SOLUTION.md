# Exercise 02 — Joint Post-Mortem with the Security Peer: the Template — Reference Solution

Paired learning file: `lessons/mod-110-security-safety-and-compliance-reliability-crossover/exercises/exercise-02-joint-post-mortem-with-security-peer-template.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below
> is derived from this module's chapter on joint retrospectives with the security /
> safety peer. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

> **Scope boundary.** This template is the *AI-SRE-plus-security-peer* retrospective for
> incidents where the proximate cause, the contributing factors, or the response spans
> both functions. It **extends** — it does not replace — `mod-108`'s blameless
> post-mortem template. The same blameless-tone discipline applies; the additions here
> are co-authorship structure, the dual-root-cause walk, the joint-fixture section, and
> the security-peer-specific disclosure gate.

## 1. Solution overview

The deliverable is a committed template file `templates/joint-post-mortem.md` plus a
decision-rights addendum `templates/joint-post-mortem-decision-rights.md` and a worked
example `examples/jpm-illustrative-prompt-injection-slo-burn.md`. The template extends
`mod-108` chapter 03 with:

1. **Dual authorship header** — explicit AI SRE and security peer co-authors, plus a
   role-only distribution list.
2. **Dual-category root-cause walk** — the incident gets a *reliability category* (from
   the `mod-108` controlled vocabulary) and a *security / safety category* (from the
   OWASP ML Top 10 / MITRE ATLAS vocabularies). Many incidents warrant one of each.
3. **Joint fixture section** — fixtures are tagged with their ownership function and
   validated by both sides. A joint incident requires at least one fixture that lands in
   the security / safety control plane and at least one that lands in the reliability
   control plane, unless one side formally attests none is applicable.
4. **Disclosure gate** — the retro's external-publication path is gated by the security
   peer's disclosure review (which may defer to legal; see `mod-108` chapter 06).
5. **Decision-rights addendum** — the RACI matrix for who owns which post-mortem
   decisions during authoring and after publication.

The retro remains blameless. "Blameless" in a joint retro also means the two functions
do not blame each other; see §5.

## 2. Implementation — worked answer

### 2.1 Template — `templates/joint-post-mortem.md`

```markdown
---
template_version: 2026-11-01
extends: templates/post-mortem.md  # mod-108 chapter 03
owner: ai-sre-function + security-engineering-function
---

# Joint incident post-mortem — `<incident-id>` — `<one-line title>`

## Header

- **Incident ID:** `<inc-YYYY-MM-DD-rb-slug-surface>`
- **Severity (final):** `<SEV-1 | SEV-2 | SEV-3>`
- **Start / detect / declare / resolve (UTC):** `<HH:MM | HH:MM | HH:MM | HH:MM>`
- **TTD / TTM / TTR:** `<mm:ss | HH:MM | HH:MM>`
- **Co-authors (required, both):**
  - **AI SRE lead:** `<ai-sre-ic-oncall>` (reliability narrative, SLO impact,
    runbook / telemetry fixtures)
  - **Security / safety lead:** `<security-engineering-oncall | safety-engineering-oncall>`
    (threat-model narrative, control impact, policy / detection fixtures)
- **Composition reviewer (required if categories span):** `<ai-sre-reviewer>` — the
  reviewer whose role is to keep the narrative coherent across both halves.
- **Distribution:** role-only; see §Decision rights below.
- **Legal / compliance engagement:** `<triggered | not triggered | ongoing>` (see
  `mod-108` chapter 06 trigger card).

## Summary

One paragraph. Must stand alone: what happened, scope, both root-cause categories
(reliability and security / safety if applicable), mitigation that resolved it, the one
fixture the retro commits to from each side. No more than 180 words (slightly more than
the `mod-108` ceiling, to accommodate the second category).

## Impact

(Same as `mod-108` with an explicit safety / security row.)

- **User-visible impact:** `<who experienced what, with numbers>`
- **SLO impact:** `<error-budget burn by SLO, named>`
- **Revenue / contractual impact:** `<quantified where measurable>`
- **Safety / policy impact:** `<classifier events; policy-class counts; see chapter N>`
- **Security impact:** `<control defeated, data class exposed, scope, confidence>`
- **Data / privacy impact:** `<if applicable>`

## Timeline

(Same shape as `mod-108`: flat chronological, facts only, actor-role voice. See §4
decision rights for who contributes.)

## Dual root-cause analysis

### Primary reliability cause

**Category (`mod-108` controlled vocabulary):** one of `model-weight | kernel | scheduler |
prompt-config | retrieval | provider | agent-cascade | eval-judge-drift |
unknown-multi-class`.

**Specific instance:** one sentence.

### Primary security / safety cause

**Category:** one of the entries in the vocabulary below (OWASP ML Top 10 + MITRE ATLAS
tactics + the module's safety-class vocabulary).

| Vocabulary | Slug | Covers |
|------------|------|--------|
| OWASP ML Top 10 | `ml01-input-manipulation` | Adversarial / prompt-injection input attacks. |
| OWASP ML Top 10 | `ml02-data-poisoning` | Training / fine-tune / RLHF-set contamination. |
| OWASP ML Top 10 | `ml03-model-inversion` | Reconstruction of training data from queries. |
| OWASP ML Top 10 | `ml04-membership-inference` | Training-membership deduction. |
| OWASP ML Top 10 | `ml05-model-theft` | Model extraction through query access. |
| OWASP ML Top 10 | `ml06-supply-chain` | Compromised model / weight / dependency. |
| OWASP ML Top 10 | `ml07-transfer-learning` | Attacks across a transfer-learning handoff. |
| OWASP ML Top 10 | `ml08-model-skewing` | Deliberate skew of model predictions. |
| OWASP ML Top 10 | `ml09-output-integrity` | Tamper of model output in transit / at rest. |
| OWASP ML Top 10 | `ml10-neural-net-poisoning` | Weight-level backdoor / trojan. |
| MITRE ATLAS tactic | `atlas-recon` | Model reconnaissance. |
| MITRE ATLAS tactic | `atlas-resource-dev` | Build an adversary's capability. |
| MITRE ATLAS tactic | `atlas-initial-access` | Gain access to the ML system. |
| MITRE ATLAS tactic | `atlas-ml-attack-staging` | Prepare the attack on the ML model. |
| MITRE ATLAS tactic | `atlas-exfil` | Exfiltrate data / model / weights. |
| MITRE ATLAS tactic | `atlas-impact` | Degrade / cause harm to the ML system. |
| Safety-class | `safety-content-policy` | Output violating a safety-content policy
  (e.g., self-harm, CSAM, CBRN) without an underlying adversarial-attack trigger. |
| Safety-class | `safety-policy-bypass` | A safety control was bypassed (jailbreak,
  policy-change regression). |
| Compliance | `compliance-obligation` | Breach of a regulatory / contractual obligation
  (data-residency, ZDR, retention, auditability). |
| None | `none-security-adjacent` | The incident has no security / safety primary cause;
  the reliability narrative is sufficient. |

**Specific instance:** one sentence.

### Contributing factors (joint)

Each contributing factor is tagged with its owning function:

- **[reliability]** `<fact>`. **System-shape question:** `<question>`.
- **[security]** `<fact>`. **System-shape question:** `<question>`.
- **[joint]** `<fact>`. **System-shape question:** `<question>`.

### Five-whys walk (joint)

If the root cause is chained across functions, the walk is written jointly with a brief
annotation on each step indicating which function's lens it is through.

## What went well

- **Reliability controls that held:** `<fallback, canary, breaker>`.
- **Security controls that held:** `<detection, rate-shape, policy class>`.
- **Joint response that worked:** `<e.g., the on-call bridge pulled in security within 3
  minutes; the preservation discipline kept customer content out of the open channel>`.

## What went poorly

- **Reliability detection / response gaps.**
- **Security detection / response gaps.**
- **Joint coordination gaps** (e.g., paging escalation, shared terminology, divergent
  runbooks).

## Action items

| ID | Description | Owner function | Owner role | Due | Priority | Fixture type | Verification |
|----|-------------|----------------|------------|-----|----------|--------------|--------------|
| AI-1 | `<action>` | `reliability | security | joint` | `<role>` | YYYY-MM-DD | P0/P1/P2 | `<fixture class>` | `<how we will know it landed>` |

## Fixture back-feed (mandatory, dual-sided)

A joint incident requires at least one fixture that lands in the **security / safety
control plane** and at least one that lands in the **reliability control plane**, unless
one side formally attests no fixture is applicable. The attestation is itself a
reviewable artifact.

```yaml
fixtures_reliability:
  - kind: alert | canary_eval_case | eval_set_entry | load_test_shape | chaos_experiment | runbook_update
    system: <where it lives>
    name: <stable identifier>
    description: <one line>
    owner: <role>
    verification: <how we proved it would have caught this>

fixtures_security:
  - kind: detection_rule | policy_class | policy_corpus_entry | redteam_fixture | abuse_signature | audit_rule
    system: <where it lives>
    name: <stable identifier>
    description: <one line>
    owner: <role>
    verification: <how we proved it would have caught this>
```

If either list is empty, the retro contains an **attestation paragraph** from the
corresponding function's lead explaining why no fixture is appropriate and what
alternative ownership was accepted.

## Disclosure gate (if external publication is planned)

External publication of this retro — blog, incident write-up, status-page retrospective,
customer notification letter — requires:

1. **Security peer disclosure review.** The security peer reviews the retro for
   exploitable detail: specific bypass payloads, specific defeated detection rules,
   attribution of attacker capability, timelines that advertise vulnerable windows.
   Specific redactions are negotiated per `mod-108` chapter 03 §7 (not blanket).
2. **Legal / compliance review.** Per `mod-108` chapter 06 triggers.
3. **AI SRE reviewer.** Confirms the technical narrative survives the redactions and
   still teaches.
4. **External-comms owner sign-off.** The external-comms function owns the final
   publication; neither the AI SRE nor the security peer publishes directly.

Internal distribution is **not** gated by (1)–(4); the retro is committed to the
internal retro repository at draft and the dual authors have narrated it to the on-call
rotations within 48h per `mod-108` discipline.

## Related incidents and prior art

- **Repeat events:** `<inc-...>` (same security / safety category); `<inc-...>` (same
  reliability category).
- **Related runbooks / chapters:** `rb-<slug>` (both halves).
- **External references:** `<public post-mortems that match the shape — carefully
  summarised, not reproduced verbatim>`.

## Appendix

- Trace IDs, Grafana / Honeycomb pins, SIEM queries pinned to the window.
- Threat model excerpt (if relevant).
- SME write-ups (one per function).
- Scribe raw timeline.
- Bridge-call notes.
```

### 2.2 Decision rights addendum — `templates/joint-post-mortem-decision-rights.md`

```markdown
# Joint post-mortem — decision rights (RACI)

| Decision | AI SRE lead | Security peer lead | Composition reviewer | Legal | Exec |
|----------|-------------|---------------------|------------------------|-------|------|
| Co-author composition | R | R | A | — | — |
| Timeline facts | R | R | A | — | — |
| Reliability root-cause classification | A | C | R | — | — |
| Security / safety root-cause classification | C | A | R | — | — |
| Reliability fixtures | A | C | C | — | — |
| Security / safety fixtures | C | A | C | — | — |
| Internal distribution | R | R | A | I | I |
| External publication | C | C | C | A | A |
| Redactions in an external version | C | A | C | A | C |
| Public attribution of an attacker | — | C | — | A | A |
| Customer notification | C | C | — | R | A |

R = responsible, A = accountable (single owner), C = consulted, I = informed.

Note: the AI SRE is **consulted** on security-category decisions and vice versa; neither
function unilaterally sets the other's root-cause category. The composition reviewer is
the backstop who holds the narrative's coherence across both halves.
```

### 2.3 Worked example — `examples/jpm-illustrative-prompt-injection-slo-burn.md`

Written in the voice the committed document would use. The scenario is fictional.

```markdown
# Joint incident post-mortem — inc-2026-11-07-prompt-injection-advice — Prompt-injection in `/advice` triggered a classifier-latency regression and a safety-policy bypass

## Header

- **Incident ID:** inc-2026-11-07-prompt-injection-advice
- **Severity (final):** SEV-2
- **Start / detect / declare / resolve (UTC):** 09:22 / 09:27 / 09:30 / 10:14
- **TTD / TTM / TTR:** 05:00 / 00:33 / 00:52
- **Co-authors:**
  - **AI SRE lead:** ai-sre-ic-oncall
  - **Security lead:** security-engineering-oncall
- **Composition reviewer:** ai-sre-reviewer
- **Distribution:** `#inc-channel`, reliability rotation, security rotation, safety rotation
- **Legal / compliance engagement:** not triggered (no serious-harm threshold, no data
  exfil confirmed; `mod-108` chapter 06 trigger card consulted).

## Summary

At 09:22 UTC the input-scan stage of the `/advice` guardrail sidecar began receiving a
new class of prompt-injection payload that exploited a template-rendering interaction
with the user-provided `persona` field. The payloads simultaneously (a) escalated the
input-scan classifier's latency 3× by inducing long tokenizations and (b) induced the
policy class `safety-policy-bypass` on roughly 0.6% of responses. The sidecar entered
degraded mode at 09:27 under the fast-burn SLO alert; the security peer pushed a
detection rule and a policy-corpus entry by 09:44; the input-template was hardened at
10:05; the sidecar exited degraded mode at 10:14. One fixture lands in each control
plane: a token-budget cap on `persona` plus a detection rule with corpus coverage.

## Impact

- **User-visible impact.** 28.4k sessions on `/advice`; 171 confirmed policy-bypass
  outputs delivered (classifier pattern-match + manual review).
- **SLO impact.** `/advice` latency SLO 28-day budget burned 3.1% (fast-burn alert fired
  at 09:27; see Grafana panel link).
- **Safety / policy impact.** 171 bypass outputs; sampled categories: policy-adjacent
  advice (139) + explicit policy-bypass (32). All 171 preserved under
  `legal-hold-2026-11-07-01` in the controlled-access store.
- **Security impact.** Control defeated: the input-scan tokenization pre-filter. Data
  class exposed: user-generated content (no customer PII pattern matched in the
  payloads; confidence high based on classifier scan).
- **Data / privacy impact.** None identified in the window.

## Timeline

| Time (UTC) | Actor (role) | Event type | Description |
|------------|--------------|------------|-------------|
| 09:22 | — | signal | `guardrail:stage_latency_seconds:p95{stage=input-scan}` 320 ms → climbing. |
| 09:25 | — | signal | `guardrail_total_latency_seconds_p95` crosses 400 ms. |
| 09:27 | pagerduty | signal | GuardrailSidecarSLOFastBurn fires. |
| 09:28 | ai-sre-ic-oncall | decision | SEV-3 declared; `#inc-...` opened. |
| 09:28 | ai-sre-ic-oncall | decision | Security-oncall paged per the module's joint-retro trigger. |
| 09:30 | — | signal | `safety_policy_bypass_rate{route=/advice}` crosses 10× 7d baseline. |
| 09:30 | ai-sre-ic-oncall | decision | SEV-3 → SEV-2 on safety signal. |
| 09:33 | — | signal | Sidecar enters `guardrail_degraded_mode_active=1`. |
| 09:38 | security-engineering-oncall | mitigation | Threat-model pattern identified: `persona`-field-triggered re-tokenization. |
| 09:44 | security-engineering-oncall | mitigation | Detection rule `sec-prompt-injection-persona-retok` deployed to SIEM. |
| 09:50 | security-engineering-oncall | mitigation | Policy-corpus entry pushed; redteam fixture queued. |
| 10:00 | ai-sre-ic-oncall | mitigation | `persona` field token budget cap deployed (soft 64 → hard 32). |
| 10:05 | ai-engineering-oncall | mitigation | Input-template hardened to escape `persona` content. |
| 10:10 | — | signal | `safety_policy_bypass_rate` returns to baseline. |
| 10:12 | — | signal | `guardrail:stage_latency_seconds:p95{stage=input-scan}` returns to 145 ms. |
| 10:14 | ai-sre-ic-oncall | decision | Degraded mode exited; incident resolved. |

## Dual root-cause analysis

### Primary reliability cause

**Category:** `prompt-config`.

**Specific instance:** The input-template rendered the user-supplied `persona` field
without a token-budget cap, allowing adversarial content to inflate tokenization time
and compound input-scan latency.

### Primary security / safety cause

**Category:** `ml01-input-manipulation` (OWASP ML Top 10) and `safety-policy-bypass`
(safety-class), with `atlas-ml-attack-staging` as the matching ATLAS tactic.

**Specific instance:** A deliberately-crafted `persona` field (a) triggered a
tokenization long-tail that exceeded the input-scan stage budget, and (b) under degraded
mode, the sidecar's classifier fallback did not cover the bypass's policy class on
`/advice`.

### Contributing factors (joint)

- **[reliability]** The `persona` field had no token-budget cap. **System-shape
  question:** our template inputs are user-supplied without a mechanical bound; the
  sidecar budget assumed bounded input.
- **[security]** The sidecar's classifier-fallback pattern coverage did not include the
  `persona`-triggered re-tokenization payload class. **System-shape question:** our
  classifier-fallback corpus is behind the detection rules; the policy-corpus and the
  classifier-fallback need a shared source of truth.
- **[joint]** Degraded-mode onset caused the bypass rate to spike. **System-shape
  question:** our degraded-mode downshift changes both stacks' behaviour simultaneously;
  the joint reliability/security review of the downshift matrix was last updated 11
  months ago.
- **[reliability]** Degraded-mode entry did not trigger a safety-peer page; only the
  reliability on-call was paged. **System-shape question:** our paging rules for degraded
  mode do not fan out to the safety peer.

### Five-whys walk (joint)

1. [reliability] Why did the SLO burn? Input-scan latency climbed. — Classifier scaled
   with input length; new payload class inflated length.
2. [security] Why could the payload inflate length? `persona` was unbounded. Cap missing.
3. [security] Why was the bypass successful under degraded mode? The classifier-fallback
   corpus did not have a pattern for the `persona`-retok class. Corpus was behind the
   detection rule set.
4. [joint] Why were the two corpora out of sync? The policy-corpus (safety peer) and the
   classifier-fallback corpus (AI SRE) are separate stores without a sync job.
5. [joint] Why were they separate? Historical — the classifier-fallback was added during
   the sidecar's reliability work; the policy-corpus grew under the safety peer. (This
   is the joint fixture.)

## What went well

- **Reliability controls that held.** Fast-burn alert fired on time (5 min after start).
  Degraded-mode entry prevented SLO breach beyond 3.1%.
- **Security controls that held.** The detection rule was deployable within 16 min of
  the pattern being identified; the policy-corpus entry followed within 22 min.
- **Joint response.** The dual page at SEV-3 declare meant security was in the bridge
  within 2 minutes of the AI SRE IC's decision. The preservation discipline (controlled-
  access store for the 171 bypass outputs) held on both sides.

## What went poorly

- **Reliability.** The `persona` field had no cap. The cap is a one-line config change;
  the absence of the cap is a surface-shape failure.
- **Security.** The classifier-fallback corpus was not reviewed against the policy-corpus
  on a schedule. The security peer thought the AI SRE kept them in sync; the AI SRE
  thought the security peer owned the classifier-fallback corpus.
- **Joint.** Degraded-mode entry did not page the safety peer; the audit digest is
  daily, which is too slow for an active bypass. This is the joint fixture (page on
  degraded mode + sync job between the two corpora).

## Action items

| ID | Description | Function | Role | Due | Priority | Fixture type | Verification |
|----|-------------|----------|------|-----|----------|--------------|--------------|
| J-1 | Cap `persona` token budget; CI check blocks regressions. | reliability | ai-engineering-oncall | 2026-11-18 | P0 | ci-gate | PR replay against the 2026-11-07 payload corpus; input-scan latency stays flat. |
| J-2 | Add detection rule `sec-prompt-injection-persona-retok` to standing SIEM rule set + corpus. | security | security-engineering-oncall | 2026-11-18 | P0 | detection_rule + policy_corpus_entry | Replay the 2026-11-07 payloads through SIEM; rule fires. |
| J-3 | Sync job between policy-corpus and classifier-fallback corpus; nightly diff + alert on divergence. | joint | ai-sre-oncall + safety-engineering-oncall | 2026-11-25 | P1 | audit_rule | Divergence > 0 fails CI on either repo. |
| J-4 | Page safety peer on degraded-mode entry (not daily digest). | joint | ai-sre-oncall | 2026-11-18 | P1 | alert | Alert test with a synthetic degraded-mode gauge. |
| J-5 | Redteam fixture against the `persona`-retok class; add to the standing redteam suite. | security | redteam-engineering | 2026-12-02 | P1 | redteam_fixture | Suite blocks on fixture regression. |

## Fixture back-feed (dual-sided)

```yaml
fixtures_reliability:
  - kind: ci-gate
    system: chatcat-input-templates
    name: persona-token-budget-cap
    description: Static cap + CI diff on persona-field length; blocks template PRs that lift the cap.
    owner: ai-engineering-oncall
    verification: Replay 2026-11-07 payloads; input-scan latency stays under 150 ms p95.

  - kind: alert
    system: prometheus
    name: GuardrailDegradedModePagesSafetyPeer
    description: Degraded-mode entry pages the safety peer in addition to the reliability oncall.
    owner: ai-sre-oncall
    verification: Alert test with a synthetic degraded-mode gauge.

fixtures_security:
  - kind: detection_rule
    system: siem
    name: sec-prompt-injection-persona-retok
    description: Detection rule matching the 2026-11-07 payload family.
    owner: security-engineering-oncall
    verification: Replay 2026-11-07 payloads through SIEM; rule fires with precision >= 0.98 on the corpus.

  - kind: policy_corpus_entry
    system: policy-corpus
    name: persona-retok-pattern-family
    description: Policy corpus entry covering the payload family; auto-synced to classifier-fallback corpus.
    owner: safety-engineering-oncall
    verification: Classifier-fallback corpus contains the family after nightly sync.

  - kind: redteam_fixture
    system: redteam-suite
    name: persona-retok-family
    description: Red-team scenario reproducing the 2026-11-07 payload family.
    owner: redteam-engineering
    verification: Suite CI blocks on regression.
```

## Disclosure gate

- External publication requested by product-marketing for a customer-trust blog post.
- Status: the security peer reviewed the draft and redacted (a) the specific payload
  structure and (b) the stage-level budget numbers. The technical narrative survives the
  redactions.
- Legal engagement: not triggered (no serious-harm threshold, no customer-data exfil).
- External-comms owner sign-off: pending.

## Related incidents and prior art

- **Repeat events.** inc-2026-04-19-prompt-injection-template-field (same OWASP ML01
  class, different field).
- **Related runbooks / chapters.** `rb-guardrail-degraded-mode`, `rb-prompt-injection-
  containment`; this module's chapter on guardrails + chapter on joint retros.
- **External references.** OWASP ML Top 10 ML01 entry; MITRE ATLAS
  `atlas-ml-attack-staging` tactic page.
```

## 3. Validation steps

1. **Template compliance.** Automated: the retro has the sections `Header`, `Summary`,
   `Impact`, `Timeline`, `Dual root-cause analysis`, `What went well`, `What went
   poorly`, `Action items`, `Fixture back-feed`, `Disclosure gate`, `Related incidents`.

   ```bash
   REQUIRED=('## Header' '## Summary' '## Impact' '## Timeline' \
             '## Dual root-cause analysis' '## What went well' '## What went poorly' \
             '## Action items' '## Fixture back-feed' '## Disclosure gate' \
             '## Related incidents and prior art')
   for f in postmortems/joint/*.md; do
     for h in "${REQUIRED[@]}"; do
       grep -qF -- "$h" "$f" || { echo "FAIL: $f missing $h"; exit 1; }
     done
   done
   ```

2. **Co-authorship check (hard gate).** Automated: the header must list both an AI SRE
   lead and a security / safety lead; a single author on a joint retro fails. A
   composition reviewer must be named.

3. **Dual-category check.** Automated: if the reliability category is non-trivial, the
   security / safety category must be either populated from the controlled vocabulary or
   explicitly set to `none-security-adjacent` with a one-line justification.

4. **Dual-fixture check.** Automated: both `fixtures_reliability` and `fixtures_security`
   are non-empty, OR either list is empty with an attestation paragraph from the
   corresponding function's lead.

5. **Decision-rights addendum coherence.** Human: an external-publication decision and
   a public-attribution decision must each have a single accountable role per the RACI;
   any contradiction with the chosen co-authorship is a reviewer flag.

6. **Blameless cross-function tone.** Human: read for "security dropped the ball" or
   "reliability ignored the detection." Both are proximate-human blame across functions;
   the system-shape voice applies across the boundary as much as within a function.

7. **Preservation / sensitive-data discipline.** Cross-check with exercise-05:
   customer content referenced in the retro is in the controlled-access store, not in
   the retro body; the retro references artifacts by ID.

8. **Semi-annual drill.** The dual on-call rotations walk through a scripted joint
   incident using the template; the composition reviewer times authoring to 48h
   per `mod-108` discipline.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at >= 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Dual authorship *(hard gate)* | Single author on a joint incident | Both named but composition reviewer absent | Both named + composition reviewer |
| 2 | Dual root-cause from controlled vocabularies *(hard gate)* | Missing a category | One category, no justification for absence | Both categories or explicit `none-security-adjacent` justification |
| 3 | Contributing factors tagged by function | Untagged | Partial | Every factor tagged `[reliability]` / `[security]` / `[joint]` |
| 4 | Dual-sided fixture back-feed *(hard gate)* | One list missing, no attestation | Both present but vague | Both non-empty with owner + verification, or attestation paragraph |
| 5 | Action items have function + owner + due + priority + fixture type | Missing fields | Partial | All fields on every row |
| 6 | Blameless tone across both functions *(hard gate)* | Cross-function blame present | Occasional "security should have" | System-shape voice across the boundary |
| 7 | Decision-rights addendum used | Missing | Referenced | RACI consulted and reconciled with co-authorship choices |
| 8 | Disclosure gate operated | Missing | Mentioned | Security review + legal check + redaction call + external-comms sign-off |
| 9 | Sensitive-data discipline | Customer content pasted in retro | Mixed | Content in controlled-access store; retro references by ID |
| 10 | Timeline voice | Interpretation mixed in | Facts but inconsistent roles | Facts only, role voice, scribed |

## 5. Common mistakes

- **Letting one function narrate for the other.** The AI SRE writes a security
  classification; the security peer writes a reliability classification. Each should be
  consulted — not the author of — the other's category.
- **A joint retro with a single author.** A tell that the joint-ness was procedural
  only. The composition reviewer catches this at the header.
- **Cross-function blame.** "If security had a better detection rule..." / "If the
  sidecar's budget were tighter..." — both are system-shape questions to the author's own
  function in disguise. Blameless applies across the boundary.
- **Fixture asymmetry without attestation.** A joint retro with four reliability
  fixtures and zero security fixtures (or vice versa) without an attestation paragraph
  is incomplete. The attestation itself is a reviewable artifact.
- **Combining the joint retro with the external blog post.** The retro is internal; the
  blog is external. They share facts but not form. The disclosure gate is the hand-off,
  not a merge.
- **Publishing payload specifics externally.** The security peer's redaction is not
  blanket (per `mod-108` chapter 03 §7); it is specific and negotiated. But *do* redact
  exploitable detail.
- **Attributing an attacker.** Attribution is a legal / executive decision, not an
  engineering one. The retro's engineering body does not attribute; the RACI puts
  attribution on legal + exec.
- **Two root-cause categories but no `[joint]` contributing factors.** Most joint
  incidents have at least one factor that is irreducible to either function alone — the
  interface between them. If the retro has none, the dual root-cause walk probably
  short-changed the interface.
- **Treating `none-security-adjacent` as a convenience escape.** The category requires a
  one-line justification; "no security angle" is not a justification by itself.
- **Letting the fast-burn SLO alert be the only paging path.** A joint incident often
  starts on one side and spreads. The sidecar's degraded-mode alert, the SIEM's
  bypass-rate alert, and the retro's own after-the-fact classification should each be
  able to engage the other function on the first page.

## 6. References

- Module chapter on joint retrospectives with the security / safety peer (this module)
  — the dual-authorship discipline, the vocabulary bridging, the dual-fixture rule, and
  the disclosure gate.
- `mod-108` chapter 03, *Blameless Post-Mortems for AI Systems* — the parent template
  this one extends, the controlled vocabulary for reliability root-cause, and the
  co-authorship rule for multi-category incidents (which this template operationalises
  across functions).
- `mod-108` chapter 06, *Legal / Compliance / Regulator-Facing Handoff* — the trigger
  card referenced by the Disclosure gate and the preservation / legal-hold discipline
  that the joint retro relies on when customer content is in scope.
- `mod-108` chapter 05, *Regression-Fixture Back-Feed Discipline* — the reliability
  fixture-class vocabulary extended here with the security fixture classes.
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — the ML01–ML10 vocabulary that the security root-cause category uses.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — the tactic and technique
  vocabulary used as the ATLAS lens in the security root-cause category.
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the Measure / Manage functions the joint retro's action items operationalise.
- NIST AI 600-1, *Generative AI Profile*, [nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
  — the GAI-specific action vocabulary that maps to the `safety-*` and `compliance-*`
  slugs.
- Google SRE book, *Postmortem Culture: Learning from Failure* chapter
  ([sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/))
  and John Allspaw, *Blameless PostMortems and a Just Culture* ([codeascraft.com](https://www.codeascraft.com/2012/05/22/blameless-postmortems/))
  — the foundational blameless premise that this template extends across the
  cross-function boundary.
