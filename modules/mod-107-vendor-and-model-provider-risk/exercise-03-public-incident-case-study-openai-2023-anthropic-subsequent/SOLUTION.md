# Exercise 03 — Public Incident Case Study: Frontier-Lab Outages (Solution)

> **Factual-care note.** This solution walks learners through how to
> conduct a case study of publicly reported model-provider outages
> *without paraphrasing vendor postmortems from memory*. Specific
> timelines, durations, and root-cause phrasings MUST come from the
> vendor's own status page or an official postmortem; this file
> intentionally does not assert those specifics and marks slots with
> `<!-- cite-primary-source: ... -->` where the learner is expected to
> cite primary sources. Those tags are blocking for auto-merge, per
> the module's source policy.

## 1. Solution overview

The goal of the exercise is NOT to recount the history of frontier-lab
outages. It is to:

1. Teach a repeatable method for extracting *reliability lessons* from
   someone else's public incident so the learner's own team doesn't
   re-learn them through their own outage.
2. Produce an artifact a team can actually use: a one-page
   "What-would-break-here?" review that maps each lesson to a specific
   control the team either already has, is missing, or has partially.
3. Reinforce that the primary sources are the vendor's status page and
   postmortem, cross-referenced with the vendor's engineering blog if
   one exists. Journalism is useful for blast-radius context (who
   noticed, who complained) but should not be the sole source for
   technical claims.

Case studies done this way are reusable: as new frontier-lab outages
are published, the learner slots them into the same frame rather than
writing a new document from scratch each time.

## 2. Worked implementation

### 2.1 Case-study frame (template)

Every case study — whether for the 2023 OpenAI outages, subsequent
Anthropic incidents, or any future provider — uses the same frame.

```markdown
# Case Study: <Vendor> — <Short incident title>

Date range: <YYYY-MM-DD to YYYY-MM-DD>
Primary sources (links):
  - <vendor status page incident URL>
  - <vendor engineering postmortem URL, if any>
  - <corroborating independent reports, dated>

## 1. What happened (vendor-stated)
<Two to five bullet points, each with inline citation to a primary
source. If the vendor has not published a cause, say so explicitly.
Do not invent a cause.>

## 2. Surfaces affected (vendor-stated)
- API: <affected components, per status page>
- UX surfaces (ChatGPT, Claude UI, Gemini UI, etc.): <per status page>
- Regions: <per status page or "not stated">
- Downstream products that integrate the provider: <public reports>

## 3. Timeline (from primary sources, in UTC)
| Time (UTC) | Event | Source |
|---|---|---|
| <t0> | Impact begins per status page | <link> |
| <t1> | Vendor acknowledges | <link> |
| <t2> | Mitigation starts | <link> |
| <t3> | Services recovered | <link> |
| <t4> | Final update / postmortem | <link> |

## 4. Observed symptoms (what a dependent team would have seen)
- Elevated 5xx or 4xx: <specifics only if vendor stated or widely reported>
- Latency: <specifics only if vendor stated>
- Partial availability (one model family, one region, one feature): <per sources>
- Status-page signal lag relative to in-the-wild impact: <if quantifiable>

## 5. Mechanism (if vendor disclosed)
<Only what the vendor actually said. If they said "infrastructure
issue" and nothing more, write exactly that. Flag weak disclosure
rather than filling the gap with plausible speculation.>

## 6. Lessons the learner can extract (one line each)
L1. ...
L2. ...
...

## 7. Our-team mapping
| Lesson | Our current control | Gap | Owner |
|---|---|---|---|
| L1 | <what we do today> | <delta to the lesson> | <team> |
| L2 | ... | ... | ... |

## 8. Ticket backlog (opened from the gaps column)
- [ ] <ticket link> — <one-sentence what-and-why>
```

### 2.2 Illustrative worked application

Below is a worked example of the frame — deliberately short in
vendor-specific claims because this file does not have access to
status-page content at author time. The learner's submission should
flesh in every `cite-primary-source` span with a cited primary source.

```markdown
# Case Study: OpenAI — 2023 availability incidents

Date range: 2023-01-01 to 2023-12-31
Primary sources:
  - OpenAI Status — incident history, https://status.openai.com/history
  - OpenAI engineering/blog posts for the specific incidents cited
    <!-- cite-primary-source: link each incident to its exact status-page
         incident ID and postmortem, if any. Do not paraphrase a
         postmortem the student has not read. -->

## 1. What happened (vendor-stated)
- Multiple separately-reported availability events across API and
  ChatGPT over the year, each with its own status-page entry.
  <!-- cite-primary-source: enumerate the incident IDs the student selected
       for the case study; at least one must have a postmortem or an
       extended vendor statement attached. -->
- One widely reported event on 2023-11-08 was attributed by OpenAI
  on its status page to a distributed-denial-of-service pattern of
  traffic affecting the API and ChatGPT.
  <!-- cite-primary-source: confirm exact language and status-page URL
       from https://status.openai.com/history before quoting. -->

## 2. Surfaces affected (vendor-stated)
<!-- cite-primary-source: per incident, list components the vendor marked
     impacted (e.g., "API", "ChatGPT", "Playground", "Labs"). -->

## 3. Timeline (from primary sources, UTC)
<!-- cite-primary-source: fill rows from the status-page incident. Do NOT
     reconstruct timestamps from Twitter/press coverage. -->

## 4. Observed symptoms (what a dependent team would have seen)
Generalizable patterns across vendor outages that the learner should
be able to see in their own logs when reviewing historical windows:
- 5xx spikes against the vendor host, with a corresponding rise in
  TCP/TLS handshake failure rate.
- Longer time-to-first-byte before hard errors begin, as upstream
  queues saturate.
- Partial impact — e.g., chat completions degraded while embeddings
  remain healthy, or one model family affected and others not.
- Vendor status page updates arrive after our own alerts fire
  (this is the normal direction, not a flaw of either side —
  our telemetry is closer to our traffic).

## 5. Mechanism (if vendor disclosed)
<!-- cite-primary-source: paste only what the vendor wrote. If the vendor
     said "we are investigating an issue" and nothing further, write
     that; do NOT fill in with plausible causes from other providers'
     postmortems. -->

## 6. Lessons the learner can extract
L1. Status-page polling must assume the vendor updates after we see
    impact; our fallback contract (ex-02) must not wait for the status
    page to flip.
L2. Partial degradation of one model family while others are healthy
    requires per-model, not per-provider, circuit breakers.
L3. DDoS-pattern events (or anything front-door driven) can take the
    vendor's auth and billing APIs with them, not just inference; our
    auth path must tolerate a cold cache across the ride.
L4. Multi-provider is only useful if the fallback does not share the
    same auth plane, egress path, or shared L7 vendor; verify in
    ex-01 that fallback-a is independent of primary at those layers.
L5. Public comms channels (status page, vendor's social account) are
    not an SLA. Our comms to our users must not depend on the
    vendor's cadence.
L6. "ChatGPT is down" trends on social media reach our support team
    before our monitoring reaches our on-call; a runbook should cover
    the comms direction for that case even when our users are fine.

## 7. Our-team mapping (example rows)
| Lesson | Our current control | Gap | Owner |
|---|---|---|---|
| L1 | Status page scraped every 60 s and routed to #alerts | We only alert on `major_outage`; we miss `partial_outage` | SRE |
| L2 | Per-provider circuit breaker | No per-model breaker; model-X failure takes down model-Y traffic for same provider | gateway |
| L3 | Vendor API key rotation runbook | Rotation path assumes vendor auth API is up — no offline bootstrap | security |
| L4 | Fallback contract exists (ex-02) | Both providers front-ended by same CDN vendor | platform |
| L5 | Support macros reference vendor status page | Macros auto-link vendor language; need internal-first messaging | support ops |
| L6 | None | No "vendor-looks-down-but-we're-fine" runbook | on-call |

## 8. Ticket backlog
- [ ] <JIRA-xxx> — Add `partial_outage` to alert fan-in rules
- [ ] <JIRA-xxx> — Add per-model circuit breaker keyed on model id
- [ ] <JIRA-xxx> — Document offline-bootstrap flow for key rotation
- [ ] <JIRA-xxx> — Audit CDN / L7 commonality across fallback group
- [ ] <JIRA-xxx> — Separate internal support macro from vendor copy
- [ ] <JIRA-xxx> — Add "vendor-degraded-user-unaffected" runbook
```

### 2.3 How to assemble the primary-source bundle

1. **Start at the vendor status page history URL.** For each vendor in
   the surface map (ex-01) at tier-1, pull the status history over the
   case-study window.
2. **Filter to impacting incidents.** Ignore `maintenance` and
   `minor_outage` unless a measurable business impact is also present.
3. **For each incident, capture:** incident ID, start/end timestamps
   in UTC (not local), affected components, the vendor's final-update
   text verbatim, and a direct URL to the incident.
4. **If a public postmortem exists**, link it. Do not summarize; quote.
5. **Corroborate blast radius** with at least one independent source
   dated within the incident window. Independent sources are useful
   for *who was affected* (downstream products, geographic skew), not
   for mechanism claims.
6. **If the vendor has not disclosed a cause**, state that. The
   absence of disclosure is itself a reliability finding for a
   dependent team.

### 2.4 Guard against invented facts

The single biggest hazard with a case study of publicly-remembered
outages is stating as fact a plausible-sounding number from memory.
Enforce in review:

- No wall-clock durations without a status-page citation.
- No percentages (availability, error rate, user share) without a
  citation.
- No root-cause verbs ("misconfiguration", "deploy", "hardware",
  "DDoS", "cert expiry") unless the vendor used them.
- No "subsequent frontier-lab" claims that aggregate vendors unless
  each component is individually sourced; otherwise write
  "<!-- cite-primary-source: vendor+incident -->".

A reviewer must be able to click every non-trivial claim.

## 3. Validation steps

1. **Source-citation lint.** Every status-page claim is a Markdown
   link to a vendor-controlled host. A short script can grep each
   statement and flag unlinked claims. Suggested simple check:
   each row in the Timeline table must have a non-empty `Source`
   cell with a `https://` URL.
2. **No-invented-fact lint.** Grep for the following in the finished
   doc and treat each as a block if it is not a direct quote:
   percentages (`\d+%`), durations longer than a sentence
   (`\b\d+\s*(minutes?|hours?|days?)\b`), and the verbs listed in
   §2.4 above.
3. **cite-primary-source gate.** Count `cite-primary-source` tags. Zero is the
   target for merge. Any remaining tag blocks auto-merge per the
   module policy and must be either resolved or justified in a PR
   comment.
4. **Independent check.** A reviewer who did not write the doc
   opens each cited URL and confirms the paraphrased claim is
   supported by the destination.
5. **Our-team mapping completeness.** Every lesson in §6 has exactly
   one row in §7 and an assigned owner. Lessons without assigned
   owners are converted to `cite-primary-source: owner` tags.
6. **Backlog linkage.** Each row in §8 either links a real ticket or
   is marked `[ ] draft — must be filed within one week of case
   study publication`.

## 4. Rubric

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Primary sourcing** | Claims lack citations | Most claims cited; a few unsourced | Every non-trivial claim links to a vendor-controlled page or a dated independent corroboration |
| **No invented facts** | Specific numbers / mechanisms stated without citation | A few unquoted specifics remain | Clean: numbers and mechanism verbs appear only as quoted source material |
| **Timeline** | Missing or reconstructed from memory | Present, partial sources | Complete, UTC, every row sourced |
| **Mechanism honesty** | Guesses at cause when vendor silent | Mixes vendor statement with speculation | States exactly and only what the vendor disclosed; flags under-disclosure explicitly |
| **Lessons** | Generic ("have retries") | Specific to incident | Specific and directly tied to a control the team either has or lacks |
| **Our-team mapping** | Not present | Present but incomplete owners/gaps | Every lesson mapped; owner named; gap concrete |
| **Backlog** | Absent | Draft TODO list | Linked tickets with owners and one-sentence rationale |
| **cite-primary-source tags** | Many unresolved tags, no plan | Few tags remain, plan exists | Zero remaining tags before merge |
| **Portability** | One-off write-up | Reusable template extracted | Template saved in the module's shared-artifacts area; next case study swaps the contents without rewriting the frame |

Passing: average ≥ 1.5 with no 0s on **Primary sourcing**, **No
invented facts**, and **cite-primary-source tags**.

## 5. Common mistakes

- **"Writing from memory."** Confident paraphrases of what happened
  during a well-known outage are the most common failure mode here.
  If a specific claim does not have a URL under it, it is not in the
  submission.
- **Over-reading a short status-page post.** Vendor status posts are
  often terse by design. Resist the urge to turn "elevated errors on
  the API" into a mechanism; it is not one.
- **Aggregating across vendors.** "Frontier labs all had outages in
  2023–2024" is a true but useless generalization. Treat each vendor
  and each incident separately; aggregate only in the Lessons section
  where the pattern is explicitly identified.
- **Treating press coverage as mechanism source.** News reporting is
  valuable for blast-radius signals but should not be cited for
  "the cause was X." Vendor sources only for cause.
- **Blaming the vendor in the Lessons section.** The exercise is to
  learn, not to grade the vendor. Phrase lessons as "our control
  gap," not "they should have."
- **Lessons that fire-and-forget.** A lesson with no row in the
  mapping table and no ticket is a philosophy statement. Remove or
  convert.
- **Confusing partial outage with total outage.** Many real
  frontier-lab incidents affected only one surface (chat vs.
  embeddings vs. one model) or one region. Reconstructing the
  incident as total erases the key lesson about per-feature
  circuit breakers.
- **Skipping the independence check.** If the "lesson" is "we need
  multi-provider," verify in ex-01 that the fallback is actually
  independent at auth, CDN, and egress layers — otherwise the lesson
  is unfilled even after the ticket closes.

## 6. References

### Primary sources the learner is expected to consult

- **OpenAI status page and history** —
  <https://status.openai.com/> and <https://status.openai.com/history>.
  The authoritative source for vendor-reported incidents on OpenAI
  services. Pull incident IDs and vendor-stated causes from here.
- **Anthropic status page** — <https://status.anthropic.com/>.
  Authoritative source for Anthropic service availability events.
- **Google Cloud Vertex AI / Gemini status** —
  <https://status.cloud.google.com/>. Authoritative source for
  provider-side Gemini / Vertex availability events.
- **Vendor engineering blogs**, when postmortems are published there
  in addition to the status page. Link each explicitly.

### Standards this exercise anchors to

- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. The
  `MANAGE 4.1` and `MANAGE 4.3` subcategories cover incident
  response, post-incident learning, and feedback loops for
  third-party components. The case-study artifact is the operational
  evidence for those subcategories.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  Case studies of upstream outages reinforce `ML06 AI Supply Chain
  Attacks` in a reliability (not only security) framing — our
  posture must not assume provider availability.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The `AML.T0010 ML
  Supply Chain Compromise` technique applies: provider outages are
  the reliability analogue of supply-chain risk, and the case
  study's lessons become threat-modeling inputs.

### Upstream / downstream artifacts in this module

- `exercise-01-vendor-dependency-surface-map` provides the vendor
  list the case study iterates through and the "independence" check
  Lessons map against.
- `exercise-04-provider-status-ingestion-into-incident-pipeline`
  provides the automated feed that, if it had existed, would have
  shortened the Timeline's detection gap. Lessons that call it out
  should link to the ex-04 artifact.
- `exercise-05-sla-vs-slo-reconciliation-and-error-budget-sharing`
  uses the Timeline's duration to compute how much of our error
  budget a vendor outage consumed — the case study's numbers are
  inputs to that reconciliation.

<!-- cite-primary-source: before merging, replace every cite-primary-source tag
     in this file with a citation to a specific vendor status-page
     incident URL and quoted language. This tag is required by the
     module source policy and is what blocks auto-merge. -->
