# Exercise 06 — DORA Four Keys for Model Releases — Reference Solution

Paired learning file: `lessons/mod-111-release-engineering-and-progressive-delivery/exercises/exercise-06-dora-four-keys-for-model-releases.md`.

> **Scope note.** The paired exercise asks the learner to adapt the DORA "Four Keys" metrics (Deployment Frequency, Lead Time for Changes, Change Failure Rate, Time to Restore Service) to model-backed services and produce (a) the metric definitions as adapted, (b) the ingestion pipeline that reads the release ledger produced by exercises 01, 03, 04, and (c) a dashboard and review cadence. The reference solution uses the illustrative `chatcat` product and the release ledger shape produced by the upstream exercises.

## 1. Solution overview

The DORA "Four Keys" research (Google Cloud and the DORA team) names four metrics that correlate with elite software-delivery performance; the research and definitions are published in *Accelerate* (Forsgren, Humble, Kim) and in the annual [*State of DevOps* reports](https://dora.dev/) and [*Google Cloud / Four Keys* project](https://github.com/dora-team/fourkeys). The four keys are:

- **Deployment Frequency** — how often the organisation successfully releases to production.
- **Lead Time for Changes** — how long it takes a commit to reach production.
- **Change Failure Rate** — the percentage of deployments causing a failure in production.
- **Time to Restore Service** — how long it takes to restore service after a failure.

The metrics translate to model-backed services with three adjustments:

1. **Count model-only releases as deployments.** A provider-version bump, a prompt-template change, or a retrieval-index rebuild is a change that ships user-visible behaviour, even if no container image moved. The release ledger (exercises 01, 03, 04) records these; the DORA pipeline reads the ledger, not the container registry.
2. **Expand "failure" beyond the SLO definition.** A release that passes the SLO gate but fails the eval gate — a measurable quality regression — is a change failure. The release ledger's verdict is authoritative.
3. **"Restore service" has two clocks.** For SLO-only incidents, the clock stops at SLO-restored. For eval-only incidents (quality regression), the clock stops at *shipped-fix-passes-gate*. The faster the ledger closes those, the closer the metric reflects real user experience.

### 1.1 Decision rationale — why read the ledger, not git

- Many production changes are **not git commits**: a feature flag flip, a provider-side model version bump, a prompt revision in a prompt store, a retrieval-index rebuild.
- A git-anchored DORA pipeline undercounts deployments and thus overestimates quality (change failure rate denominators are smaller than reality).
- The release ledger is the one place every change — from exercises 01, 03, 04 — is recorded, so it is the natural source of truth for the DORA keys.

### 1.2 Metric definitions as adapted

| Key | Standard definition | Adapted definition | Source field |
|-----|---------------------|---------------------|--------------|
| **Deployment Frequency** | Count of successful production deployments in a window | Count of release-ledger entries where `status = shipped` AND `route in production_routes` AND `strategy ∈ {canary, blue-green, shadow-promotion, flag-flip}` | `ledger.deployment_frequency_eligible` (computed) |
| **Lead Time for Changes** | Median time from code-committed to code-in-production | Median of `shipped_at - change_anchored_at`, where `change_anchored_at` is the git commit, prompt-store commit, or flag-change timestamp (whichever applies) | `ledger.shipped_at - ledger.change_anchor.at` |
| **Change Failure Rate** | % of deployments causing a failure in production | % of `shipped` entries that are followed within 24 h by either (a) a release-ledger abort/rollback referring to this change, (b) an incident post-mortem naming this release in `probable_cause_change`, or (c) a quality regression verdict from the release-gate service naming this build | `ledger.change_failed`  (computed) |
| **Time to Restore Service** | Time to restore service after a failure | For SLO failures: `slo_restored_at - slo_violated_at`. For eval failures: `fix_shipped_at - regression_detected_at`. Reported separately and in combination. | `ledger.restored_at - ledger.failure_detected_at` |

## 2. Implementation — worked answer

### 2.1 Release-ledger schema recap

Entries are appended by exercises 01 (canary), 03 (shadow-promotion), 04 (blue/green), and by the incident / rollback tooling (`mod-108`). The schema the DORA pipeline reads:

```json
{
  "release_id": "rel-2026-11-13-chatcat-serving-g42a3",
  "service": "chatcat-serving",
  "route": "faq",
  "strategy": "canary | shadow-promotion | blue-green | flag-flip | rollback",
  "change_anchor": {
    "kind": "git | prompt-store | provider-version | flag",
    "ref": "sha-abc... | commit-xyz... | gpt-4o-mini-2024-07-18 | flag-name",
    "at": "2026-11-13T09:30:00Z"
  },
  "shipped_at": "2026-11-13T14:08:00Z",
  "status": "shipped | aborted | rolled_back",
  "gate_verdict_ref": "release-gate.chatcat.json@v15;verdict-id-...",
  "failure": {
    "detected_at": "2026-11-13T15:40:00Z",
    "cause_class": "slo | eval | both | none",
    "restored_at": "2026-11-13T16:22:00Z"
  } | null,
  "related_incident_ref": "inc-2026-11-13-rb-ttft-regression-chatcat" | null
}
```

### 2.2 Ingestion pipeline — `dora/ingest.py`

```python
from __future__ import annotations
import json
from dataclasses import dataclass
from datetime import datetime, timedelta
from pathlib import Path
from statistics import median


@dataclass
class LedgerEntry:
    release_id: str
    service: str
    route: str
    strategy: str
    change_anchor_at: datetime
    change_anchor_kind: str
    shipped_at: datetime | None
    status: str
    failure_detected_at: datetime | None
    failure_cause_class: str | None
    failure_restored_at: datetime | None


def _parse(path: Path) -> list[LedgerEntry]:
    rows = [json.loads(line) for line in path.read_text().splitlines() if line.strip()]
    out: list[LedgerEntry] = []
    for r in rows:
        shipped_at = datetime.fromisoformat(r["shipped_at"]) if r.get("shipped_at") else None
        failure = r.get("failure") or {}
        out.append(LedgerEntry(
            release_id=r["release_id"],
            service=r["service"],
            route=r["route"],
            strategy=r["strategy"],
            change_anchor_at=datetime.fromisoformat(r["change_anchor"]["at"]),
            change_anchor_kind=r["change_anchor"]["kind"],
            shipped_at=shipped_at,
            status=r["status"],
            failure_detected_at=datetime.fromisoformat(failure["detected_at"]) if failure.get("detected_at") else None,
            failure_cause_class=failure.get("cause_class"),
            failure_restored_at=datetime.fromisoformat(failure["restored_at"]) if failure.get("restored_at") else None,
        ))
    return out


_PROD_STRATEGIES = {"canary", "shadow-promotion", "blue-green", "flag-flip"}


def deployment_frequency(entries: list[LedgerEntry], since: datetime, until: datetime) -> dict:
    eligible = [e for e in entries
                if e.status == "shipped"
                and e.strategy in _PROD_STRATEGIES
                and e.shipped_at is not None
                and since <= e.shipped_at < until]
    return {
        "window_start": since.isoformat(),
        "window_end": until.isoformat(),
        "deployments": len(eligible),
        "per_day": len(eligible) / max(1.0, (until - since).total_seconds() / 86400),
    }


def lead_time(entries: list[LedgerEntry], since: datetime, until: datetime) -> dict:
    eligible = [e for e in entries
                if e.status == "shipped"
                and e.shipped_at is not None
                and since <= e.shipped_at < until]
    if not eligible:
        return {"n": 0, "median_seconds": None, "p75_seconds": None}
    secs = sorted((e.shipped_at - e.change_anchor_at).total_seconds() for e in eligible)
    def _p(q):
        idx = int(q * (len(secs) - 1))
        return secs[idx]
    return {"n": len(eligible), "median_seconds": median(secs), "p75_seconds": _p(0.75)}


def change_failure_rate(entries: list[LedgerEntry], since: datetime, until: datetime, failure_window: timedelta = timedelta(hours=24)) -> dict:
    eligible = [e for e in entries
                if e.status == "shipped"
                and e.shipped_at is not None
                and since <= e.shipped_at < until]
    if not eligible:
        return {"n": 0, "rate": None}
    failed = [e for e in eligible if e.failure_detected_at is not None and e.failure_detected_at - e.shipped_at <= failure_window]
    return {"n": len(eligible), "failed": len(failed), "rate": len(failed) / len(eligible)}


def time_to_restore(entries: list[LedgerEntry], since: datetime, until: datetime) -> dict:
    failures = [e for e in entries
                if e.failure_detected_at is not None
                and e.failure_restored_at is not None
                and since <= e.failure_detected_at < until]
    if not failures:
        return {"n": 0, "slo": None, "eval": None, "combined": None}

    def _median(xs):
        return median(xs) if xs else None

    slo_secs = [(e.failure_restored_at - e.failure_detected_at).total_seconds()
                for e in failures if e.failure_cause_class in ("slo", "both")]
    eval_secs = [(e.failure_restored_at - e.failure_detected_at).total_seconds()
                 for e in failures if e.failure_cause_class in ("eval", "both")]
    all_secs = [(e.failure_restored_at - e.failure_detected_at).total_seconds() for e in failures]

    return {
        "n": len(failures),
        "slo_median_seconds": _median(slo_secs),
        "eval_median_seconds": _median(eval_secs),
        "combined_median_seconds": _median(all_secs),
    }


def compute_window(path: Path, since: datetime, until: datetime) -> dict:
    entries = _parse(path)
    return {
        "deployment_frequency": deployment_frequency(entries, since, until),
        "lead_time": lead_time(entries, since, until),
        "change_failure_rate": change_failure_rate(entries, since, until),
        "time_to_restore": time_to_restore(entries, since, until),
    }
```

### 2.3 Daily job

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dora-four-keys-chatcat
  namespace: chatcat
spec:
  schedule: "0 6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: dora
              image: ghcr.io/example/dora-four-keys:1.0.0
              args:
                - compute
                - --ledger=s3://chatcat-releases/ledger.ndjson
                - --window=28d
                - --out=s3://chatcat-releases/dora/daily-$(date +%F).json
```

### 2.4 Dashboard shape

The Grafana dashboard reads the daily JSON and renders four panels, each with the four DORA performance bands from the *Accelerate* / *State of DevOps* taxonomy (Elite / High / Medium / Low). Bands are drawn as reference ranges, not hard thresholds — the point of the metric is movement over time, not punishment of a single bad week.

- Panel **Deployment Frequency**: deploys-per-day over 28 days, broken down by `strategy`. A sudden drop in `canary` strategy with a rise in `flag-flip` is diagnostic (team is working around the pipeline, not improving it).
- Panel **Lead Time for Changes**: median and p75 over 28 days, broken down by `change_anchor_kind`. Prompt-store changes should be fast; git changes depend on CI.
- Panel **Change Failure Rate**: 28-day rate, broken down by `failure.cause_class`. Trend lines separate SLO-caused and eval-caused failures; if eval-caused is rising while SLO-caused is flat, the eval gate is doing its job.
- Panel **Time to Restore Service**: 28-day median, broken down by `failure.cause_class`. The *eval* clock is usually hours-to-days longer than the *SLO* clock, and that gap is the honest number. A single combined MTTR obscures it.

### 2.5 Review cadence

- **Weekly.** The release lead reviews the dashboard with the on-call engineer. Any week with a change-failure-rate above the team's last-quarter median triggers a 15-minute "what fired, what didn't" review.
- **Quarterly.** The function uses the metric trend to prioritise investments: if Lead Time rose and was caused by eval gate waits, the eval pipeline is the investment target; if Deployment Frequency fell and was caused by canary aborts, the release-gate contract (exercise 05) is the investment target.
- **Annual.** The function sets a published target band (Elite / High / Medium / Low) for the next year. The target is reviewed, not the result; the metric is diagnostic, not a performance review.

### 2.6 Known traps the adaptation addresses

- A provider-only model bump counts as a deployment. Teams that run on hosted models would otherwise show "0 deployments" and misread themselves.
- A prompt-store change counts as a deployment. These are often the most frequent user-visible changes.
- A rolled-back canary is a failure, not a non-event. The ledger's `aborted` status feeds the CFR denominator *and* numerator.
- Eval-caused incidents have their own restore clock; mixing them into SLO MTTR hides the quality-side latency.

## 3. Validation steps

1. **Ingestion unit tests** (`pytest dora/tests/`):
   - A ledger with one `shipped` entry inside the window and one outside returns `deployments = 1`.
   - A shipped entry with `failure.detected_at` inside the 24 h window increments CFR numerator.
   - An entry without `failure` is not counted as a change failure.
   - A rollback entry (`status = rolled_back`) does not count as a deployment, but does count as the restore event for the release it rolls back.

2. **Fixture reproduction.** The reference solution ships `fixtures/ledger-28d.ndjson` with 120 entries including every status × cause_class combination. Running `python -m dora.ingest compute --ledger fixtures/ledger-28d.ndjson --window 28d` must produce the expected JSON in `fixtures/expected.json` byte-for-byte.

3. **Dashboard renders.** Import the Grafana dashboard JSON into a scratch instance pointed at the fixture file; confirm all four panels render with four bands.

4. **Band-alignment sanity.** For the shipped fixtures:
   - Deployment Frequency around 4–5 / day falls in the "High" band.
   - Lead Time median around 24 h falls in the "High" band.
   - CFR around 10 % falls in the "Medium" band.
   - MTTR median around 2 h falls in the "High" band.
   The sanity check is not that the fixture *matches* a particular band but that the band-drawing function places the number sensibly in reference to published DORA bands (see [dora.dev](https://dora.dev/)).

5. **Integration test with release ledger.** Run a staged canary (exercise 01) in a staging cluster that aborts; verify an entry lands in the ledger with `status = aborted`, and that the next daily DORA job counts it correctly.

6. **Eval-vs-SLO split.** Inject a fixture with cause_class `eval` only and verify the Time to Restore panel reports an `eval_median_seconds` but leaves `slo_median_seconds` null for that window.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Source of truth *(hard gate)* | Git log | Git log + rollback records | Release ledger produced by exercises 01 / 03 / 04 |
| 2 | Deployment definition *(hard gate)* | Container image push | Image push + rollback | Image push + prompt-store commit + provider-version bump + flag-flip |
| 3 | Lead Time anchor | Only `git commit` | Multiple anchors but no `kind` field | `change_anchor.kind` tagging + per-kind breakdown |
| 4 | Change Failure Rate definition *(hard gate)* | Only SLO | SLO + rollback | SLO + eval regression + rollback + linked incident |
| 5 | MTTR split | Combined only | Named in prose | Combined + SLO + eval reported separately |
| 6 | Window selection | 7d | 28d only | 28d default + configurable window |
| 7 | Dashboard | Single number per key | Four panels | Four panels + per-dimension breakdown + reference bands |
| 8 | Review cadence | Missing | Monthly or less | Weekly + quarterly + annual target-band review |
| 9 | Ingestion tests | None | Happy path | Happy path + rollback + eval-only failure |
| 10 | Published-band alignment | Not referenced | Referenced in prose | Referenced in prose and drawn on panels |

## 5. Common mistakes

- **Counting only git deploys.** Hosted-model products get undercounted and look "slow" on Deployment Frequency even when they ship user-visible changes daily via prompt-store commits.
- **Treating eval regressions as "not an incident".** Change Failure Rate without eval-caused failures reads like a fast team shipping without defects, when in fact the eval gate is being skipped or the eval pipeline is silent.
- **Combined MTTR only.** A single MTTR number hides the fact that eval-caused restores take longer. The transparency of the split *is* the diagnostic.
- **Chasing the elite band.** The DORA research is diagnostic, not prescriptive. A team shifting from Low to Medium is improving; a Low team targeting Elite by next quarter is setting itself up to game the metric.
- **Gaming via `flag-flip` noise.** Counting a thousand micro-flag-flips as deployments inflates the number without reflecting real throughput. The recipe is to tag the `strategy` and look at the breakdown; a sudden rise in `flag-flip` alongside flat `canary` is a sign of metric gaming.
- **No tie to the release ledger.** A dashboard fed by hand-curated spreadsheets loses fidelity within a quarter.
- **Long window hides regressions.** A 90-day window is smooth but hides the week that caused the quarter's regression. 28 days is the useful default; the one-week spotlight is what the weekly review covers.

## 6. References

- Forsgren, N., Humble, J., Kim, G., *Accelerate: The Science of Lean Software and DevOps* (IT Revolution Press). The original formulation of the Four Keys.
- DORA, *State of DevOps* report — [dora.dev](https://dora.dev/) — the annual report that publishes the performance bands and the standard metric definitions.
- Google Cloud / DORA, *Four Keys* reference implementation — [github.com/dora-team/fourkeys](https://github.com/dora-team/fourkeys) — the ingestion / dashboard shape this solution mirrors (with the model-release adaptations above).
- Google Cloud Architecture Center, *DevOps measurement: Key metrics* — [cloud.google.com/architecture/devops/devops-measurement-key-metrics](https://cloud.google.com/architecture/devops/devops-measurement-key-metrics) — the definitions used in §1.2.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — the vocabulary for `cause_class = eval` failure modes.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `MEASURE` function names release metrics as a required control; adapted DORA is one implementation.
