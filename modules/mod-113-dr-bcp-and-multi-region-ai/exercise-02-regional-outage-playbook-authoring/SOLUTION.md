# Exercise 02 — Regional Outage Playbook Authoring — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-02-regional-outage-playbook-authoring.md`.

> **Scope note.** The paired exercise prompt asks the learner to author the regional-outage playbook for an AI workload — the operator-facing document the on-call IC reaches for when a cloud region is confirmed down and the ISCP from exercise 01 has been activated. The reference solution works the playbook against the same fabricated `chatcat` RAG-chat surface used across this module family. A learner who chose a different illustrative product should be scored against the rubric in §4.

## 1. Solution overview

A regional-outage playbook is a *different* artifact from the ISCP (exercise 01) and from an incident runbook (`mod-108/exercise-01`). It sits between them:

- The **ISCP** names the activation authority, the sequences of recovery, and the roles.
- The **runbook** fragment handles one specific symptom (e.g., `rb-regional-failover-dns`) — one component, one action.
- The **playbook** is the operator script that ties runbooks together for a specific *scenario* and keeps the IC on rails during a high-stress event. It is prescriptive where the ISCP is descriptive and multi-step where a runbook is single-step.

A good regional-outage playbook for an AI workload has to do five things that a traditional DR playbook does not:

1. **Separate serving failover from quality failover.** Flipping DNS makes the route respond. The judge pipeline has to confirm groundedness is back at baseline before "recovered" is declared.
2. **Pre-flight the fallback before a real outage.** The pre-flight is a daily job, not an incident-time task. The playbook *verifies* the pre-flight; it does not run it.
3. **Handle the common degraded mode** where the region is partly up (one AZ, one service) — most "regional outages" start as single-AZ or single-service degradation.
4. **Name the authority to cut traffic.** The DNS flip and the router breaker are the two most-abused operator actions in a chaotic incident; the playbook explicitly scopes them.
5. **Handle vendor dependency**, which may be colocated with the failed region (same cloud provider) or independent (different cloud provider).

The deliverable is a single committed file `playbooks/regional-outage.md` plus two supporting artifacts:

- `playbooks/pre-flight-checks.md` — the daily pre-flight checklist that the playbook assumes has run.
- `tests/regional-outage-drill.md` — the tabletop walkthrough that exercises the playbook (hands off to `exercise-05`).

The playbook is paired with the dependency map in the ISCP (exercise 01 §6 appendix B).

### 1.1 Decision rationale — playbook vs. runbook vs. ISCP

| Document | Audience | Shape | When read | When updated |
|----------|----------|-------|-----------|--------------|
| ISCP | Plan owner + auditor | Narrative + policy | Before an event | Annually + triggered |
| Playbook | Oncall IC | Scripted, numbered, one-scenario | During an event | After each drill |
| Runbook | On-call engineer | One-symptom, one-decision | During an event | After each fire |

The reference playbook is explicitly the "during-event IC script", not an educational doc. It assumes a tense context and keeps decisions bounded.

## 2. Implementation — worked answer

### 2.1 The regional-outage playbook (`playbooks/regional-outage.md`)

```markdown
# Regional Outage Playbook — chatcat

**Playbook ID:** `pb-regional-outage-chatcat`.
**Version:** 2026-11-13. **Owner:** AI SRE Director.
**Audience:** oncall IC. **Last drilled:** <date>; next due: <date + 90d>.

> You are the IC. Follow this top-down. If a step is already done, mark it and move on.
> If a step fails, stop and page the named escalation — do not improvise ordering.

## 0. Pre-conditions (should already be true; verify fast)

- [ ] ISCP activated per the criteria in `plan/chatcat-iscp.md` §3.1.
- [ ] Incident channel open, scribe assigned (`mod-108` chapter 02).
- [ ] You can see the serving dashboards for both region-A and region-B.

## 1. Confirm the scope of the outage (≤ 10 min)

- [ ] Cloud provider status page: **region-A** declared impaired or down.
- [ ] `chatcat_route_availability{region="A"}` dashboard shows: `<value>`.
- [ ] `chatcat_route_availability{region="B"}` dashboard shows: `<value>`.
- [ ] Is region-B serving healthy? If **no**, this is not a regional outage — this is a
      bigger event. Page CTO; pause this playbook. See §6 "Both regions impaired".
- [ ] Is the hosted-model provider also impacted in region-A? Check
      `chatcat_provider_error_rate{region="A"}`. If yes, you will also run §4.

Decision: proceed to §2 only if region-B is healthy. Otherwise §6.

## 2. Serving failover to region-B (target: RTO 30 min)

- [ ] Verify pre-flight: `playbooks/pre-flight-checks.md` last-passed within 24h. If
      not, page the Data Platform lead before proceeding — the retrieval replica may be
      stale.
- [ ] Execute `rb-regional-failover-dns` to shift weighted DNS to region-B.
  - **Authority to execute:** IC can authorise.
  - **Expected effect within 5 min:** `chatcat_route_availability{region="B"}` rises,
    `chatcat_requests_total{region="A"}` falls toward zero.
  - **Rollback trigger:** if region-B error rate rises above 5% within 2 min of flip,
    revert the DNS change and go to §6.
- [ ] Execute `rb-retrieval-index-promote-replica` on region-B.
  - **Authority to execute:** IC can authorise.
  - **Pre-check:** replica lag at time of flip is logged in the incident channel. If
    replica lag > 1h (index RPO target), note this in the channel — index will be stale
    and quality recovery will take longer.
- [ ] Execute `rb-session-store-promote-replica` on region-B.
  - **Authority to execute:** IC can authorise.
- [ ] Verify serving: within 10 min of DNS flip,
  - `chatcat_route_availability{region="B"}` ≥ 0.995 for 10 consecutive minutes;
  - `chatcat_requests_total{region="B"}` at or above 90% of pre-event baseline;
  - no new 5xx spikes.

**Declare: RTO (serving) met.** Post in channel: "serving recovered in region-B at <time>".
Status page: "mitigating — service restored on standby region, monitoring quality."

## 3. Quality reconstitution (target: RTO(quality) 2h)

- [ ] Warm-start the judge pipeline in region-B via `rb-judge-pipeline-warm-start`.
  - **Authority to execute:** ML Platform lead (page them now if not already present).
  - **Expected duration:** 20-40 min including snapshot restore.
- [ ] After warm-start, trigger a one-off groundedness sample against production traffic
      for a 30-min window.
- [ ] Watch the following three signals; wait 2 cycles (15 min each) at within-tolerance
      before declaring:
  - `chatcat_groundedness_score{region="B"}` within 2 sigma of pre-event baseline.
  - `chatcat_refusal_rate{region="B"}` within 2 sigma.
  - `chatcat_structured_output_valid_total{region="B"}` ratio ≥ 0.98.
- [ ] If any signal is outside 2 sigma after 2 cycles: this is a quality regression, not
      a failover artifact. Keep serving on region-B; open sub-incident for the quality
      track; see `mod-108/exercise-01` runbook `rb-hallucination-rate-spike`.

**Declare: RTO (quality) met.** Status page: "resolved — service fully restored."

## 4. Provider path (only if region-A outage includes provider)

Some hosted-model providers co-locate their endpoints in the same cloud regions as your
compute. If the provider's region-A endpoint is also impaired, the provider's own
fallback inside region-A will not help.

- [ ] Verify provider status for region-A vs. region-B endpoints.
- [ ] If provider region-A endpoint is impaired **and** provider region-B is healthy:
      the DNS flip in §2 already routes to region-B, which will use provider region-B
      by default. Nothing further required.
- [ ] If provider region-B endpoint is **also** impaired:
  - Execute `rb-vendor-fallback-pin` to pin the router breaker to the pre-attested
    fallback provider.
  - **Authority to execute:** Vendor Liaison + IC concurrence.
  - **Pre-check:** daily vendor-canary last-passed within 24h. The script
    `scripts/vendor-canary.sh` is the pre-flight (see `playbooks/pre-flight-checks.md`).
  - **Expected effect:** `chatcat_provider_error_rate` falls back under 1% within 5 min.
  - **Rollback trigger:** if fallback provider also 5xx-ing or rate-limiting, open
    sub-incident `provider-dual-failure`; this is an unmitigatable path for the duration;
    enable the degraded "no-RAG, canned response" mode per `rb-degraded-canned-mode`.

## 5. Communications cadence during the event

- Every 30 min until RTO(serving) met: internal Slack update in incident channel with
  current signal snapshot.
- Every 60 min after RTO(serving) met until RTO(quality) met: internal update + status
  page update.
- On declaring full recovery: send the post-recovery summary to leadership list.

## 6. Both regions impaired

- [ ] Open parallel incident; this is SEV-1 by default.
- [ ] Page: CTO, VP AI Platform, enterprise BCP coordinator.
- [ ] Enable the degraded "no-RAG, canned response" mode via `rb-degraded-canned-mode`.
      This mode returns a user-visible banner and a pre-authored response that routes to
      the human-agent queue.
- [ ] Vendor Liaison attempts to stand up the tertiary site (fallback provider in a
      third cloud); this is the "last-resort continuity" per the ISCP §2.2.
- [ ] Keep the degraded mode until at least one region recovers.

## 7. After recovery — back to primary

The return to active-active primary happens outside this playbook's activation window.
Open a separate change with the full change-control process; do not fold it into the
incident.

- [ ] Within 48h of incident resolution, run `playbooks/return-to-primary.md`.

## 8. Known hazards and anti-patterns during this playbook

- **Flipping DNS before replica promotion.** If DNS flips and the region-B replicas are
  not promoted, writes to the index or session store will fail against read-only replicas.
  Promote replicas first, or trigger them in parallel — not after.
- **Impatient quality declaration.** The judge-pipeline warm-start takes real wall-clock
  time. Declaring quality recovery before 2 cycles have passed is the fastest way to miss
  a real regression.
- **Un-pinning the fallback provider too early.** If §4's fallback is engaged, do not
  un-pin while the primary provider is "mostly back". Wait for the provider's own
  all-clear plus 15 min of your own canary green.
- **Ignoring the pre-flight staleness.** If the pre-flight checklist is > 24h stale, the
  implicit assumption of this playbook (replicas current, fallback valid) is broken. The
  playbook can still be run, but every step's rollback trigger becomes more likely.

## 9. Role-and-authority quick reference

| Action | Primary owner | Authority needed | Backup owner |
|--------|---------------|------------------|--------------|
| Activate ISCP | Oncall IC | Oncall IC alone | Secondary oncall |
| Execute DNS flip | Oncall IC | Oncall IC alone | Network engineer oncall |
| Promote retrieval replica | Data Platform oncall | IC concurrence | Data Platform lead |
| Promote session replica | Platform oncall | IC concurrence | Platform lead |
| Pin fallback provider | Vendor Liaison | IC concurrence | Oncall IC |
| Enable degraded canned mode | Oncall IC | IC alone (SEV-1 only) | — |
| Declare recovery | Oncall IC | IC alone | — |
| Return to primary (post-incident) | AI SRE Director | Change control | — |

## 10. Change log

| Date | Author | Change |
|------|--------|--------|
| 2026-11-13 | AI SRE | Initial publication |
```

### 2.2 Pre-flight checklist (`playbooks/pre-flight-checks.md`)

```markdown
# Pre-flight checks for pb-regional-outage-chatcat

**Runs:** daily at 02:00 UTC (automated) + ad-hoc. **Owner:** AI SRE SRE-2 rotation.

The automated job emits a pass / fail per section; a fail pages the on-call SRE, not the IC.

## Replicas
- Retrieval index: replica lag region-B ≤ 1h (RPO target).
- Session store: cross-region replication healthy, lag ≤ 5min.
- Prompts: git mirror in region-B fetched today.

## Standby components
- Judge pipeline in region-B stood up to "warm" (not full capacity) and the last snapshot
  loaded successfully.
- Weight attestation re-verified against the current pins.

## Vendor fallback
- `scripts/vendor-canary.sh` has run against the pre-attested fallback provider within
  24h; last 50 samples green.

## Playbook currency
- `pb-regional-outage-chatcat` last-drilled within the drill cadence (`mod-113/exercise-05`).

## Dependency surprise check
- `enterprise-continuity/bcp-v*` has no newer version than the ISCP has tracked.
- `mod-107` vendor-register has no provider in "degraded" state unaddressed.
```

### 2.3 Tabletop fixture (`tests/regional-outage-drill.md`)

A short fixture to let the playbook be rehearsed without a real outage:

```markdown
# Tabletop drill — region-A outage

**Cadence:** quarterly (per `mod-113/exercise-05`).

## Scenario script
At 14:00 UTC, the cloud provider status page shows region-A as "impaired, investigating".
`chatcat_route_availability{region="A"}` is 0.72 and dropping. The hosted-model provider
reports its region-A endpoints as "degraded, failover in progress".

## Roles present
- Oncall IC (driver)
- Scribe
- Vendor Liaison
- Data Platform oncall
- ML Platform lead
- Observer / facilitator (reads the scenario, grades)

## Script beats
- T+00: drop the scenario. IC opens the playbook.
- T+05: facilitator confirms region-B is healthy.
- T+15: IC should have executed §2 steps 1-3.
- T+25: facilitator confirms serving in region-B above threshold.
- T+30: IC should declare RTO(serving) met.
- T+40: facilitator injects the "provider region-B also impaired" twist.
- T+50: IC should authorise §4 pin to fallback provider.
- T+90: facilitator injects "quality signal inside tolerance" and IC should declare
  full recovery.

## Grading
Pre-event handed to each role; graded against the rubric in `mod-113/exercise-05`.
```

### 2.4 How this playbook differs from a non-AI playbook

A traditional "web-service regional outage" playbook terminates at §2: route is up in
the other region, declare recovered. The AI additions here are:

- §3 "Quality reconstitution" — dedicated, with its own RTO and its own declaration
  moment. A traditional playbook has no equivalent.
- §4 "Provider path" — the hosted-model provider is a *second* system with its own
  regional layout, which may or may not match yours. A traditional playbook assumes the
  database replica is enough; a hosted-model playbook requires a fallback provider.
- §8 "Known hazards" — called out the two AI-specific anti-patterns (impatient quality
  declaration; un-pinning the fallback too early) that have no analogue in a web
  playbook.

## 3. Validation steps

1. **Red-team review.** A second on-call IC walks through the playbook top-down
   against the tabletop in §2.3 without having authored it. Any step that requires
   information not in the playbook is a defect.
2. **Timing floor.** The sum of the "within N min" bounds in §2 and §3 must be ≤
   RTO(serving) + RTO(quality) from the BIA. If not, either the playbook is too slow or
   the RTO is unachievable.
3. **Runbook cross-reference integrity.** Every `rb-*` named in the playbook must
   resolve to a runbook in `mod-108/exercise-01`.
4. **Authority table complete.** Every action in §2–§7 appears exactly once in §9 with
   a primary owner and backup.
5. **Drill the playbook.** Run the tabletop from §2.3. Grade the IC against the rubric.
   A playbook that cannot be driven cleanly through the tabletop is not shippable.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Playbook is scenario-scoped *(hard gate)* | Catch-all / "AI incident" | Named scenario but mixes symptoms | One scenario, clear pre-conditions |
| 2 | Dual-RTO treatment *(hard gate)* | Single RTO | Serving + quality mentioned | Separate sections, separate declaration moments |
| 3 | Rollback / abort triggers per step *(hard gate)* | None | For the DNS flip only | Rollback trigger on each action |
| 4 | Provider-path handling | Not addressed | Mentioned | Full section with concurrence authority and fallback-provider pin |
| 5 | Pre-flight dependency | None | Mentioned | Separate pre-flight checklist with freshness window |
| 6 | Degraded mode *(hard gate)* | None | Mentioned | Section 6 covers both-regions-impaired with canned-mode activation |
| 7 | Authority / owner table | Missing | Implicit | Explicit matrix, every action covered once |
| 8 | Comms cadence | Missing | One-shot | Pre/during/post with status-page cadence |
| 9 | Drill fixture | None | Narrative only | Tabletop script with timing beats + twist injection |
| 10 | Post-event return-to-primary separation | Folded in | Mentioned | Explicitly deferred to a separate playbook |

## 5. Common mistakes

- **Drift into runbook territory.** The playbook starts rewriting runbook content inline.
  The playbook's job is to *sequence* runbooks; the content of each runbook lives in
  `mod-108/exercise-01`. If a learner copied the runbook body, the two artifacts will
  drift as runbooks are updated.
- **Playbook with no activation pre-conditions.** Without the §0 pre-conditions, the
  playbook can be invoked out of context (say, during a provider-only outage) and will
  mis-sequence the response. The pre-conditions double as "if this doesn't match, this
  isn't the right playbook."
- **Missing rollback triggers.** A step that only has a forward action is a step that
  cannot fail safely. Every operator action in a crisis needs a rollback trigger with a
  signal.
- **Un-named authority.** "Flip DNS" without naming who can flip DNS is how two people
  flip DNS in opposite directions during the same incident. The authority table in §9 is
  not boilerplate.
- **Baking in the return-to-primary.** The return to active-active primary is a change
  management action, not an incident-response action. Folding it into the recovery
  playbook pressures the IC to run a non-urgent change under time pressure.
- **No vendor-path section.** A playbook that assumes the vendor is in a different
  region than your workload is wrong often enough that the assumption must be verified at
  §1, not implicit.
- **Decorative tabletop.** A tabletop that walks the "happy path" without a mid-event
  twist (provider failure, quality regression, replica lag) does not stress the
  playbook. The §2.3 drill script deliberately injects the twist at T+40.

## 6. References

- NIST SP 800-34 Rev. 1 §4 — the Activation / Notification / Recovery / Reconstitution
  phase structure this playbook implements.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the MANAGE function names "response and recovery plans" as a managed control for AI systems; the dual-RTO structure here operationalises that.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML supply chain` motivates the vendor-canary pre-flight in §2.2; `ML09: output integrity attack` motivates the structured-output-validity gate in §3.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the ML-inference-API-access and supply-chain technique classes are the adversary motion that explains why the "both-regions-impaired" §6 includes a canned-response degraded mode rather than silent passthrough.
- `mod-108/exercise-01` runbook library — `rb-regional-failover-dns`,
  `rb-retrieval-index-promote-replica`, `rb-session-store-promote-replica`,
  `rb-vendor-fallback-pin`, `rb-judge-pipeline-warm-start`, `rb-degraded-canned-mode`.
- `mod-108/exercise-02` incident-command cadence — the comms cadence in §5 inherits from
  that chapter.
- `mod-113/exercise-01` ISCP — the plan this playbook is the executable form of.
- `mod-113/exercise-05` test programme — the drill cadence in §2.3.
