# Exercise 03 — Blameless Post-Mortem Template, AI-flavoured — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-03-blameless-post-mortem-template-ai-flavored.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below is
> derived from chapter 03 of the module (which the exercise is explicitly paired to) and chapter
> 03 §8's worksheet. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

## 1. Solution overview

The exercise produces two artifacts: **(a)** the AI-flavoured post-mortem template as a Markdown
file the function commits to its documentation repository, and **(b)** one worked
post-mortem for an illustrative incident that exercises every section of the template. The
template extends the Google / Allspaw shape with the three AI-specific additions chapter 03
codifies: AI root-cause categories, mandatory fixture back-feed, and cross-peer co-authorship
for incidents that span ownership.

The worked post-mortem below describes an illustrative `summariser` TTFT regression that
turned out to be a prompt-template cache-cliff. The incident is fabricated for pedagogy; the
template, taxonomy, and process are drawn from chapter 03.

## 2. Implementation — worked answer

### 2.1 Template — `templates/post-mortem.md`

```markdown
---
template_version: 2026-11-01
owner: ai-sre-function
---

# Incident post-mortem — `<incident-id>` — `<one-line title>`

## Header

- **Incident ID:** `<inc-YYYY-MM-DD-rb-slug-surface>`
- **Severity (final):** `<SEV-1 | SEV-2 | SEV-3>`
- **Start / detect / declare / resolve (UTC):** `<HH:MM | HH:MM | HH:MM | HH:MM>`
- **TTD / TTM / TTR:** `<mm:ss | HH:MM | HH:MM>` (detect, mitigate, resolve)
- **Authors:** `<retro-owner>` + `<co-author(s)>` (co-author required for multi-category
  incidents — see §Co-authorship below)
- **Reviewers:** peer-SME, AI SRE, legal (if triggered)

## Summary (one paragraph)

Executive summary of the event. Must stand alone: what happened, scope, primary cause class,
mitigation that resolved it, and the one fixture we are back-feeding. No more than 150 words.

## Impact

- **User-visible impact:** <who experienced what, with numbers>
- **SLO impact:** <error-budget burn by SLO, named>
- **Revenue / contractual impact:** <quantified where measurable>
- **Safety / policy impact:** <if applicable; this is what legal and policy will read>
- **Data / privacy impact:** <if applicable>

## Timeline

Flat chronological list. Each event: UTC timestamp, actor-role, event type
(signal / decision / mitigation / communication), one-line description. No interpretation.

| Time (UTC) | Actor (role) | Event type | Description |
|------------|--------------|------------|-------------|
| HH:MM      | <role>       | signal     | <one line>  |

## Root-cause analysis

### Primary cause

**Category (controlled vocabulary):** one of `model-weight | kernel | scheduler | prompt-config
| retrieval | provider | agent-cascade | eval-judge-drift | unknown-multi-class`.

**Specific instance:** one sentence naming the exact instance in this incident.

### Contributing factors

Two to five entries. Each is written in two parts: the **fact**, and the **system-shape question**
it raises.

- **Fact:** <fact>. **System-shape question:** <question>.

### Five-whys (or equivalent)

If the root cause is unclear or chained, document the five-whys / Ishikawa / fault-tree walk.

## What went well

- `<observability caught it: which signal, which runbook, how early>`
- `<fallback contract held: which route failed and which fallback absorbed>`
- `<any other preserved control to NOT disassemble in the fix>`

## What went poorly

- **Detection gaps:** <what should have detected earlier>
- **Response gaps:** <what took longer than it should have, with the system-shape question>
- **Communication gaps:** <named stakeholder not briefed on time>

## Action items

| ID | Description | Owner | Due | Priority | Fixture type | Verification |
|----|-------------|-------|-----|----------|--------------|--------------|
| AI-1 | <action> | <role> | YYYY-MM-DD | P0/P1/P2 | alert/canary/eval/load/chaos/runbook/doc | <how we will know it landed> |

## Fixture back-feed (mandatory)

```yaml
fixtures_added:
  - kind: <alert | canary_eval_case | eval_set_entry | load_test_shape | chaos_experiment | runbook_update>
    system: <where it lives>
    name: <stable identifier>
    description: <one line>
    owner: <role>
    verification: <how we proved it would have caught this>

fixtures_updated:
  - kind: <...>
    system: <...>
    change: <one line>
    owner: <role>
```

If this section is empty, the retrospective is not complete. Document why no fixture is
possible as a reviewable exception.

## Co-authorship note (required when primary + contributing factors span >=2 categories)

- **Categories spanned:** <list>
- **Co-authors by category:** <role per category>
- **Composition oversight:** <AI SRE co-author by default>

## Related incidents and prior art

- **Repeat events:** `<inc-...>` (same primary cause category), `<inc-...>` (same runbook class)
- **Related runbooks / chapters:** `rb-<slug>`, `mod-105 chapter N`
- **External references:** <public post-mortems that match the shape>

## Appendix

- Trace IDs, Honeycomb saved queries, Grafana panel links pinned to the incident window.
- SME write-ups (one section per SME contributor).
- Scribe's raw timeline (link).
- Bridge-call notes (link; scribe re-posted decisions to the channel at the time).
```

### 2.2 Controlled vocabulary — AI root-cause categories

Chapter 03 §3 defines the eight (plus an unknown bucket). Template-compliant values are the
exact slugs below.

| Slug | Covers |
|------|--------|
| `model-weight` | Provider-side silent version update; in-house fine-tune promotion; weight-corruption event; progressive-rollout regression. `mod-105` chapter 03 "silent output drift" sub-class. |
| `kernel` | GPU driver / CUDA / cuDNN / FlashAttention / NCCL / numeric-format change; Pytorch / Triton / XLA / TensorRT compilation surface. `mod-105` chapter 03 "kernel regression" sub-class. |
| `scheduler` | Kubernetes scheduler, autoscaler, preemption, PDB, priority-class, node pool. `mod-105` chapter 03 "scheduler eviction" sub-class. |
| `prompt-config` | Prompt template / system prompt / agent scaffolding / feature flag / model-id / application code. |
| `retrieval` | Index staleness, embedding drift, embedding-model change, chunking change, tokenizer change, ingest failure, replica desync. `mod-105` chapter 05. |
| `provider` | Hosted model / embedding / moderation provider outage, degradation, deprecation, breaking API change, silent behavioural change. `mod-107` chapter 03. |
| `agent-cascade` | Multi-step agent, orchestrator, tool-call pipeline, planner feedback loop. `mod-105` chapter 06. |
| `eval-judge-drift` | Judge-model, canary-eval-set, or eval-pipeline change. `mod-105` chapter 07. |
| `unknown-multi-class` | Reserved for cases that resist single classification. Reviewed closely; the retro owner documents what was tried. |

### 2.3 Worked post-mortem — illustrative

```markdown
# Incident post-mortem — inc-2026-11-04-ttft-summariser — `summariser` TTFT regression traced to prompt-template cache cliff

## Header

- **Incident ID:** inc-2026-11-04-ttft-summariser
- **Severity (final):** SEV-2
- **Start / detect / declare / resolve (UTC):** 13:45 / 13:54 / 13:57 / 14:38
- **TTD / TTM / TTR:** 09:00 / 00:33 / 00:41
- **Authors:** `ai-sre-ic-oncall` (retro owner) + `ai-engineering-oncall` (co-author, prompt-config category)
- **Reviewers:** `ai-infra-inference-engineer-oncall`, `ai-sre-reviewer`, (legal review not triggered)

## Summary

At 13:45 UTC a prompt-template change deployed to the `summariser` route reduced the
prompt-cache hit ratio from 0.82 to 0.21, which compounded the per-request TTFT by 2.4× and
burned 11% of the chat-latency SLO's 28-day budget before the 1h burn-rate alert fired at
13:54. The IC declared at 13:57, opened the router breaker to the fallback provider at 14:04,
and the on-call engineer confirmed the template commit as the correlate; the template was
reverted at 14:30, cache warmed, and the breaker closed at 14:38. The fixture back-feed is a
CI check in the `chatcat-prompts` repository that blocks template PRs moving the stable-prefix
hash without a declared cache-warmup plan.

## Impact

- **User-visible impact.** 14.2k affected sessions, p95 TTFT 4.8s vs. 1.6s baseline. ~2.1k
  sessions abandoned mid-response (telemetry-measured, not survey).
- **SLO impact.** chat-latency SLO 28-day error-budget burned 11% (dashboard panel linked).
- **Revenue / contractual impact.** No per-customer SLO breach per the contracts register. No
  measurable revenue impact (`summariser` is a non-billable surface).
- **Safety / policy impact.** None identified; no safety-classifier events in the window.
- **Data / privacy impact.** None.

## Timeline

| Time (UTC) | Actor (role) | Event type | Description |
|------------|--------------|------------|-------------|
| 13:40 | ai-engineering-oncall | mitigation | Deployed `chatcat-prompts@a1b2c3` — summariser template refactor.
| 13:45 | — | signal | `chatcat_ttft_seconds` p95 on `summariser` begins climbing (1-min granularity dashboard).
| 13:45 | — | signal | `chatcat_prompt_cache_hit_ratio` on `summariser` drops from 0.82 to 0.21.
| 13:54 | pagerduty | signal | 1h burn-rate alert fires on chat-latency SLO.
| 13:57 | ai-sre-ic-oncall | decision | Incident declared at SEV-3; `#inc-2026-11-04-ttft-summariser` channel opened.
| 13:58 | ai-sre-ic-oncall | decision | SEV-3 → SEV-2 after 5-min burn-rate alert also fires.
| 14:01 | ai-infra-inference-engineer-oncall | mitigation | Opened `rb-ttft-regression` confirm checklist; `rb-cache-hit-cliff` identified as true runbook at step 3.
| 14:04 | ai-infra-inference-engineer-oncall | mitigation | Router breaker opened to fallback provider; fallback engagement at 95% by 14:05.
| 14:06 | ai-sre-comms-oncall | communication | Status page updated to "investigating."
| 14:12 | ai-engineering-oncall | decision | Correlated `chatcat-prompts@a1b2c3` deploy; stable-prefix hash moved.
| 14:30 | ai-engineering-oncall | mitigation | Reverted template PR; cache begins warming.
| 14:35 | — | signal | `chatcat_prompt_cache_hit_ratio` on `summariser` returns to 0.78.
| 14:38 | ai-sre-ic-oncall | decision | Breaker closed; incident declared resolved.
| 14:40 | ai-sre-comms-oncall | communication | Status page updated to "resolved."

## Root-cause analysis

### Primary cause

**Category:** `prompt-config`.

**Specific instance:** The 13:40 UTC deploy of `chatcat-prompts@a1b2c3` refactored the
`summariser` system prompt in a way that moved the stable-prefix hash, invalidating the
provider-side prompt cache and multiplying per-request prefill cost by ~2.4×.

### Contributing factors

- **Fact:** The template PR did not have a cache-impact label and was reviewed as a stylistic
  refactor. **System-shape question:** our template-repo review process has no mechanical signal
  for cache invalidation; the stable-prefix hash was not computed in CI.
- **Fact:** The `chatcat_prompt_cache_hit_ratio` metric was on a dashboard but had no alert.
  **System-shape question:** our observability stack has the signal but the SLO stack does not
  consume it as an alert.
- **Fact:** The runbook on-call initially opened `rb-ttft-regression`, not `rb-cache-hit-cliff`.
  **System-shape question:** our confirm checklist in `rb-ttft-regression` names cache-hit as
  step 3, not step 1. Route to `rb-cache-hit-cliff` would be faster if cache were earlier.
- **Fact:** The template-repo PR was merged within 12 min of open with a single reviewer who
  was not from `ai-infra-inference-engineer`. **System-shape question:** our template
  cross-review expectation (inference-engineer on cache-touching PRs) is a norm, not a
  mechanical check.

### Five-whys walk

1. Why did TTFT regress? Prompt cache hit ratio collapsed.
2. Why did the cache hit ratio collapse? The stable-prefix hash changed at the template deploy.
3. Why did the deploy change the prefix hash? The refactor moved a conditional block above
   the stable prefix line without recognising the cache implication.
4. Why was this not caught in review? The reviewer was application-side and not aware that
   moving the block would invalidate the cache; no mechanical signal flagged it.
5. Why is there no mechanical signal? The template repo's CI does not compute the stable-prefix
   hash before and after the PR and surface a diff. (This is the fixture.)

## What went well

- **The 1h burn-rate alert fired on time.** The chat-latency SLO burn-rate alert caught the
  event at 13:54 (9 min after root cause).
- **The fallback contract held.** The router breaker to the fallback provider absorbed traffic
  while the template was reverted, bounding user impact to the first ~15 min of elevated TTFT.
- **The scribed timeline was clean.** The retro had a readable timeline within an hour of
  resolve, which cut the retro-authoring time roughly in half.

## What went poorly

- **Detection gap.** `chatcat_prompt_cache_hit_ratio` was visible but not alerted on; an alert
  on this signal would have caught the regression by ~13:46, 8 min before the burn-rate alert.
- **Response gap.** The on-call opened `rb-ttft-regression` first; the switch to
  `rb-cache-hit-cliff` cost ~4 min of mitigation time. The confirm-checklist order in
  `rb-ttft-regression` is a candidate for a fixture (step promotion).
- **Review-process gap.** The template-repo PR did not trigger an inference-engineer review;
  the merge was single-reviewed. This is a procedural gap flagged as an action item, but the
  **implementation-shaped** fix is a CI check (see fixture below), not "educate reviewers."

## Action items

| ID | Description | Owner | Due | Priority | Fixture type | Verification |
|----|-------------|-------|-----|----------|--------------|--------------|
| AI-1 | Add burn-rate alert on `chatcat_prompt_cache_hit_ratio` (drop > 20pp over 10m). | `ai-infra-inference-engineer-oncall` | 2026-11-18 | P0 | alert | Replay the 13:40–14:00 UTC trace through the new alert rule in staging. |
| AI-2 | CI check in `chatcat-prompts` that computes and diffs the stable-prefix hash; block PRs that move the hash without a `cache-warmup-plan` label. | `ai-engineering-oncall` | 2026-11-25 | P0 | ci-gate (fixture class: runbook / doc + mechanical check) | PR authored against the pre-incident template; confirm CI blocks. |
| AI-3 | Promote cache-hit-ratio check from step 3 to step 1 in `rb-ttft-regression`. | `ai-sre-ic-oncall` | 2026-11-15 | P1 | runbook_update | Peer dry-run with two on-call engineers; confirm time-to-correct-runbook decreases. |
| AI-4 | Add a load-test shape in `chatcat-loadgen` that reproduces the cache-invalidated prefill burst. | `ai-infra-performance-oncall` | 2026-12-02 | P1 | load_test_shape | Run against the pre-revert config; confirm TTFT p95 doubles. |
| AI-5 | Document the correlated-event checklist step in the retro template so future retros flag single-reviewer template merges. | `ai-sre-reviewer` | 2026-11-20 | P2 | doc | Reviewer checklist updated and cross-linked. |

## Fixture back-feed (mandatory)

```yaml
fixtures_added:
  - kind: alert
    system: prometheus
    name: chatcat_prompt_cache_hit_ratio_drop
    description: Burn-rate alert on prompt-cache hit ratio (>= 20pp drop over 10m on any route)
    owner: ai-infra-inference-engineer-oncall
    verification: Replay 2026-11-04 trace through the rule; alert would have fired at 13:46.
  - kind: runbook_update
    system: runbooks/rb-ttft-regression.md
    name: rb-ttft-regression
    description: Promote cache-hit check from confirm step 3 to step 1
    owner: ai-sre-function
    verification: Peer dry-run with two on-call personas against a scripted alert.
  - kind: load_test_shape
    system: chatcat-loadgen
    name: prefill-burst-on-cache-invalidation
    description: Loadshape reproducing the 2.4× TTFT impact of a cache-cliff deploy
    owner: ai-infra-performance-oncall
    verification: Reproduces p95 TTFT 4.5s on the pre-revert configuration in staging.

fixtures_updated: []
```

## Co-authorship note

- **Categories spanned:** primarily `prompt-config`; the serving-path and observability
  contributing factors do not constitute a second primary category per chapter 03 §4.2's rule
  (a chained incident across categories would).
- **Co-authors by category:** `ai-engineering-oncall` on the `prompt-config` fact; the AI SRE
  retro owner on composition oversight.

## Related incidents and prior art

- **Repeat events.** inc-2026-05-19-tokenizer-cache (tokenizer version bump invalidated cache —
  same primary category `prompt-config`, different specific mechanism).
- **Related runbooks / chapters.** `rb-ttft-regression`, `rb-cache-hit-cliff`; chapter 01 §3.3
  of this module; `mod-105` chapter 03 on silent output drift (adjacent class).
- **External references.** Chapter 04's dissection of the Anthropic September 2025
  post-mortem includes a context-window routing bug that aged similarly in the cache; our
  review cycle references that dissection.

## Appendix

- Grafana: `chatcat/serving-health` panel `Prompt cache hit ratio` pinned to the window.
- Honeycomb: saved query `chatcat / prompt-prefix-hash by template-version`.
- Scribe raw timeline: `/incidents/inc-2026-11-04-ttft-summariser/timeline.md`.
- Bridge-call notes: n/a (no bridge was used; the incident was text-only).
```

## 3. Validation steps

1. **Template compliance check.** Automated: a linter verifies every post-mortem has the
   sections `Header`, `Summary`, `Impact`, `Timeline`, `Root-cause analysis`, `What went well`,
   `What went poorly`, `Action items`, `Fixture back-feed (mandatory)`, and
   `Related incidents and prior art`. Any missing section fails the retro.

   ```bash
   REQUIRED=('## Header' '## Summary' '## Impact' '## Timeline' \
             '## Root-cause analysis' '## What went well' '## What went poorly' \
             '## Action items' '## Fixture back-feed' '## Related incidents and prior art')
   for f in postmortems/*.md; do
     for h in "${REQUIRED[@]}"; do
       grep -qF -- "$h" "$f" || { echo "FAIL: $f missing $h"; exit 1; }
     done
     grep -qE '^- \*\*Category:\*\*' "$f" \
       || { echo "FAIL: $f missing primary-cause category"; exit 1; }
   done
   ```

2. **Controlled-vocabulary check.** Automated: the primary-cause category must be one of the
   nine slugs in §2.2. Any other value fails.

3. **Fixture non-empty check.** Automated: the `fixtures_added` block must have at least one
   entry, OR the retro must include a reviewed-and-approved exception paragraph.

4. **Blameless-tone review.** Human: a non-author reviewer reads for proximate-human language
   ("Sam deployed the wrong template," "the on-call missed the alert"). Any such language
   sends the retro back for rewrite.

5. **Co-authorship check (triggered).** Human: if the primary cause's category is distinct
   from any contributing-factor's category, the author list must include a co-author from the
   contributing-factor peer; the AI SRE is a default co-author for multi-category incidents.

6. **Timing dashboard.** Automated: filing timestamps. The retro must be drafted within two
   business days of resolve and submitted within five business days (chapter 03 §4.4). The
   dashboard surfaces overdue retros to leadership.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at ≥ 16 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Template sections complete *(hard gate)* | One or more required section missing | All sections present, some thin | All sections present and populated |
| 2 | Primary cause from controlled vocabulary *(hard gate)* | Missing or outside the nine slugs | Present but imprecise instance | Named slug + specific instance sentence |
| 3 | Contributing factors with system-shape question | Prose list only | Facts present, questions missing | Each factor has fact + system-shape question |
| 4 | Fixture back-feed non-empty *(hard gate)* | Empty with no exception | Non-empty but vague | Non-empty with class, system, owner, verification |
| 5 | Action items have owners, dates, priorities | Missing one or more of those | Partial | Every item: owner, due, priority, fixture type, verification |
| 6 | Blameless tone *(hard gate)* | Proximate-human blame present | Occasional "we should have" lapses | Consistent system-shape voice |
| 7 | Co-authorship on multi-category *(hard gate)* | Multi-category with single author | Co-author present but AI SRE absent | Co-author per category, AI SRE present |
| 8 | Timeline voice | Interpretation in-timeline | Facts but inconsistent role voice | Facts only, actor-role voice, scribed |
| 9 | Observation vs. inference separated | Mixed in root-cause narrative | Partial | Clear split; inference contestable without rewriting observation |
| 10 | Timing met | > 5 business days to submission | 2–5 days | Draft within 2 days, submission within 5 |

## 5. Common mistakes

- **Picking the convenient root cause.** The proximate deploy-shaped change is often not the
  real cause for provider / embedding / judge-drift incidents (chapter 03 §7).
- **"We will be more careful" as an action item.** Not a fixture. Chapter 03 §7 is explicit:
  if the system requires humans to remember something, the system is unchanged.
- **Blaming the engineer who "did the thing."** Repeat-victim pattern. Ask why the system
  kept routing them to that surface (chapter 03 §7).
- **Over-redacting on legal request.** Legal asks for *specific* redactions, not blanket ones.
  A retro that redacts the technical narrative teaches nothing.
- **Drafting in a DM.** The function's culture is explicit: drafts are visible; reviewers help.
- **Missing the fixture back-feed section.** The most-often-missed section; the template makes
  it mandatory for a reason.
- **Single author on a multi-peer incident.** A tell that the retro undercounts the surface
  (chapter 03 §4.2). The AI SRE is a default co-author for composition oversight.
- **Classifying as `unknown-multi-class` without documentation.** The category is reserved for
  cases that resist single classification; the retro owner must document what was tried.

## 6. References

- Module chapter 03, *Blameless Post-Mortems for AI Systems* — the template structure,
  controlled vocabulary, co-authorship rule, drafting discipline, review cycle, and
  trap-patterns this exercise implements.
- Module chapter 05, *Regression-Fixture Back-Feed Discipline* — the six-class fixture
  vocabulary the mandatory section uses.
- Module chapter 01, *Top-Twenty AI Runbook Library* — the runbook slugs referenced in
  the `rb-` IDs and the related-runbooks section.
- Google SRE book, *Postmortem Culture: Learning from Failure* chapter (public at
  [sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/))
  — the parent template this one extends.
- John Allspaw, *Blameless PostMortems and a Just Culture* (Code as Craft, 2012)
  [codeascraft.com/2012/05/22/blameless-postmortems/](https://www.codeascraft.com/2012/05/22/blameless-postmortems/)
  — the foundational essay on the blameless premise.
- NIST AI Risk Management Framework, [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the risk-management vocabulary when the retro's safety / policy impact section engages.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — relevant when the primary cause
  category intersects adversarial-technique surfaces (e.g., `eval-judge-drift` on a poisoned
  set; `model-weight` on a tampering event).
- OWASP Machine Learning Security Top 10, [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary the retro may reference when the primary cause aligns (e.g., ML02
  training-data poisoning aligns with `eval-judge-drift`).
