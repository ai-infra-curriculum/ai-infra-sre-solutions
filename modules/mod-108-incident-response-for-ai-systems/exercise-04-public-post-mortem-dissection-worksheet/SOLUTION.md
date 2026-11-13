# Exercise 04 — Public Post-Mortem Dissection Worksheet — Reference Solution

Paired learning file: `lessons/mod-108-incident-response-for-ai-systems/exercises/exercise-04-public-post-mortem-dissection-worksheet.md`.

> **Scope note.** The paired exercise prompt is still scaffolded; the scope inferred below is
> derived from chapter 04 of the module (which the exercise is explicitly paired to) and chapter
> 04 §7's worksheet. If the authored exercise narrows or widens scope on a subsequent content
> cycle, this solution should be regenerated.

## 1. Solution overview

The exercise asks the learner to run chapter 04's dissection method end-to-end on at least
one public post-mortem and to produce a quarterly synthesis (§5 of chapter 04) across a small
corpus. The deliverable is:

- The **dissection worksheet** (template bound; a Markdown record) for one or more public
  incidents.
- A **quarterly synthesis** that rolls up the dissected records into four answers: recurring
  failure classes, dominant detection signals, mitigations that held, repeated gaps.

The reference solution below includes the template, one fully worked dissection of the
December 2024 ChatGPT outage (whose status-page record is the public source chapter 04 names
in §2.1), and the shape of the quarterly synthesis. A learner submission for this exercise
typically produces three to five dissections; one is enough to illustrate the method.

Every fact in the worked dissection is cited to the publisher's public document. Where a
field is unknown because the publisher did not publish it, the record uses
`unknown — not in public document` explicitly; chapter 04 §3 requires this.

## 2. Implementation — worked answer

### 2.1 Dissection template — `templates/public-dissection.md`

The template mirrors chapter 04 §3 with light tightening for machine-readability (front-matter
for catalogue indexing).

```markdown
---
dissection_id: <slug-vendor-date-incident>
classification:
  primary_cause_category: <model-weight | kernel | scheduler | prompt-config | retrieval | provider | agent-cascade | eval-judge-drift | unknown-multi-class>
  matching_internal_runbook: <rb-...>
  mod_105_class: <class>
source:
  url: <permanent link>
  publisher: <vendor | cloud provider | news outlet | catalogue>
  publication_date: <YYYY-MM-DD>
  our_reading_date: <YYYY-MM-DD>
---

# Dissection: <vendor> — <incident title>

## Summary (internal paraphrase, 2–3 sentences)

<what happened, scope, mitigation, outcome, in our vocabulary>

## Timeline (from the source; UTC)

<flat chronological list; mark every inference explicitly>

## Impact

- User-visible: <paraphrase from source>
- Duration: <from vendor; ours if dependent>
- Scope: <routes, geographies, product surfaces>

## What the vendor said went wrong

<quoted or paraphrased technical narrative>

## What the vendor said they fixed / are fixing

<quoted or paraphrased remediation>

## Our translation

- **What signal would have caught this on our stack?** <...>
- **Do we have the signal?** <yes / no / partial; where it lives>
- **What mitigation would we have run?** <specific `rb-*` entry or ad-hoc path>
- **Is our matching runbook sufficient?** <yes / no — if no, what addition does this argue for>

## Candidate fixture back-feeds for us

<list, each as a fixture entry per chapter 03 §2.9>

## Gaps in the public document

<fields we would want but the publisher did not include>

## Internal classification for cross-referencing

- **AI SRE function slug:** <internal slug for the corpus index>
- **Tags:** <controlled tag list — e.g., "substrate-cascade", "telemetry-induced", "progressive-rollout">
```

### 2.2 Worked dissection — ChatGPT, 11 December 2024

The published source is OpenAI's incident record on the public status page. Chapter 04 §2.1
explicitly cites it. The dissection below treats the status-page narrative as the primary
source; all quoted language is attributed.

```markdown
---
dissection_id: openai-2024-12-11-chatgpt-outage
classification:
  primary_cause_category: scheduler
  matching_internal_runbook: rb-preemption-spike  # closest analogue; the real class is upstream substrate
  mod_105_class: substrate-control-plane
source:
  url: https://status.openai.com/incidents/ctrsv3lwd797
  publisher: OpenAI (status page)
  publication_date: 2024-12-12
  our_reading_date: 2026-11-13
---

# Dissection: OpenAI — 11 December 2024 ChatGPT, API, and Sora outage

## Summary

On 11 December 2024, a newly deployed telemetry service inside OpenAI's Kubernetes clusters
generated sufficient load on the Kubernetes API control plane to interrupt DNS-based service
discovery across every cluster, taking ChatGPT, the API, and Sora offline for the duration.
OpenAI's published narrative identifies the proximate symptom (Kubernetes-control-plane and
DNS-resolution failure across clusters) and the upstream cause (the telemetry service's
Kubernetes API load pattern). Recovery required regaining control-plane access before the
telemetry service could be removed. The incident record was published on the public status
page.

## Timeline (from the source; UTC)

The published record provides specific timestamps for the user-visible impact window; the
exact minute-by-minute engineering response timeline is not in the public document.

- `unknown — not in public document` : telemetry service deployment begins.
- 15:16 PT (23:16 UTC) : user-visible impact begins per status-page acknowledgement.
- `unknown — not in public document` : engineering detects the overloaded control plane.
- `unknown — not in public document` : responders begin regaining control-plane access.
- 19:38 PT (03:38 UTC the next day) : status-page record reports mitigation and service restoration.

**Inferred** (marked): the time between detection and control-plane recovery was constrained
by the same symptom responders were trying to fix (control-plane unavailability), which is a
recurring shape in substrate-cascade incidents.

## Impact

- **User-visible:** Across-the-board unavailability of ChatGPT, API, and Sora during the
  window. Public record.
- **Duration:** Approximately 4h 22m end-to-end user impact per the status-page incident record.
- **Scope:** Global; all clusters.

## What the vendor said went wrong

Per OpenAI's status-page narrative (paraphrased for internal use): a newly deployed telemetry
service generated overwhelming load on the Kubernetes API control plane across every cluster,
breaking DNS-based service discovery between services. The user-visible symptom was total
service unavailability; the upstream cause was the telemetry service's API load pattern.

## What the vendor said they fixed / are fixing

Per the published record: the telemetry service was removed, and OpenAI identifies follow-on
actions around safer rollout and telemetry load management. The public document does not
publish a detailed engineering write-up with specific rollout gates.

## Our translation

- **What signal would have caught this on our stack?** Our Kubernetes API server latency and
  error-rate signals (`apiserver_request_duration_seconds` p99 and
  `apiserver_request_total{code=~"5.."}` rate), combined with DNS-resolution error rate on
  `kube-dns` (or CoreDNS), would surface a substrate-cascade of this shape within minutes.
- **Do we have the signal?** Yes (partial). Our substrate dashboard includes these panels,
  but no alert is linked to them on a per-cluster basis; this is a gap chapter 04 §4.1 names
  (observability catches the wrong layer first).
- **What mitigation would we have run?** No runbook in the top-twenty library directly
  mitigates a telemetry-induced control-plane cascade; the closest is `rb-preemption-spike`
  for scheduler events. The real mitigation is a staged rollout gate on telemetry components
  that are themselves load-bearing.
- **Is our matching runbook sufficient?** No. This argues for a new runbook class —
  `rb-substrate-cascade-control-plane` — and an accompanying progressive-rollout gate on
  telemetry components.

## Candidate fixture back-feeds for us

```yaml
fixtures_added:
  - kind: alert
    system: prometheus
    name: kube_apiserver_request_latency_p99_high
    description: Per-cluster p99 latency on API server; alert on >= 2× 24h baseline for 5m
    owner: ai-infra-platform-oncall
    verification: Replay a 5-min staging burst of apiserver load; alert fires.
  - kind: runbook_update
    system: runbooks/
    name: rb-substrate-cascade-control-plane
    description: New runbook for control-plane overload resulting in cluster-wide DNS failure
    owner: ai-sre-function
    verification: Peer dry-run with two on-call personas against a fabricated alert bundle.
  - kind: chaos_experiment
    system: chatcat-chaos
    name: telemetry-loadshaped-apiserver-stress
    description: On staging, run the telemetry service at 10× its nominal API-call rate; verify
      the staged-rollout gate catches it before promotion
    owner: ai-infra-platform-oncall
    verification: The gate opens; service does not promote; alert fires on the panel above.
```

## Gaps in the public document

- Specific detection signal and time-to-detect for the engineering team.
- Specific mitigation ordering and time-to-effect per mitigation.
- Engineering retrospective beyond the status-page record (no detailed engineering blog post
  was published for this event at the time of our reading).
- Attribution to specific rollout gates added; the public record says "follow-on actions" at
  a high level.

## Internal classification for cross-referencing

- **AI SRE function slug:** `ext-openai-2024-12-11`
- **Tags:** `substrate-cascade`, `telemetry-induced`, `control-plane`, `progressive-rollout`
```

### 2.3 Quarterly synthesis template — `templates/dissection-synthesis-Q<N>-<YYYY>.md`

Chapter 04 §5 defines the four questions. The synthesis rolls up all dissections filed in
the quarter and answers each with evidence.

```markdown
---
synthesis_id: dissection-synthesis-Q1-2027
corpus:
  - ext-openai-2024-12-11
  - ext-anthropic-2025-09-postmortem-of-three-recent-issues
  - ext-cloudflare-2025-11-18
  - ext-aws-us-east-1-2025-10
window: 2026-10-01 — 2026-12-31
author: ai-sre-reviewer
---

# Quarterly dissection synthesis — Q1 2027

## 1. Which failure classes recurred across the corpus?

<list of classes with counts; for each, cite the dissections>

- **Substrate-cascade / control-plane** — 2 of 4 (ext-openai-2024-12-11, ext-aws-us-east-1-2025-10).
  Argues for prioritising the new `rb-substrate-cascade-control-plane` runbook and the
  corresponding alert set.
- **Progressive-rollout evasion (model-weight)** — 1 of 4 (ext-anthropic-2025-09). Argues for
  a canary eval cadence tighter than the vendor's rollout cadence on the primary surface.
- **Edge / CDN substrate correlation** — 1 of 4 (ext-cloudflare-2025-11-18). Argues for the
  shared-substrate correction to our SLA composition (`mod-107` chapter 05).

## 2. Which signals detected first on the vendor's stack?

<for each dissection, the first detection signal the vendor named>

Across the corpus, substrate-shaped signals (kube API server latency, DNS error rate) detected
first on 2 of 4 public records; eval / quality signals detected first on 0 of 4; this reflects
publication-bias (vendors tend to publish substrate incidents more comprehensively than
quality incidents, chapter 04 §4.4) more than it reflects the real detection shape.

## 3. Which mitigations held across the corpus?

<for each dissection, the mitigation(s) the vendor credited with mitigation>

Router-breaker / fallback mitigations held in 2 of 4 records; cache-pressure relief held in
1; manual recovery of control-plane access was the mitigation of last resort in 1 (the OpenAI
incident, where the substrate was itself the control plane).

## 4. Which gaps are repeated?

- **Per-replica rollout gates** — mentioned as either present (held) or absent (should have
  existed) in 3 of 4 records. Argues for `mod-111`'s release-engineering module to prioritise
  this.
- **Telemetry components as load-bearing without being gated** — specific to the OpenAI
  record but a shape worth monitoring.

## Prioritised internal fixture backlog from this synthesis

1. `rb-substrate-cascade-control-plane` runbook authored and merged.
2. Per-cluster apiserver latency alerts deployed.
3. Progressive-rollout gates on telemetry components authored.
4. Shared-substrate SLA composition update (`mod-107` chapter 05) revisited and recalibrated.

## Composition with modules

- Feeds `mod-109`'s chaos program with the new experiment: telemetry-loadshaped apiserver stress.
- Feeds `mod-111`'s release-engineering module with the progressive-rollout gate as a class.
- Feeds this module's chapter 05 fixture back-feed cycle with the four prioritised candidates.
```

## 3. Validation steps

1. **Source URL resolves.** For every dissection, the `source.url` resolves at the reading
   date and is a permanent link (status-page incident record, engineering blog permalink,
   catalogue entry) — not an ad-hoc Grafana link or a time-expiring preview.

   ```bash
   # Each dissection front-matter has a resolvable URL at our reading date
   for f in dissections/*.md; do
     url=$(yq eval '.source.url' "$f")
     curl --silent --head --fail "$url" > /dev/null \
       || { echo "FAIL: $f URL does not resolve: $url"; exit 1; }
   done
   ```

2. **Classification controlled vocabulary.** Primary cause category must be one of the nine
   slugs in exercise 03's §2.2. Matching internal runbook must be a `rb-*` slug that exists
   in `runbooks/` OR is marked `<proposed>` in a cover-note.

3. **Unknowns explicit.** Grep each dissection for `unknown — not in public document` where
   the record cannot cite a field. Every missing field should be marked, not silently absent.

4. **Inferences marked.** Grep each timeline for the word `Inferred` (or the project's
   agreed marker) where the author has deduced a timing. A timeline with no inference marks
   but with reconstructed internal response times is a tell that the author has imported
   speculation as fact.

5. **Candidate fixture back-feeds non-empty.** Every dissection should produce at least one
   candidate. If the incident is a shape the fleet does not touch, document it in the
   `Our translation` section and justify the empty candidate list.

6. **Reading-responsibility audit.** Human review: does the dissection re-attribute to
   engineers the vendor did not name? Does it speculate publicly about user data? Chapter 04
   §6 is the test.

7. **Synthesis coverage.** The quarterly synthesis file references every dissection in the
   quarter; dissections not referenced are either rolled in or the file documents why they
   were excluded.

## 4. Rubric / review checklist

Score 0 / 1 / 2 per dimension. Pass at ≥ 14 / 20 with no zero in `(hard gate)` dimensions.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Source citation *(hard gate)* | Missing URL or a transient link | Permanent URL, unverified | Permanent URL verified at the reading date |
| 2 | Classification *(hard gate)* | Missing or outside the nine slugs | Category assigned, no internal runbook | Category + matching runbook + mod-105 class |
| 3 | Unknowns explicit *(hard gate)* | Missing fields are silent | Some marked | All missing fields marked `unknown — not in public document` |
| 4 | Inferences marked | Mixed into the timeline | Partial | Every inference explicitly tagged |
| 5 | Our translation | Missing | Present but light on signals | All four sub-questions answered specifically |
| 6 | Candidate fixtures *(hard gate)* | Empty with no justification | Present but generic | Specific fixtures mapped to chapter 05 classes |
| 7 | Gaps in public document named | Missing | Partial | Specific fields the publisher did not include |
| 8 | Reading responsibility | Re-attributes or speculates | Paraphrases vendor's language | No re-attribution, no speculation, respects redactions |
| 9 | Quarterly synthesis | Missing | Partial coverage | Covers all four §5 questions + prioritised backlog |
| 10 | Composition with other modules | None | Mentions one | References `mod-107`, `mod-109`, `mod-111`, or chapter 03 / 05 |

## 5. Common mistakes

- **Treating news reporting as a post-mortem.** A New York Times article about Bard's James
  Webb Space Telescope answer (chapter 04 §4.4) is a *source*, not a post-mortem. The
  dissection record is explicit about which fields are from vendor publication vs. from
  reporting.
- **Guessing internal engineering response times.** If the vendor did not publish a specific
  detection or mitigation time, do not estimate it. Mark it `unknown — not in public document`.
- **Re-attribution.** If the vendor's record does not name an engineer, the dissection does
  not either. Chapter 04 §6 is explicit.
- **Editorialising the vendor.** "They should have known better" is not a dissection product.
  The dissection's object is internal value, not vendor assessment.
- **Blanket copy from the status page into the dissection.** The dissection re-indexes the
  incident into the function's taxonomy; it does not just quote.
- **Empty candidate fixtures with no justification.** If the dissection produces no
  candidate, the `Our translation` section must say why. "We do not do this surface" is a
  valid answer when documented; an empty block without comment is not.
- **Omitting the gaps section.** The gaps field is the function's monitoring plan for the
  vendor's next event; omitting it loses the compounding value of the corpus.
- **Skipping the quarterly synthesis.** A folder of individual dissections without a synthesis
  is a reading log, not an operational practice.

## 6. References

- Module chapter 04, *Pattern-Matching Against Public Post-Mortems* — the dissection template,
  the sources this exercise consumes, the recurring patterns, and the synthesis step.
- Module chapter 03, *Blameless Post-Mortems for AI Systems* — the controlled-vocabulary slugs
  used in the classification block; the fixture back-feed schema the candidate block mirrors.
- Module chapter 05, *Regression-Fixture Back-Feed Discipline* — the six fixture classes the
  candidate list maps to.
- OpenAI status page incident, 11 December 2024 — [status.openai.com/incidents/ctrsv3lwd797](https://status.openai.com/incidents/ctrsv3lwd797).
  The worked example's source.
- Anthropic engineering, *A Postmortem of Three Recent Issues* (September 2025) —
  [www.anthropic.com/engineering/a-postmortem-of-three-recent-issues](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues).
  Cited in chapter 04 §2.1 and in the synthesis template.
- Cloudflare engineering, *Cloudflare service outage, 12 June 2025* and *18 November 2025 outage* —
  [blog.cloudflare.com/cloudflare-service-outage-june-12-2025/](https://blog.cloudflare.com/cloudflare-service-outage-june-12-2025/)
  and [blog.cloudflare.com/18-november-2025-outage/](https://blog.cloudflare.com/18-november-2025-outage/).
  Cited in chapter 04 §2.2.
- AI Incident Database, [incidentdatabase.ai](https://incidentdatabase.ai/) — the catalogue
  chapter 04 §2.3 names for safety and quality events.
- OECD AI Incidents and Hazards Monitor (AIM), [oecd.ai/en/incidents](https://oecd.ai/en/incidents)
  — the cross-jurisdiction catalogue chapter 04 §2.3 names.
- NIST AI Risk Management Framework and the Generative AI Profile,
  [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)
  — the risk-class vocabulary cross-referenced in chapter 04 §2.3.
- MITRE ATLAS, [atlas.mitre.org](https://atlas.mitre.org/) — relevant for safety-adjacent
  dissections (jailbreaks, adversarial prompts).
- OWASP Machine Learning Security Top 10,
  [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/)
  — class vocabulary for cross-indexing dissections that align with ML top-10 categories.
