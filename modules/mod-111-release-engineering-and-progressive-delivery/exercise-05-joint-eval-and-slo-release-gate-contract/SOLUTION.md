# Exercise 05 — Joint Eval and SLO Release Gate Contract — Reference Solution

Paired learning file: `lessons/mod-111-release-engineering-and-progressive-delivery/exercises/exercise-05-joint-eval-and-slo-release-gate-contract.md`.

> **Scope note.** The paired exercise asks the learner to author the *contract* between the eval team (which owns quality signals) and the SRE team (which owns the SLO stack), used by progressive-delivery tools (exercises 01, 03, 04) as the release gate. The output is a design artefact: a written contract, a machine-readable schema, and a rubric against which both teams can grade a candidate contract.

## 1. Solution overview

Progressive-delivery pipelines for model-backed services need a *single, callable* release gate. Historically two stacks produce the signals that gate deserves to consume:

- **Eval stack** — groundedness, answer correctness, refusal rate, structured-output validity, safety classifier scores. Owned by the eval / applied-AI team. Signal is sampled, scheduled, and often human-labelled.
- **SLO stack** — error-rate SLIs, latency SLIs, saturation SLIs, provider error rate, cost rate. Owned by SRE. Signal is continuous, high-cardinality, machine-produced.

Without a contract, three failure modes dominate:

1. **Gate ambiguity.** "Eval passed" and "SLO is healthy" mean different things on different days to different humans. A release that is green on one but not the other gets argued, shipped, and later rolled back.
2. **Gate staleness.** A gate that passes because the eval signal is 36 hours old is worse than a gate that fails. Progressive-delivery automations (exercises 01, 03) must distinguish "passed on fresh signal" from "passed on no signal".
3. **Gate asymmetry.** One side's veto blocks the release; the other side's green is advisory. The result is a culture where the stronger team's signal dominates even when it is the wrong signal for the change.

The reference solution is a three-part contract:

1. **A human-readable contract document** (`contracts/release-gate.md`) that says who owns which signal, what thresholds mean, and how veto authority works.
2. **A machine-readable schema** (`contracts/release-gate.schema.json`) that every progressive-delivery pipeline posts against. The schema encodes the per-signal, per-route thresholds and the freshness bound per signal.
3. **A verdict service** (`services/release-gate/`) that evaluates the schema against current signal values and returns a single `pass | fail | needs-review` verdict with full reasoning.

### 1.1 Design principles

- **Symmetric veto.** Either team can veto a release on its own signal class. A veto is a per-signal "fail", not a global vote.
- **Freshness is a first-class predicate.** Every signal has a `max_staleness`; a signal older than that is treated as a `fail`, not a pass.
- **Thresholds are per-route and per-signal, not per-release.** Routes with regulated users have tighter bounds; the contract encodes that once and does not relitigate per release.
- **Route-class defaults are opinionated.** New routes inherit from a route class (`faq`, `legal-adj`, `api-call`, `safety-adj`); explicit overrides are logged and reviewed quarterly.
- **No "needs-review" bucket for fully-automated pipelines.** The verdict either greenlights the pipeline or does not. "Needs-review" exists as an outcome only when a human is watching (manual release command), and is treated as `fail` by any unattended pipeline.
- **A failed gate is a data event, not a human punishment.** Every `fail` verdict is logged with its predicate so the next contract revision has evidence.

## 2. Implementation — worked answer

### 2.1 Human-readable contract — `contracts/release-gate.md`

```markdown
# Release gate contract — chatcat family

**Owner split.**
- *Eval signals* are owned by `ai-eval-engineer`. The owner is responsible for the
  freshness, calibration, and documentation of every signal in the "eval" class.
- *SLO signals* are owned by `ai-infra-sre`. The owner is responsible for the
  freshness, instrumentation, and documentation of every signal in the "slo" class.

**Veto is symmetric.** Either owner's signal class can veto a release. A release ships
iff every signal returns a `pass` or a `not_applicable`.

**Freshness.** Every signal declares a `max_staleness`. The gate treats a signal older
than that as `fail`, regardless of its value. Staleness is measured end-to-end: from
the observation producing the value to the gate's query time.

**Thresholds** are declared per `route_class` with explicit per-route overrides.
Route classes and their defaults live in `release-gate.schema.json`. A route that does
not appear in the schema inherits the `default` class; releases on undeclared routes
require a `needs-review` verdict (and therefore a human approver).

**Veto appeal.** A release that fails only one signal class may be force-shipped by
the signal owner (not by anyone else), with a written reason that is appended to
the release ledger. This is the escape hatch for a signal that is itself broken; it
is NOT the escape hatch for "we disagree with the signal."

**Revision cadence.** The contract is reviewed quarterly. Thresholds move only when
backed by at least one of: a documented incident, a documented false-fail with
analysis, or an explicit product decision (e.g. a tighter SLO on a new regulated
route). Threshold drift without justification is reverted.

**Dispute resolution.** If the eval owner and SRE owner disagree on whether a signal
is correctly configured, the dispute escalates to `ai-sre-function-lead` for a written
decision. The dispute stays open; the release does not ship on a tied signal class.
```

### 2.2 Machine-readable schema — `contracts/release-gate.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "chatcat.release-gate.v1",
  "type": "object",
  "required": ["route_classes", "routes", "signals"],
  "properties": {
    "route_classes": {
      "type": "object",
      "patternProperties": {
        "^[a-z0-9-]+$": {
          "type": "object",
          "required": ["description", "signal_bounds"],
          "properties": {
            "description": { "type": "string" },
            "signal_bounds": {
              "type": "object",
              "additionalProperties": {
                "type": "object",
                "required": ["predicate", "threshold", "owner"],
                "properties": {
                  "predicate": { "enum": ["<=", ">=", "within_abs", "within_rel", "equals"] },
                  "threshold": {},
                  "owner": { "enum": ["eval", "slo"] }
                }
              }
            }
          }
        }
      }
    },
    "routes": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["class"],
        "properties": {
          "class": { "type": "string" },
          "overrides": { "type": "object" }
        }
      }
    },
    "signals": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["owner", "source", "unit", "max_staleness_seconds"],
        "properties": {
          "owner": { "enum": ["eval", "slo"] },
          "source": { "type": "string" },
          "unit": { "type": "string" },
          "max_staleness_seconds": { "type": "integer", "minimum": 60 },
          "description": { "type": "string" }
        }
      }
    }
  }
}
```

### 2.3 A concrete schema instance — `contracts/release-gate.chatcat.json`

```json
{
  "route_classes": {
    "default": {
      "description": "FAQ-level freeform; no PII; no auto-publish.",
      "signal_bounds": {
        "error_rate_5m":         { "predicate": "<=", "threshold": 0.01, "owner": "slo" },
        "ttft_p95_5m":           { "predicate": "<=", "threshold": 2.0,  "owner": "slo" },
        "provider_error_rate":   { "predicate": "<=", "threshold": 0.02, "owner": "slo" },
        "cost_per_request_ratio":{ "predicate": "<=", "threshold": 1.15, "owner": "slo" },
        "groundedness_delta":    { "predicate": ">=", "threshold": -0.02,"owner": "eval" },
        "refusal_delta":         { "predicate": "<=", "threshold": 0.05, "owner": "eval" },
        "structured_valid_rate": { "predicate": ">=", "threshold": 0.98, "owner": "eval" },
        "policy_safety_regressions": { "predicate": "equals", "threshold": 0, "owner": "eval" }
      }
    },
    "legal-adj": {
      "description": "Routes with legal-adjacent content; tighter quality bounds.",
      "signal_bounds": {
        "error_rate_5m":         { "predicate": "<=", "threshold": 0.005,"owner": "slo" },
        "ttft_p95_5m":           { "predicate": "<=", "threshold": 2.5,  "owner": "slo" },
        "groundedness_delta":    { "predicate": ">=", "threshold": -0.01,"owner": "eval" },
        "refusal_delta":         { "predicate": "<=", "threshold": 0.02, "owner": "eval" },
        "structured_valid_rate": { "predicate": ">=", "threshold": 0.99, "owner": "eval" },
        "policy_safety_regressions": { "predicate": "equals", "threshold": 0, "owner": "eval" }
      }
    },
    "api-call": {
      "description": "Structured output; exact-match required.",
      "signal_bounds": {
        "error_rate_5m":         { "predicate": "<=", "threshold": 0.005,"owner": "slo" },
        "ttft_p95_5m":           { "predicate": "<=", "threshold": 1.5,  "owner": "slo" },
        "structured_exact_match_rate": { "predicate": ">=", "threshold": 0.995, "owner": "eval" },
        "policy_safety_regressions": { "predicate": "equals", "threshold": 0, "owner": "eval" }
      }
    }
  },
  "routes": {
    "faq":         { "class": "default" },
    "support":     { "class": "default" },
    "legal-help":  { "class": "legal-adj" },
    "tool-call":   { "class": "api-call" },
    "memory-upsert": { "class": "api-call", "overrides": { "error_rate_5m": { "predicate": "<=", "threshold": 0.001, "owner": "slo" } } }
  },
  "signals": {
    "error_rate_5m":         { "owner": "slo",  "source": "prometheus:chatcat_error_rate_5m",             "unit": "ratio",     "max_staleness_seconds": 180,   "description": "5-minute rate of 5xx + 429 responses as fraction of total." },
    "ttft_p95_5m":           { "owner": "slo",  "source": "prometheus:chatcat_ttft_seconds_p95_5m",       "unit": "seconds",   "max_staleness_seconds": 180,   "description": "p95 of time-to-first-token over 5m window." },
    "provider_error_rate":   { "owner": "slo",  "source": "prometheus:chatcat_provider_error_rate",      "unit": "ratio",     "max_staleness_seconds": 180 },
    "cost_per_request_ratio":{ "owner": "slo",  "source": "prometheus:chatcat_cost_per_request_ratio",   "unit": "ratio",     "max_staleness_seconds": 300,   "description": "Canary tokens-per-request / baseline tokens-per-request." },
    "groundedness_delta":    { "owner": "eval", "source": "eval-pipeline:groundedness_delta",            "unit": "delta",     "max_staleness_seconds": 3600,  "description": "Hourly sampled groundedness: canary - baseline." },
    "refusal_delta":         { "owner": "eval", "source": "eval-pipeline:refusal_delta",                 "unit": "delta",     "max_staleness_seconds": 3600 },
    "structured_valid_rate": { "owner": "eval", "source": "eval-pipeline:structured_valid_rate",         "unit": "ratio",     "max_staleness_seconds": 3600 },
    "structured_exact_match_rate": { "owner": "eval", "source": "eval-pipeline:structured_exact_match",  "unit": "ratio",     "max_staleness_seconds": 3600 },
    "policy_safety_regressions": { "owner": "eval", "source": "shadow-pipeline:safety_regressions",      "unit": "count",     "max_staleness_seconds": 7200,  "description": "New-allow verdicts on safety-adjacent prompts in the last 24h of shadow traffic." }
  }
}
```

### 2.4 Verdict service — `services/release-gate/gate.py`

```python
from __future__ import annotations
import json
import time
from dataclasses import dataclass
from typing import Protocol


class SignalSource(Protocol):
    def read(self, source: str) -> tuple[float, float] | None:
        """Return (value, observed_at_unix) or None if missing."""


@dataclass
class SignalVerdict:
    name: str
    status: str   # "pass" | "fail" | "not_applicable"
    reason: str
    owner: str


@dataclass
class Verdict:
    passed: bool
    signals: list[SignalVerdict]

    def to_json(self) -> dict:
        return {
            "pass": self.passed,
            "signals": [s.__dict__ for s in self.signals],
            "failed_owners": sorted({s.owner for s in self.signals if s.status == "fail"}),
        }


_PREDICATES = {
    "<=":  lambda value, thr: value <= thr,
    ">=":  lambda value, thr: value >= thr,
    "equals": lambda value, thr: value == thr,
    "within_abs": lambda value, thr: abs(value) <= thr,
}


def _bounds_for_route(contract: dict, route: str) -> dict:
    route_entry = contract["routes"].get(route)
    if route_entry is None:
        cls = "default"
        overrides = {}
    else:
        cls = route_entry["class"]
        overrides = route_entry.get("overrides", {})
    bounds = dict(contract["route_classes"][cls]["signal_bounds"])
    bounds.update(overrides)
    return bounds


def evaluate(contract: dict, route: str, source: SignalSource, now: float | None = None) -> Verdict:
    now = now if now is not None else time.time()
    bounds = _bounds_for_route(contract, route)
    signals = contract["signals"]

    results: list[SignalVerdict] = []
    for name, rule in bounds.items():
        sig_meta = signals.get(name)
        if sig_meta is None:
            results.append(SignalVerdict(name, "fail", "signal declared in bounds but missing from signals registry", rule["owner"]))
            continue
        reading = source.read(sig_meta["source"])
        if reading is None:
            results.append(SignalVerdict(name, "fail", f"no observation available for {sig_meta['source']}", rule["owner"]))
            continue
        value, observed_at = reading
        age = now - observed_at
        if age > sig_meta["max_staleness_seconds"]:
            results.append(SignalVerdict(name, "fail", f"stale: age={age:.0f}s > max={sig_meta['max_staleness_seconds']}s", rule["owner"]))
            continue
        predicate = _PREDICATES[rule["predicate"]]
        if predicate(value, rule["threshold"]):
            results.append(SignalVerdict(name, "pass", f"value={value} {rule['predicate']} {rule['threshold']}", rule["owner"]))
        else:
            results.append(SignalVerdict(name, "fail", f"value={value} violates {rule['predicate']} {rule['threshold']}", rule["owner"]))

    passed = all(r.status != "fail" for r in results)
    return Verdict(passed, results)
```

### 2.5 HTTP shape

```
POST /v1/release-gate
{
  "contract_ref": "release-gate.chatcat.json@v15",
  "route": "legal-help",
  "candidate_build_id": "chatcat-serving@sha256:abc...",
  "requested_at": "2026-11-13T12:00:00Z"
}

200 OK
{
  "pass": false,
  "signals": [
    { "name": "groundedness_delta", "status": "fail", "reason": "value=-0.031 violates >= -0.01", "owner": "eval" },
    { "name": "error_rate_5m", "status": "pass", "reason": "value=0.003 <= 0.005", "owner": "slo" },
    ...
  ],
  "failed_owners": ["eval"]
}
```

The pipelines in exercises 01 / 03 / 04 call this endpoint. On `pass: false`, they stop the rollout and emit the full `signals` array to the release ledger.

### 2.6 Logging and feedback

Every verdict is written to the release ledger:

```json
{
  "contract_ref": "release-gate.chatcat.json@v15",
  "candidate_build_id": "chatcat-serving@sha256:abc...",
  "route": "legal-help",
  "verdict": { "pass": false, "failed_owners": ["eval"], "signals": [ ... ] },
  "requested_at": "...",
  "decided_at": "...",
  "appeal_ref": null
}
```

Quarterly, the ledger is read to answer: which signals fire most often, which signals have never fired (dead gate), which signals were followed by an appeal (contract mismatch).

## 3. Validation steps

1. **Schema validation.** `jq -e 'empty' contracts/release-gate.chatcat.json` must succeed. Validate against `release-gate.schema.json` using any JSON-Schema validator (e.g. `python -m jsonschema -i release-gate.chatcat.json release-gate.schema.json`).

2. **Unit tests for the verdict service.**
   - Every signal reading present and in-bounds → `pass`.
   - One SLO signal out of bounds → `fail`, `failed_owners == ["slo"]`.
   - Eval signal older than `max_staleness_seconds` → `fail`, `failed_owners == ["eval"]`.
   - Route not in `routes` registry → uses `default` class and still returns a verdict.
   - Route with `overrides` applies the override, not the class default.
   - Missing signal reading → `fail` with explicit reason.
   - Safety regressions > 0 on a safety-adjacent route → `fail`.

3. **Integration test with the exercise-01 pipeline.** Run Argo Rollouts / Flagger against a staging deployment. Force the eval pipeline to emit `groundedness_delta = -0.05` and confirm the rollout aborts at the 5 % step with the contract failure recorded.

4. **Freshness test.** Stop the eval pipeline. Attempt a release 2 h later. Verdict must fail with `stale: age=…`.

5. **Appeal path.** Submit an appeal with a written reason. Confirm the ledger records the appeal and that the release proceeds. Confirm that an unauthorised user cannot submit an appeal (only the signal's owner role may).

6. **Threshold-change audit.** Modify the `groundedness_delta` threshold in the schema. Confirm the schema diff lands on a PR, that the review is cross-team (eval + SRE), and that the merged schema becomes a new `contract_ref` version.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Owner split *(hard gate)* | Not stated | Named but not tied to signal classes | Each signal has a named owner and veto authority |
| 2 | Freshness predicate *(hard gate)* | Missing | Mentioned in prose only | `max_staleness_seconds` on every signal, enforced by the service |
| 3 | Per-route thresholds *(hard gate)* | Single global threshold | Per-route override but no route classes | Route classes + per-route overrides + default fallback |
| 4 | Symmetric veto | SLO vetoes only / Eval vetoes only | Symmetric in prose | Symmetric in schema and in the verdict service |
| 5 | Machine-readable schema | None | YAML prose | JSON Schema with validator and example instance |
| 6 | Verdict service | Shell script | Python function | Deployable service with HTTP shape + logging |
| 7 | Appeal path | Missing | Prose-only | Named in contract + logged in ledger + authz check |
| 8 | Ledger records | None | Record on failure only | Record on every verdict with full signal breakdown |
| 9 | Revision cadence *(hard gate)* | Not stated | Named cadence | Named cadence + source-of-evidence rule |
| 10 | Tests | Happy-path only | Covers one failure class | Covers pass, fail, stale, missing, appeal |

## 5. Common mistakes

- **Asymmetric veto.** A contract where "eval passes" is required but "SLO is advisory" (or the reverse) is not a release gate, it is politics. Both sides veto; both sides can appeal.
- **"Fresh enough" as a prose rule.** If freshness is not in the schema and enforced by the service, gates pass on stale signal silently. The service must fail closed on staleness.
- **Thresholds in code, not in schema.** Thresholds living in the Prometheus query or in the pipeline manifest are impossible to review together. A single schema is the point.
- **No `not_applicable` handling.** Some signals don't make sense on some routes (e.g. `structured_exact_match_rate` on a free-form route). The schema allows a route class to omit a signal; the verdict service must treat omission as `not_applicable`, not as `fail`.
- **Thresholds set by the team that owns the pipeline.** If SRE sets the eval thresholds, the eval team cannot defend the floor during the next incident. The *owner* field in the schema binds threshold authorship to the signal owner.
- **"Needs review" as the default fallback.** An unattended pipeline (canary, shadow promotion, blue/green flip) cannot block on `needs-review`. Either the contract knows how to grade the signal or the release is blocked and a human is paged. "Needs review" that an automation treats as "proceed" is the worst possible default.
- **Contract never revised.** A contract whose thresholds have not moved in a year is either perfect or ignored; it is almost always ignored. The quarterly cadence exists because the signals and products both drift.

## 6. References

- Google SRE workbook, *Alerting on SLOs* — [sre.google/workbook/alerting-on-slos/](https://sre.google/workbook/alerting-on-slos/) — the SRE-side signal discipline this contract depends on.
- OpenAPI schema grammar / JSON Schema draft 2020-12 — [json-schema.org/draft/2020-12](https://json-schema.org/draft/2020-12) — the schema-as-contract shape.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML09: Output Integrity Attack` and `ML06: ML Supply Chain` justify a mandatory eval-side veto, not merely an advisory signal.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "ML Model Evasion" technique class motivates the policy-safety-regressions signal as a hard-equals-0 bound.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `MANAGE` function names "pre-release decision authority" as a required control; this contract names it explicitly.
