# Exercise 02 — Incident Command for AI Systems: Role and Cadence — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-02-incident-command-for-ai-systems-role-and-cadence.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below is
> derived from chapter 02 of the module (which the exercise is explicitly paired to) and chapter
> 02 §8's worksheet. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

## 1. Solution overview

The exercise asks the learner to produce the AI-flavoured version of the incident-command
operating document that chapter 02 of this module describes: a **roles chart**, an **incident
channel operating standard**, a **posting-cadence template**, a **handoff note shape**, and a
**pre-committed safety-and-compliance decision tree** (chapter 02 §6). The deliverable fits in a
`docs/incident-command/` directory and is consumable by any on-call engineer during an incident
without pre-explanation.

The solution below provides the five artifacts for the same `chatcat` illustrative product used
in exercise 01. The product is fictional; the roles vocabulary, decision-tree triggers, and
cadence rules are drawn directly from chapter 02 and are portable.

## 2. Implementation — worked artifacts

### 2.1 Roles chart — who does what, by function (not by name)

| Role | One-line duty | Who holds it by default |
|------|----------------|-----|
| **Incident Commander (IC)** | Owns the room; maintains the timeline; assigns tasks; decides severity; approves blast-radius mitigations; calls executive / legal / PR escalation; declares stable / resolved; hands off retro ownership. Does not debug. | `ai-sre-ic-oncall` (weekly rotation independent of primary on-call). |
| **Operations Lead (OL)** | Drives hands-on mitigation on the primary thread; executes the runbook's confirm checklist; reports state to the IC. | On-call engineer on the owning peer rotation (`ai-infra-inference-engineer`, `ai-eval-engineer`, `ai-engineering`, etc.). For the first 30 min, the AI SRE on-call is OL and may hand off once the class is confirmed. |
| **Subject-Matter Experts (SMEs)** | Named engineers on named peer rotations who consult on a specific root-cause surface (serving, model-platform, kernel / performance, AI engineering, eval, retrieval, vendor-risk, safety / policy). Do not drive the incident. | Paged by function from the on-call index (not by name from memory). |
| **Communications Lead (Comms Lead)** | Writes internal and external stakeholder updates. Does not drive mitigation and does not typically own the technical timeline. | `ai-sre-comms-oncall`; for safety-adjacent incidents, pairs with legal / PR earlier than classical. |
| **Scribe** | Keeps the timeline in human-readable form in the incident channel's pinned message / document. | IC scribes for small incidents; a dedicated scribe is paged for sustained incidents. |
| **Executive liaison** | Decides what can be said externally and when; decides whether a formal-notification threshold has been crossed; decides government / regulator-facing escalation. | `exec-oncall` paged at SEV-1 by default and at SEV-2 for safety-adjacent incidents. |
| **Legal liaison** | Formal-communication authority; discoverability advice; retention and privilege calls. | `legal-oncall` paged at trigger (not at severity threshold) — see §2.5. |
| **PR liaison** | Owns external statements and press communication. | `pr-oncall` paged alongside legal for user-visible safety or reputational events. |

**Pinning convention.** At incident declare, the IC updates the channel topic to a single-line
role chart:

```
IC @ic-handle | OL @ol-handle | Comms @comms-handle | Scribe @scribe-handle | SMEs: serving @srv | eval @eval | app @app | vendor @vrisk
```

Changes to any role update the topic within one minute.

### 2.2 Incident channel operating standard

Hard rules the IC enforces from declare onward.

- **One channel per incident.** Name convention `#inc-<YYYY-MM-DD>-<rb-slug>-<route-or-surface>`.
  Example: `#inc-2026-11-04-ttft-regression-summariser`. Never re-used across incidents.
- **Named roles pinned in topic.** Format above.
- **Timeline pinned.** A linked document (Google Doc / Notion page / channel pinned message)
  holds the scribed timeline; pin the link in the channel topic.
- **Threads for workstreams.** Each parallel SME workstream (serving diagnosis, eval re-run,
  vendor ticket, status-page copy draft) lives in a thread. Main-channel traffic is reserved
  for IC decisions, timeline updates, state posts, and all-hands context.
- **No DMs for incident content.** "Let me DM you about this" is banned by the IC. The record
  is the channel.
- **Social chatter moves elsewhere.** The channel is operational; move memes and apologies to
  a general channel.
- **No customer data in-channel.** Flagged outputs, user transcripts, PII — all go to the
  controlled-access artifact store. The channel references the artifact by ID (chapter 02
  §5.3 is explicit).

### 2.3 Posting-cadence template

The IC enforces a cadence: **every ~15 minutes in the active-mitigation phase**, **every ~30
minutes once stable**, and **on every severity change**. "No change" is a valid update.

```markdown
**State update — <HH:MM UTC>**

- **Severity:** SEV-2 (unchanged / escalated from SEV-3 at 14:03Z / de-escalated from SEV-1)
- **Symptom:** TTFT p95 on the `summariser` route is 3.4× its 7-day baseline; impact bounded
  to EU traffic
- **Scope:** `summariser` route on `gpt-4o-mini`, EU region. ~12% of EU users seeing >4s TTFT.
- **Mitigations in flight:**
  - Opened `chatcat-provider-primary` breaker at 14:04Z (owner @srv). Fallback engagement
    ratio reached 95% at 14:05Z; effect visible on dashboard.
  - Prefill-batch cap lowered from 128 → 64 at 14:07Z (owner @srv). Awaiting 5-min confirm.
- **Next decision point:** 14:20Z — if TTFT has returned to within 1.3× baseline, hold; if
  not, roll back the 13:45Z serving deploy.
- **Status page:** last updated 14:06Z ("investigating"). Next update at the next decision
  point.
```

**Why the template.** The five slots (severity, symptom, scope, mitigations, next decision) are
the five questions stakeholders ask when they page the IC out-of-band. Posting them pre-empts
the pages. Chapter 02 §2.2 is explicit: silence is the single most common failure mode of long
incidents.

### 2.4 Handoff note shape (across rotations and timezones)

When the outgoing IC's shift ends while the incident is open, the handoff is a pinned channel
message. The incoming IC reads it in-channel, asks clarifying questions in-channel, and
acknowledges in-channel. The record is on-channel so the next rotation and the post-mortem both
see it.

```markdown
**IC handoff — outgoing: @ic-handle (shift ending 15:00Z) → incoming: @incoming-ic**

- **Current state:** Stable. TTFT within 1.2× baseline since 14:35Z. Fallback engagement at
  95%; primary breaker remains open.
- **Open threads:**
  - Thread "serving diagnosis" — @srv / @perf — investigating root cause of prefill-queue
    accumulation. Expected update by 16:00Z.
  - Thread "vendor ticket" — @vrisk — ticket #V-45612 filed with the provider; awaiting
    acknowledgement.
  - Thread "post-mortem scaffold" — @ic-handle opened the retro doc; retro owner TBD at
    resolve.
- **Decisions still pending:**
  - At 17:00Z, decide whether to reclose the breaker on a canary (requires @srv sign-off).
  - Whether to engage legal: trigger did NOT fire (no safety classifier events). Will
    re-evaluate if event duration exceeds 6h (SLA credit threshold).
- **Next scheduled checkpoint:** 15:15Z state post.
- **Severity:** SEV-2; de-escalated from SEV-1 at 14:20Z.
- **Stakeholders already briefed:** engineering leadership (14:10Z), customer success
  (14:15Z), legal (for awareness only, 14:12Z).
```

**Handoff rule.** Cross-timezone handoffs follow the sun. The IC rotation is scoped so there is
always a waking IC. The AI SRE function's rotation design accounts for this (`mod-101`
chapter 02's rotation shape).

### 2.5 Pre-committed safety-and-compliance decision tree

The decision tree is authored out-of-incident and consulted in-incident. Its purpose is to
remove the "should I escalate?" judgement call at the moment of highest cognitive load.

```yaml
# docs/incident-command/escalation-triggers.yaml
version: 2026-11-01
note: |
  Any trigger below fires an immediate page regardless of the engineering diagnosis.
  Legal / safety / exec / PR make the determination of what follows; the IC's duty is
  engagement, not the determination.

triggers:
  - id: toxic-output-sustained
    trigger: >
      safety classifier (toxicity | self-harm | hate | sexual | violence) output rate
      exceeds 3× 7-day baseline on a user-facing route for >= 5m
    page: [safety-oncall, legal-oncall, exec-oncall]
    related_runbook: rb-toxic-output-blast
    chapter_06_trigger: 3.1

  - id: toxic-output-egregious-single
    trigger: >
      a single user-reported safety flag with an egregious sample on any production route
    page: [safety-oncall, legal-oncall, pr-oncall]
    related_runbook: rb-toxic-output-blast
    chapter_06_trigger: 3.1

  - id: weight-corruption-serving
    trigger: >
      weights load without checksum verification AND quality metric collapses
      (groundedness score moves >= 3 sigma worse within 15m of model swap)
    page: [security-oncall, legal-oncall]
    related_runbook: rb-weight-corruption
    chapter_06_trigger: 3.2

  - id: eval-set-poisoning-suspected
    trigger: >
      eval-set hash mismatch OR evidence that the regression eval set appeared in
      training data
    page: [security-oncall, legal-oncall]
    related_runbook: rb-eval-set-poisoning
    chapter_06_trigger: 3.2

  - id: regulated-domain-incorrect-answer
    trigger: >
      a medical / legal / financial route emits an incorrect or unsafe answer
      (regulated per customer contract or jurisdiction)
    page: [safety-oncall, legal-oncall, ai-risk-engineer-oncall]
    related_runbook: rb-hallucination-rate-spike
    chapter_06_trigger: 3.1

  - id: provider-outage-sla-credit
    trigger: >
      a provider outage on a contracted route exceeds the SLA credit threshold
      (duration + error-rate product per the dependency map)
    page: [ai-risk-engineer-oncall, legal-oncall]
    related_runbook: rb-provider-degraded-fallback
    chapter_06_trigger: 2.1  # EU AI Act Art. 72 post-market monitoring touchpoint

  - id: data-residency-violation
    trigger: >
      an EU-origin request is observed to have traversed a non-EU route against contract
    page: [legal-oncall, privacy-oncall]
    chapter_06_trigger: 3.2

  - id: zdr-no-training-violation
    trigger: >
      customer data potentially reached a provider path not covered by the ZDR / no-training
      commitment
    page: [legal-oncall, privacy-oncall, ai-risk-engineer-oncall]
    chapter_06_trigger: 3.2
```

**How to use it.** Any trigger firing is a *page*, not a *severity call*. Severity remains the
IC's decision based on symptom and scope. The decision tree exists so the IC does not have to
remember the eight conditions.

### 2.6 Communication-trap reference card

The IC consults this card before every status-page or external post. From chapter 02 §5.3.

| Trap | Instead do this |
|------|-----------------|
| Naming the vendor before they acknowledge | Say "an upstream provider is degraded." Name the vendor only after their public acknowledgement. |
| Over-promising a quality fix ("the model will stop hallucinating about X") | Commit to the mitigation step ("we have reduced feature capability until the regression is diagnosed"). |
| Under-communicating during a long tail | Maintain the 30-min cadence with "still stable, no change" posts explicitly. |
| Discussing user outputs / traces in the open channel | Reference artifacts by ID. Content goes to the controlled-access store. |

## 3. Validation steps

1. **Peer dry-run.** Hand the role chart and cadence template to a non-author engineer and ask
   them to simulate an IC role at a fake declare moment. They should be able to pin the topic,
   open the timeline, and write the first state post within 5 min.
2. **Decision-tree coverage.** For each trigger in §2.5, confirm that the corresponding runbook
   in the top-twenty library references the trigger (or that it is an orthogonal
   compliance-trigger with no runbook). Cross-reference with chapter 06 §3's trigger list; the
   sets should be isomorphic.
3. **Posting-cadence drill.** Run a one-hour tabletop where the IC posts state updates to an
   isolated Slack channel on the 15-min cadence from a scripted scenario. Measure the gap
   between actual and target cadence; the median gap should be ≤ 2 min.
4. **Handoff dry-run.** The outgoing IC writes the handoff note; the incoming IC acknowledges
   in-channel within five minutes with explicit clarifying questions. The post-mortem should be
   able to reconstruct both shifts' decisions from the channel alone.
5. **Channel-naming parse.** Automated check that every historical `#inc-*` channel name follows
   the `inc-<date>-<rb-slug>-<surface>` convention (regex) and is linked to a filed
   post-mortem.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Roles complete *(hard gate)* | Missing IC, OL, or Comms | Present but missing SME bench specifics | All chapter 02 §1 roles present with named rotations |
| 2 | Named by role, not person *(hard gate)* | Names appear in the chart | Mix of role and name | Role-only; names appear only in rotation references |
| 3 | Channel operating standard | Freeform conventions | Covers naming + pinning | Covers naming, pinning, threads, DM ban, customer-data ban |
| 4 | Posting-cadence template *(hard gate)* | Missing | Template present but no cadence rule | Template + cadence rule + severity-change trigger |
| 5 | Handoff note shape | Missing | Present but light on open-threads | Full shape: state, open threads, pending decisions, checkpoint, severity, briefed stakeholders |
| 6 | Pre-committed decision tree *(hard gate)* | Missing | <4 triggers | >=5 triggers; each with paged roles and chapter 06 touchpoint |
| 7 | Communication-trap card | Missing | Present, 1–2 traps | All four traps with concrete alternatives |
| 8 | Specificity to a product | Generic template | Named product, generic rotations | Named product, specific rotation handles, specific tooling |
| 9 | Composition with other modules | None | Mentions `mod-107` or chapter 06 | Cross-links to `mod-107` chapter 04, chapter 06 §3, and the runbook library |
| 10 | Readability under incident conditions | Prose-heavy | Lists and headings | Scannable, one-line rules, pinned-message friendly |

## 5. Common mistakes

- **Naming humans in the role chart.** A chart with `@alice` and `@bob` rots the moment the
  rotation changes. The chart names rotations; the rotation surfaces the person.
- **Posting cadence without a "no change" rule.** Silence is the most common failure mode of
  long incidents (chapter 02 §2.2).
- **Treating the decision tree as advisory.** It is a page, not a judgement. The IC's duty is
  engagement at trigger, not negotiation.
- **Coupling severity to the decision tree.** Severity remains the IC's call based on symptom
  and scope; the tree's output is "who gets paged," not "SEV-1 declared."
- **Using the channel for debugging prose.** Debugging goes in threads. Main-channel stays
  scannable.
- **Discussing customer content in the open channel.** Chapter 02 §5.3 is explicit: traces and
  outputs go to the controlled-access store.
- **Drafting external communication in the engineering channel.** Legal / PR own external
  communication. The engineer briefs.
- **Omitting the chapter 06 touchpoints** from the decision tree. Each trigger should name
  which §3 bucket it fires (safety / compliance / reporting).

## 6. References

- Module chapter 02, *Incident Command for AI Systems* — the role definitions, channel
  discipline, posting rhythm, bridge-call hygiene, lifecycle, handoff shape, stakeholder
  cadence, and safety-adjacent decision tree this exercise implements.
- Module chapter 06, *Legal / Compliance / Regulator-Facing Handoff* — the trigger vocabulary
  the decision tree maps to.
- Module chapter 03, *Blameless Post-Mortems* — the retro's dependence on a scribed, pinned
  channel timeline shapes the channel operating standard.
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the risk-management vocabulary referenced when the decision tree's compliance / reporting
  branches engage.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — relevant when the decision tree's
  safety-adjacent triggers (jailbreak, weight tampering, eval-set poisoning) map to
  adversarial-technique categories security recognises.
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — the industry-standard class vocabulary for the ML-specific triggers (ML02 data poisoning,
  ML06 supply chain) the compliance branches reference.
