# Exercise 06 — Vendor Evaluation Scorecard for Single vs. Multi-Provider (Solution)

## 1. Solution overview

The decision between single-provider and multi-provider for a model-
backed surface is not a philosophical one; it is a scored trade-off
across capability, reliability, security, portability, cost, and
operational load. Teams that treat this as a reflex ("multi-provider
is always better" or "multi-provider is premature") pay for the
mistake twice: once in the direct trade-off they missed, and again
in sunk effort when they reverse course.

This exercise produces a scorecard that:

1. Scores each candidate option — single provider A, single provider
   B, primary+fallback, true multi-provider (active-active) — on a
   consistent rubric.
2. Weights the rubric for *this surface*, not in the abstract. A
   batch embedding pipeline weighs portability and cost differently
   than a real-time chat surface.
3. Produces a decision record the team can reread in 6–12 months to
   evaluate whether the decision has aged well.

The scorecard reuses artifacts from the earlier exercises (surface
map, fallback contract, SLO reconciliation) rather than inventing new
data, and grounds its rows in the OWASP ML Top 10, NIST AI RMF, and
MITRE ATLAS controls those earlier artifacts already reference.

## 2. Worked implementation

### 2.1 Scope and candidate options

Before scoring, declare the decision scope and the candidates. These
belong at the top of the scorecard document.

```markdown
Decision scope:
  Surface: product-chat (ref ex-01)
  Timeframe: 6-month horizon
  Owner: inference-platform
  Reviewers: SRE lead, product lead, security lead, procurement

Candidate options:
  O1. Single provider — vendor A (status quo)
  O2. Single provider — vendor B (migration from A)
  O3. Primary + hot fallback — A primary, B fallback (ex-02 contract)
  O4. Active-active — both A and B serve traffic, routed by policy
  O5. Primary + self-hosted fallback — A primary, open-weights
      self-hosted (highest control, highest ops load)
```

Candidate `O5` is called out because teams often forget that
"multi-provider" can include an internal self-hosted option, which has
different scoring across almost every row.

### 2.2 Dimensions

Each dimension scores 0–3. Definitions below are the *rubric for the
rubric*; keep them in the scorecard verbatim so reviewers scoring
different options use the same scale.

| # | Dimension | 0 | 1 | 2 | 3 |
|---|---|---|---|---|---|
| D1 | **Model capability fit** | Fails on a required capability (tool calling, vision, structured output, ctx length) | Partial — some surfaces need prompt rewrites | All required capabilities present with minor caveats | Equivalent or superior to the current best-in-class for this surface, documented in equivalence tests |
| D2 | **Quality headroom** | Measurably worse on the surface's eval harness | Comparable within noise | Comparable with per-prompt wins/losses | Consistently better on the surface's harness |
| D3 | **Reliability ceiling (ex-05)** | Lowers the computed SLO ceiling | Neutral | Raises ceiling by independent fallback | Raises ceiling AND reduces blast radius for shared-vendor incidents |
| D4 | **Independence (ex-01)** | Shares auth, CDN, or egress with the primary | Independent at API, same L7 vendor | Independent at API + L7, same cloud region | Independent at API + L7 + cloud + region |
| D5 | **Capability drift risk** | Vendors A and B diverge meaningfully on >1 capability | One capability gap bridged by caller code | Negligible gap; equivalence tests pass | Gap is documented, bounded, and expected to shrink |
| D6 | **Data handling** | Non-compliant with surface's DPA needs | Compliant with manual configuration | Compliant by default, verified by security review | Compliant + supports data-residency, training-opt-out, retention controls at parity |
| D7 | **Portability / lock-in** | Fine-tuned weights, vendor-native features bind us | Non-trivial migration effort | Small per-provider adapter | Zero lock-in; OpenAI-shape caller works unchanged |
| D8 | **Cost (unit + commit)** | >1.5× current unit cost without offset | Within ±25% | Within ±10% | Lower unit cost or better commit economics |
| D9 | **Operational load** | Materially more pager minutes / incident attribution complexity | Comparable | Reduces pager load (true redundancy) | Reduces pager load AND reduces incident attribution complexity |
| D10 | **Contractual posture** | No DPA / SLA / enterprise terms | Standard self-serve terms | Signed DPA + SLA; standard remedies | Signed DPA + SLA + explicit incident-comms + data-handling attestations |
| D11 | **Supply-chain / security (OWASP ML06, MITRE AML.T0010)** | Weights/model provenance unverifiable; single trust anchor | Public policy, no attestations | Vendor publishes model provenance and tenancy model; we verified it | Multi-anchor trust: our fallback is from an independent provenance tree |
| D12 | **Observability parity** | Vendor hides streaming / token-level telemetry | Partial parity | Parity in logs and traces for both vendors | Parity + unified SLI calculation across vendors |
| D13 | **Compliance with internal risk framework (NIST AI RMF MANAGE 2.1)** | Not addressed | Mentioned in surface-map comments | Explicit MANAGE evidence linked | MANAGE evidence linked AND the policy has been tested in a drill |

Not every dimension applies to every surface; mark `N/A` and exclude
from averages when genuine. `N/A` as an escape hatch for "we didn't
check" is a review finding.

### 2.3 Weights

Weights are the mechanism by which the generic rubric becomes
surface-specific. Weights must sum to 100 and be set *before* scoring
each candidate (otherwise weights become rationalizations).

Three example weight profiles the team can start from:

```yaml
# scorecard/weight-profiles.yaml
profiles:
  product-chat-realtime:
    description: Real-time user-facing chat with strict latency + safety needs
    weights:
      D1: 10   # capability fit
      D2: 10   # quality
      D3: 15   # reliability ceiling
      D4:  8   # independence
      D5:  5   # capability drift
      D6:  8   # data handling
      D7:  4   # portability
      D8:  5   # cost
      D9:  8   # ops load
      D10: 8   # contractual
      D11: 7   # supply chain
      D12: 7   # observability parity
      D13: 5   # NIST MANAGE

  batch-embedding-pipeline:
    description: Nightly batch embedding — recomputable, less latency-sensitive
    weights:
      D1:  8
      D2: 10
      D3:  8
      D4:  5
      D5:  8
      D6: 10
      D7: 12   # portability matters because we re-embed often
      D8: 15   # cost dominates at batch scale
      D9:  6
      D10: 5
      D11: 5
      D12: 4
      D13: 4

  internal-eval-judge:
    description: Judge model for release-gate evals; correctness > uptime
    weights:
      D1:  8
      D2: 20   # quality dominates
      D3:  4   # eval can retry offline
      D4:  3
      D5: 12   # drift breaks score comparability
      D6:  8
      D7:  6
      D8:  5
      D9:  5
      D10: 6
      D11: 8
      D12: 6
      D13: 9
```

### 2.4 The scorecard template

```markdown
# Scorecard: product-chat — single vs multi-provider

Weight profile: product-chat-realtime (ref §2.3)
Scored on: <date>
Scored by: <names + roles>

## Scores

| Dim | Weight | O1 (A only) | O2 (B only) | O3 (A+B fallback) | O4 (A+B active-active) | O5 (A + self-hosted) |
|---|---|---|---|---|---|---|
| D1 | 10 | 3 | 2 | 3 | 3 | 2 |
| D2 | 10 | 3 | 2 | 3 | 2 | 1 |
| D3 | 15 | 1 | 1 | 2 | 3 | 3 |
| D4 |  8 | 0 | 0 | 2 | 2 | 3 |
| D5 |  5 | 3 | 3 | 2 | 1 | 1 |
| D6 |  8 | 2 | 2 | 2 | 2 | 3 |
| D7 |  4 | 1 | 1 | 2 | 2 | 3 |
| D8 |  5 | 2 | 3 | 2 | 1 | 1 |
| D9 |  8 | 3 | 3 | 2 | 1 | 0 |
| D10|  8 | 2 | 2 | 2 | 2 | 2 |
| D11|  7 | 1 | 1 | 2 | 2 | 3 |
| D12|  7 | 2 | 2 | 2 | 1 | 1 |
| D13|  5 | 1 | 1 | 2 | 2 | 2 |

## Weighted totals

Each cell above is multiplied by the weight; sum down each column.

| | O1 | O2 | O3 | O4 | O5 |
|---|---|---|---|---|---|
| Weighted sum | <sum> | <sum> | <sum> | <sum> | <sum> |
| Normalized (0–100) | <sum/3> | ... | ... | ... | ... |

## Dimension commentary

For every score ≤ 1, write one sentence of justification citing an
artifact: the ex-01 surface-map row, an ex-02 drill result, an
ex-04 incident, an ex-05 attribution table, or a vendor document.

## Decision

Chosen option: Oᴺ
Rationale (3–5 bullets):
  - ...
Residual risks accepted:
  - ...
Review date: <+6 months>
Trigger-based review:
  - If vendor A's D3 drops (observed burn > ex-05 soft cap for two
    consecutive windows), revisit.
  - If an equivalence test diverges by >10% on the surface harness,
    revisit.
```

The numbers above are illustrative, not prescriptive. The point is
the *shape* of the artifact: weighted rows, sourced justifications
for low scores, explicit review triggers.

### 2.5 Walkthrough — how a scoring session actually runs

1. **Pre-work (day 0).** The author collects the ex-01 surface-map
   row, the ex-02 fallback contract and its most recent drill
   results, the ex-05 ceiling math for each candidate, and the last
   quarter's incident attribution. Drafts the weight profile.
2. **Session (2 hours, 4 reviewers).** SRE lead walks each dimension;
   each reviewer scores independently on sticky notes / shared
   doc without seeing others' scores (Delphi-style first round);
   discuss any cell with a spread > 1 point; converge or record the
   spread with a note.
3. **Low-score justification pass.** Any cell ≤ 1 is written up with
   the artifact link. Cells that cannot be justified are revisited
   or marked `N/A`.
4. **Sensitivity check.** The author shifts each weight by ±3 points
   (one at a time) and checks whether the winning option changes. If
   the winner flips on small perturbations, the decision is fragile;
   split the top two options into a short follow-up.
5. **Commit.** The scorecard is checked into the repo alongside the
   fallback contract (ex-02) with the review date and triggers in
   the frontmatter.

### 2.6 Single-vs-multi — the honest trade-offs

Even with a weighted scorecard, the broad pattern is worth stating
plainly so the author knows what the math usually says:

- **Multi-provider raises D3, D4, D11 (reliability ceiling,
  independence, supply chain).** Not for free: it lowers D5 (drift),
  D9 (ops load), and sometimes D12 (observability parity).
- **Single provider raises D2 (quality), D5, D9, D12.** It costs
  D3 and D11, and leaves D7 (portability) dependent on how
  vendor-native the implementation is.
- **Primary + fallback (O3) is the usual sweet spot** for real-time
  surfaces that need multi-provider for reliability but cannot
  tolerate the D5/D9/D12 cost of active-active. It also matches
  what ex-02 is engineered to deliver.
- **Active-active (O4) pays off when a single provider cannot meet
  the SLO ceiling even after fallback** (ex-05 §C), and the surface
  can tolerate per-request drift. Batch pipelines and multi-model
  ensembles are the natural fit.
- **Self-hosted (O5) wins D4/D11 outright** and is the only option
  that fully breaks shared-cloud risk. It pays with D2 (unless the
  self-hosted model genuinely matches), D9 (ops load is real), and
  D8 (fixed cost regardless of volume).

The scorecard exists so the choice is traceable, not so the choice
is forced.

### 2.7 Review and refresh

- **Review date on the scorecard.** Default 6 months. Earlier if a
  trigger fires (§2.4 Decision section).
- **Weight review.** Weights are revisited whenever the surface's
  SLO or scope changes materially. Scores without current weights
  are not a decision; they are history.
- **Archive, do not overwrite.** Store old scorecards under
  `scorecards/<surface>/<yyyy-mm-dd>.md`. The chain of decisions is
  the audit trail that matters when someone asks "why do we do it
  this way?"

## 3. Validation steps

1. **Schema lint.** The YAML weight profile parses and sums to 100.
   A short script flags profiles that drift.
2. **Score-complete check.** No cells are empty; `N/A` has a
   justification.
3. **Low-score sourcing.** Every cell ≤ 1 links to an artifact
   path or URL. Scores without sources fail review.
4. **Sensitivity record.** The scorecard includes the ±3-point
   sensitivity result (does the winner change?).
5. **Cross-artifact consistency.** The scorecard's D3 row must be
   consistent with the ex-05 ceiling math; its D4 row must be
   consistent with the ex-01 independence record; its D1/D5 rows
   must be consistent with the ex-02 equivalence test outcome.
   Diffs between artifacts are findings.
6. **Review date set.** Scorecard without a future review date is
   not in effect.
7. **Reviewer diversity.** At least one SRE, one product, and one
   security reviewer signed. Procurement signs on D8, D10.
8. **Reusability.** The weight-profile file can be re-used for the
   next surface without edits that are specific to one vendor's
   marketing.

## 4. Rubric

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Decision framing** | Option space narrow (only "do we switch?") | Lists ≥2 options | Lists at least O1, O3, and one of O4/O5; declares scope and reviewers |
| **Rubric completeness** | Fewer than 8 dimensions scored | 8–10 dimensions | All 13 dimensions (or explicit N/A) |
| **Weight discipline** | No weights or set after scoring | Weights set before scoring | Weight profile reused from a library; profile name recorded |
| **Score sourcing** | No justifications | Some low-score justifications | Every ≤1 score links an artifact path or URL |
| **Cross-artifact consistency** | Scores contradict ex-01/02/05 | Minor drift | Consistent; diffs explained |
| **Sensitivity analysis** | Absent | Narrative mention | ±3 sweep recorded; winner stability noted |
| **Review triggers** | None | Date only | Date + measurable triggers (burn exceedance, equivalence drift) |
| **Standards traceability** | None | Mentions one of OWASP/NIST/MITRE | Dimensions tagged with the specific control (ML06, MANAGE 2.1, AML.T0010) |
| **Reviewer diversity** | Single reviewer | Two roles | SRE + product + security, procurement for cost/contract rows |

Passing: average ≥ 1.5 with no 0s on **Rubric completeness**,
**Score sourcing**, **Cross-artifact consistency**.

## 5. Common mistakes

- **Scoring before weighting.** Scorers unconsciously weight toward
  the option they already prefer. Setting weights first and keeping
  them visible during scoring is the single highest-leverage
  practice.
- **Letting D8 (cost) carry the decision alone.** Cost is a real
  input, but it is 5–15 points of 100 in these profiles for a
  reason. When D8 dominates the final ranking, check whether the
  weight profile matches the surface's real risk tolerance.
- **"Multi-provider" without a tested fallback contract.** A
  scorecard that scores O3 highly without the ex-02 contract and a
  recent drill is rewarding an aspiration. Dock D3/D9 until the
  drill is green.
- **Skipping independence (D4).** Two providers on the same CDN or
  same cloud region are not two providers for reliability purposes.
  The ex-01 map must back the D4 score.
- **Ignoring O5 (self-hosted).** Even if the team will not choose it,
  including it on the scorecard forces the right questions about
  independence, lock-in, and supply chain. Its absence is itself a
  finding.
- **Scoring in a room, archiving in a drawer.** A scorecard that is
  not checked in, versioned, and triggered-review'd rots quickly —
  the vendor landscape changes faster than team memory.
- **Letting the weights wander mid-session.** When scoring reveals
  the weights were off, finish the current session and revisit the
  weights in a separate session. Mixing weight edits with scoring
  invites motivated reasoning.
- **No link between the scorecard and operational artifacts.** A
  decision that does not show up in the fallback contract, the
  ingester routing, or the error-budget policy is not a decision;
  it is a document.

## 6. References

- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  `ML06 AI Supply Chain Attacks` is the rubric-level anchor for D11.
  `ML05 Model Theft` motivates D6 (data handling) when prompts or
  outputs contain sensitive content that leaves our boundary.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
  `GOVERN 6.1` (third-party accountability), `MAP 4.1` (identifying
  third-party components), `MANAGE 2.1` (risk-response options), and
  `MEASURE 2.6` (monitoring third-party components) are the four
  AI-RMF subcategories the scorecard provides evidence for. D13 is
  explicitly tied to MANAGE 2.1.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. `AML.T0010 ML
  Supply Chain Compromise` is the ATLAS technique D11 defends
  against; multi-anchor trust (independent provenance for
  fallbacks) is the score-3 bar.
- **Google SRE Workbook, "Implementing SLOs"** —
  <https://sre.google/workbook/implementing-slos/>. The
  weight-profile mechanic in §2.3 — surface-specific weighting of
  a shared rubric — mirrors the SLO-first principle that
  reliability targets are surface-specific, not universal.
- **Upstream artifacts in this module.** `exercise-01` (rows in the
  surface map back D4, D6, D11), `exercise-02` (fallback contract
  and drill results back D1, D3, D5, D9), `exercise-04` (status
  ingester provides D12 parity evidence), `exercise-05` (ceiling
  math backs D3; attribution backs D9), `exercise-03` (any
  incidents flagged there feed the "trigger-based review" clauses).
