# Exercise 05 — Regression-Fixture Back-Feed Discipline — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-05-regression-fixture-back-feed-discipline.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below is
> derived from chapter 05 of the module (which the exercise is explicitly paired to) and chapter
> 05 §10's worksheet. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

## 1. Solution overview

The exercise asks the learner to run chapter 05's back-feed cycle end-to-end on a worked
incident: walk the six-class decision tree per contributing factor, author the fixture
candidates, assign owners, verify, and register in the fixture ledger. The deliverable is:

- The **fixture ledger** data model (a YAML schema + an initial seed file) in a
  `fixtures/ledger/` directory.
- The **decision-tree walk** that produces the candidates from the illustrative post-mortem
  of exercise 03 (`inc-2026-11-04-ttft-summariser`).
- The **verification evidence** for each fixture (how we proved it would have caught the
  incident).
- The **SLO-stack closure view** showing that every SLO has at least one linked fixture.

The reference solution below reuses exercise 03's worked post-mortem so the two exercises
compose cleanly. A learner submission that uses a different incident is acceptable as long as
every contributing factor is walked through the decision tree.

## 2. Implementation — worked answer

### 2.1 Fixture-ledger schema — `fixtures/ledger/schema.yaml`

The schema extends chapter 05 §3 with fields for cross-incident traceability and judge-drift
protection (§7.2).

```yaml
# fixtures/ledger/schema.yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
type: object
required: [fixture_id, origin_incident, class, system, owner, status, verification]
properties:
  fixture_id:
    type: string
    description: Stable slug (fx-YYYY-MM-DD-slug-N)
    pattern: "^fx-[0-9]{4}-[0-9]{2}-[0-9]{2}-[a-z0-9-]+-[0-9]+$"
  origin_incident:
    type: string
    description: Incident ID (inc-...) that produced this fixture
  class:
    enum: [alert, canary_eval_case, eval_set_entry, load_test_shape, chaos_experiment, runbook_update]
  system:
    type: string
    description: Where the fixture lives (prometheus, chatcat-evals, chatcat-loadgen, runbooks/, chatcat-chaos)
  name:
    type: string
    description: Identifier within the system (metric name, eval slug, runbook file, experiment id)
  description:
    type: string
  threshold:
    type: string
    description: Alert threshold or canary pass/fail criterion (where applicable)
  runbook:
    type: string
    description: Runbook slug the fixture is linked to (rb-...)
  owner:
    type: string
    description: Rotation name, not person (e.g. ai-infra-inference-engineer-oncall)
  due_date:
    type: string
    format: date
  status:
    enum: [candidate, owned, implemented, verified, active, measured, retired]
  verification:
    type: object
    required: [method, date, outcome]
    properties:
      method: {type: string}
      date: {type: string, format: date}
      outcome: {type: string}
  measurement:
    type: object
    properties:
      first_live_fire: {type: string, format: date}
      subsequent_fires: {type: integer}
  judge_pin:
    type: object
    description: Required for fixtures whose correctness depends on a judge model
    properties:
      model_id: {type: string}
      model_version: {type: string}
      calibration_date: {type: string, format: date}
  eval_set_boundary:
    type: object
    description: Required for canary_eval_case and eval_set_entry fixtures
    properties:
      hash: {type: string, description: Content hash used to block training contamination}
      training_index_exclusion: {type: boolean}
```

### 2.2 Decision-tree walk — for `inc-2026-11-04-ttft-summariser`

Chapter 05 §8's tree is a per-contributing-factor walk. The post-mortem (exercise 03 §2.3)
names four contributing factors. Each walks to a fixture class.

| # | Contributing factor | Decision-tree branch | Fixture class |
|---|---------------------|----------------------|---------------|
| 1 | `chatcat_prompt_cache_hit_ratio` was on a dashboard but not alerted on. | Observability gap → alert | `alert` |
| 2 | Template PR did not have a cache-impact label; stable-prefix hash not computed in CI. | Procedural / routing gap → runbook / mechanical check. In this case the check is itself the implementation, so the fixture class is `runbook_update` paired with a CI mechanical gate. | `runbook_update` |
| 3 | `rb-ttft-regression` confirm checklist named cache-hit at step 3, not step 1. | Procedural / routing gap → runbook update. | `runbook_update` |
| 4 | Template-repo merge was single-reviewer, not reviewed by `ai-infra-inference-engineer`. | Observability-of-process gap; the mechanical fix is a CODEOWNERS entry (a CI mechanical gate). Not a standard fixture class; rolled into fixture #2's `runbook_update` with documented code-owner rule. | (consolidated into #2) |

One further fixture is produced not from a contributing factor but from the retro's
*observability-prevention* branch: a load-test shape that reproduces the symptom.

| # | Branch | Fixture class |
|---|--------|---------------|
| 5 | Capacity / performance gap (load-shape repro) → load-test shape. | `load_test_shape` |

### 2.3 Fixture ledger entries (seed file)

```yaml
# fixtures/ledger/inc-2026-11-04-ttft-summariser.yaml
fixtures:
  - fixture_id: fx-2026-11-04-ttft-summariser-1
    origin_incident: inc-2026-11-04-ttft-summariser
    class: alert
    system: prometheus
    name: chatcat_prompt_cache_hit_ratio_drop
    description: >
      Burn-rate alert on prompt-cache hit ratio: trigger when the 10m-window hit ratio
      drops >= 20 percentage points on any route versus the 7-day baseline.
    threshold: "delta(chatcat_prompt_cache_hit_ratio[10m]) <= -0.20"
    runbook: rb-cache-hit-cliff
    owner: ai-infra-inference-engineer-oncall
    due_date: 2026-11-18
    status: verified
    verification:
      method: >
        Replayed 2026-11-04 13:30–14:30 UTC trace through the new alert rule using
        Prometheus `promtool test rules` on the pinned recording of the window.
      date: 2026-11-10
      outcome: >
        Alert would have fired at 13:46 UTC — 8 minutes before the 1h SLO burn-rate alert.
    measurement:
      first_live_fire: null
      subsequent_fires: 0

  - fixture_id: fx-2026-11-04-ttft-summariser-2
    origin_incident: inc-2026-11-04-ttft-summariser
    class: runbook_update
    system: runbooks/
    name: rb-cache-hit-cliff + chatcat-prompts/ci
    description: >
      (a) CI check in `chatcat-prompts` that computes the stable-prefix hash on PR diff
      and blocks PRs that move the hash without a `cache-warmup-plan` label and an
      `ai-infra-inference-engineer` approval. (b) CODEOWNERS entry adding the
      inference-engineer function as a required reviewer on prompt-template files.
    runbook: rb-cache-hit-cliff
    owner: ai-engineering-oncall
    due_date: 2026-11-25
    status: implemented
    verification:
      method: >
        PR authored against the pre-incident template with the actual 2026-11-04 diff;
        confirmed CI blocks with the expected message; confirmed CODEOWNERS gates review
        to inference-engineer.
      date: 2026-11-21
      outcome: >
        CI blocked PR; without the `cache-warmup-plan` label, promotion is prevented;
        with the label, the inference-engineer review becomes required.
    measurement:
      first_live_fire: 2026-12-02  # a real template PR was blocked and re-labelled
      subsequent_fires: 3

  - fixture_id: fx-2026-11-04-ttft-summariser-3
    origin_incident: inc-2026-11-04-ttft-summariser
    class: runbook_update
    system: runbooks/
    name: rb-ttft-regression
    description: >
      Promote cache-hit-ratio check from confirm step 3 to step 1 in `rb-ttft-regression`.
      Add an "if this signal is anomalous, swap to `rb-cache-hit-cliff` immediately" cross-
      reference at the top of the confirm checklist.
    runbook: rb-ttft-regression
    owner: ai-sre-function
    due_date: 2026-11-15
    status: verified
    verification:
      method: >
        Peer dry-run with two on-call engineers executing the updated checklist against a
        scripted alert payload pinned to the 2026-11-04 trace.
      date: 2026-11-14
      outcome: >
        Both engineers switched to `rb-cache-hit-cliff` within 3 min of declare; prior to
        the change the historical median was 7 min.
    measurement:
      first_live_fire: null
      subsequent_fires: 0

  - fixture_id: fx-2026-11-04-ttft-summariser-4
    origin_incident: inc-2026-11-04-ttft-summariser
    class: load_test_shape
    system: chatcat-loadgen
    name: prefill-burst-on-cache-invalidation
    description: >
      Load-shape: 2.4× TTFT impact of a cache-invalidation deploy. Deploy a template that
      moves the stable-prefix hash at t=0, hold traffic shape constant, observe prefill
      queue depth, TTFT p95, and cache hit ratio.
    runbook: rb-cache-hit-cliff
    owner: ai-infra-performance-oncall
    due_date: 2026-12-02
    status: implemented
    verification:
      method: >
        Ran against the pre-revert configuration in staging.
      date: 2026-11-28
      outcome: >
        p95 TTFT reaches 4.5s (vs. 1.6s baseline), prefill queue depth tops 240, matching
        the production incident's shape within 10%.
    measurement:
      first_live_fire: null
      subsequent_fires: 0
```

### 2.4 SLO-stack closure view

Chapter 05 §4 defines the closed loop: every SLO has at least one linked fixture. The
dashboard panel below is a worked example of the leadership view.

```markdown
| SLO | Primary runbook | Linked fixtures | Last-exercise date | Status |
|-----|-----------------|-----------------|---------------------|--------|
| chat-latency (28d, 99%) | rb-ttft-regression + rb-cache-hit-cliff | fx-2026-11-04-ttft-summariser-{1,3,4} | 2026-11-28 (load_test_shape) | 3 linked fixtures, active |
| chat-availability (28d, 99.9%) | rb-provider-degraded-fallback | fx-2026-08-15-vendor-provider-{1,2} | 2026-10-22 (chaos) | 2 linked fixtures, active |
| chat-quality (28d, groundedness ≥ 0.72) | rb-hallucination-rate-spike | fx-2026-09-02-hallucination-spike-{1,2,3} | 2026-10-14 (canary_eval) | 3 linked fixtures, active |
| chat-cost (28d, per-session ≤ $0.08) | rb-cost-blow-up | (none) | n/a | SLO without memory — flagged |
```

A SLO without a linked fixture is "a SLO without memory" (chapter 05 §4) and surfaces on the
leadership panel for prioritisation.

### 2.5 AI-specific trap mitigations the ledger enforces

Chapter 05 §7 names three traps that the schema and process explicitly prevent.

- **Eval-set contamination.** Every `canary_eval_case` and `eval_set_entry` fixture in the
  ledger carries an `eval_set_boundary.hash` and `training_index_exclusion: true`. Training
  pipelines refuse to ingest items whose hash matches the exclusion set.
- **Judge-model drift invalidating a fixture.** Fixtures that read a judge carry the
  `judge_pin.model_id` and `judge_pin.model_version`. The ledger's weekly audit alarms when
  the production judge model has moved from the pinned version without a recalibration
  entry.
- **Prompt-cache invalidation via the fixture.** Fixtures that touch the production path are
  tagged `touches_prod: true` and must specify a separate cache namespace or a traffic-
  shadow route. The field is a hard requirement validated by the schema check.

### 2.6 Back-feed hygiene operations

- **Alert fatigue guardrails.** New alerts prefer burn-rate; threshold alerts require an
  explicit justification field (`justification_threshold_vs_burnrate`). Quarterly review of
  any alert that has not fired in 6 months; the decision is recorded (keep / tune / retire).
- **Fixture obsolescence.** Every fixture carries a `last_reviewed` date; the ledger's
  quarterly audit flags fixtures whose referenced metric / dashboard / runbook has moved
  (name mismatch against the active catalogue).
- **Cross-incident de-duplication.** When two incidents in a quarter produce overlapping
  candidates, the retro reviewer merges before close; the ledger tolerates one fixture with
  multiple `origin_incident` entries (an array, as of schema v2026-11-01).

## 3. Validation steps

1. **Ledger schema compliance.** Every YAML file in `fixtures/ledger/` validates against
   `schema.yaml`. Any missing required field fails the audit.

   ```bash
   # Validate every ledger file against the schema
   for f in fixtures/ledger/inc-*.yaml; do
     check-jsonschema --schemafile fixtures/ledger/schema.yaml "$f" \
       || { echo "FAIL: $f does not validate"; exit 1; }
   done
   ```

2. **Decision-tree traceability.** Every entry in a post-mortem's `fixture_back-feed` block
   has a matching ledger entry whose `origin_incident` is the post-mortem's incident ID.
   Entries without an incident-origin fail the audit.

3. **Verification evidence present.** Every fixture with status `verified` or beyond has
   a non-empty `verification.outcome` field with a specific signal / replay / peer-dry-run
   description. "Verified by inspection" fails the audit.

4. **Judge pin audit (triggered).** Every canary / eval fixture that reads a judge has a
   `judge_pin` block. When the production judge moves, the audit flags any fixture whose
   `judge_pin.model_version` lags.

5. **Eval-set boundary audit (triggered).** Every canary and eval-set fixture has a
   `training_index_exclusion: true` and an `eval_set_boundary.hash`; the training pipeline's
   exclusion list contains the hash.

6. **SLO closure dashboard.** For every production SLO, at least one linked fixture is
   `active` or `measured`. SLOs with no linked fixtures appear in the leadership panel.

7. **Fire rate review.** Quarterly: every `active` fixture is reviewed for fire rate; a
   never-fired fixture is either retained with a documented reason or retired explicitly.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at ≥ 15 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Decision-tree walk per contributing factor *(hard gate)* | Missing | Walked for some factors | Walked for every contributing factor; branch named |
| 2 | Fixture class vocabulary *(hard gate)* | Outside the six classes | Classes but ambiguous assignment | Every candidate assigned to one of six classes correctly |
| 3 | Ledger schema compliance *(hard gate)* | Does not validate | Validates but missing non-required fields | Validates + non-required fields populated |
| 4 | Verification evidence *(hard gate)* | Missing | "Verified by inspection" | Specific replay / peer-dry-run with outcome |
| 5 | Owners by role | Named humans | Role for some, name for others | Role-only for every fixture |
| 6 | SLO-stack closure | No link to SLOs | Partial link | Every SLO has ≥ 1 active fixture; gaps flagged |
| 7 | Judge-pin / eval-boundary discipline | Missing | Present for some | Every eval / canary fixture has pin + exclusion |
| 8 | Status transitions | All `candidate` | Partial progression | Each fixture has a plausible position in the state machine |
| 9 | Alert fatigue guardrails | None | Partial | Burn-rate preference documented; retire-decision recorded |
| 10 | Non-fixture candidates called out | "Educate the team" accepted | Partially caught | Chapter 05 §1.7 non-fixtures all flagged and reworked |

## 5. Common mistakes

- **"Document the lesson learned" as a fixture.** Not a fixture (chapter 05 §1.7). If the
  documentation is not consumed by code, it is cosmetic.
- **"Educate the team" as a fixture.** Not a fixture. Chapter 05 §1.7 is explicit.
- **Opening a vendor support ticket as a fixture.** A ticket is not a durable change; it is a
  plan with an external dependency.
- **"Tighten the review process" without a CI implementation.** Unless the review tightening
  is enforced by a bot, a template rule, or a CODEOWNERS entry, it is a wish.
- **Skipping verification.** A fixture at status `verified` requires a specific exercise and
  a specific outcome in the ledger.
- **Missing judge pins on eval-reading fixtures.** A judge drift (chapter 05 §7.2) silently
  invalidates the fixture; the audit catches absent pins, but the submission should not
  depend on the audit.
- **Not catching eval-set contamination.** Chapter 05 §7.1 requires a training-exclusion hash;
  a canary case that leaks into training is worse than no canary.
- **Prompt-cache invalidation from the fixture itself.** Chapter 05 §7.3 — fixtures that
  touch production must specify a separate cache namespace or a shadow route.
- **One retro, one fixture.** Multiple contributing factors mean multiple fixtures. One
  factor that touches two surfaces produces two fixtures (chapter 05 §8).
- **Never auditing fire rate.** A fixture that never fires is either mis-calibrated or
  built for a class that does not recur; both outcomes require a quarterly decision.

## 6. References

- Module chapter 05, *Regression-Fixture Back-Feed Discipline* — the six fixture classes,
  the state-machine cycle, the schema, the SLO-stack closure, the AI-specific traps, and
  the decision tree this exercise implements.
- Module chapter 03, *Blameless Post-Mortems for AI Systems* — the fixture-back-feed block
  in the template; the controlled vocabulary for root cause categories cross-referenced in
  the ledger entries.
- Module chapter 01, *Top-Twenty AI Runbook Library* — the `rb-*` slugs ledger entries link
  to.
- Module chapter 06, *Legal / Compliance / Regulator-Facing Handoff* — the discipline that
  governs fixtures built in response to triggered events.
- `mod-109` chaos engineering — consumes `chaos_experiment` fixtures from this ledger.
- `mod-103` SLO engineering — the SLO stack the closure view reads.
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — risk-management vocabulary informing the ledger's cross-indexing when fixtures are
  built in response to risk-register items.
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — vocabulary for eval-set-poisoning and training-data contamination (ML02) that the
  §7.1 eval-set-boundary discipline mitigates.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — technique vocabulary for
  safety-adjacent fixtures (e.g., fixtures built against jailbreak surfaces or adversarial
  prompt classes).
