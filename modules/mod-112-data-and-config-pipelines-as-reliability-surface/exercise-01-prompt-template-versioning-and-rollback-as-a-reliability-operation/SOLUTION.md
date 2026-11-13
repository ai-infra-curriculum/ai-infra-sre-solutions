# Exercise 01 — Prompt Template Versioning and Rollback as a Reliability Operation — Reference Solution

Paired learning file: `lessons/mod-112-data-and-config-pipelines-as-reliability-surface/exercises/exercise-01-prompt-template-versioning-and-rollback-as-a-reliability-operation.md`.

> **Scope note.** The paired exercise asks the learner to treat a prompt template as a first-class deployable artifact — versioned, promoted through environments, gated by an eval, and rollback-able as a one-step SRE operation. The reference solution below produces (a) the storage and metadata shape for prompt versions, (b) the promotion pipeline, (c) the rollback runbook and automated primitive, and (d) a wiring diagram with the serving layer so the active version is a resolvable reference, not an in-code string. The illustrative product is the fabricated `chatcat` RAG-chat surface used across this module family.

## 1. Solution overview

A production prompt-template change is a production configuration change. If the serving code is treated as reliability-critical but the prompt is a string in a commit three repos away, the organisation has two change paths with two different control sets — and the one with no control set is the one that burns users. The reference design treats every prompt template as a versioned artifact with:

1. **An immutable version id** derived from the template body plus its metadata (model, tool definitions, output schema). Changing any field produces a new id.
2. **A resolvable alias** per route and environment — `chatcat/faq@prod` resolves to a single version id at a point in time. The serving process reads the alias, not the body.
3. **A promotion ledger entry per transition**, written by CI, carrying the eval-gate verdict that justified the promotion.
4. **A one-command rollback** that flips the alias back to the previously-active version and writes a ledger entry with `reason=rollback`.

This is directly the control shape the NIST AI RMF names in `MANAGE 4.1` ("Post-deployment AI system monitoring plans are implemented, including mechanisms for capturing and evaluating input from users and other relevant AI actors, appeal and override, decommissioning, incident response, recovery, and change management") and `GOVERN 1.5` ("Ongoing monitoring and periodic review of the risk management process and its outcomes are planned, and organizational roles and responsibilities clearly defined, including determining the frequency of periodic review"). Rollback is a change-management primitive, so it must be as cheap and as audited as the forward deploy.

### 1.1 Decision rationale — why not "just a git revert"

A git revert of a prompt template works only if three preconditions hold, and in practice they rarely all do:

- The prompt is actually stored as a file in a repo the serving process rebuilds from (often it is in a prompt store, a database row, or a config map).
- The serving process picks up the revert in under the time a regression is tolerable (often CI + image build + rollout is 20+ minutes — long enough for a prompt regression to generate a real incident).
- The reverted body is behaviourally identical to the known-good version (true for a one-line revert, false the moment anything downstream — a tool list, an output schema — changed alongside the prompt).

A version-alias design removes all three preconditions: rollback is a metadata flip, propagates in seconds, and resolves to a version id whose eval verdict is on record.

### 1.2 What counts as "the prompt"

The version id must cover everything that changes model behaviour, not only the user-visible text. The reference content-hash inputs are:

- template body (system + user templates, verbatim),
- tool definitions and their JSON schemas,
- output schema (for structured-output routes),
- model id and provider (`gpt-4o-mini-2024-07-18`, `claude-sonnet-4-6`, etc.),
- decoding parameters that materially affect output (temperature, top_p, max_tokens, response_format),
- retrieval configuration ref for RAG routes (which index, which k, which rerank — recorded by reference, not inlined).

A change in any of these yields a new version id, which forces a fresh eval gate. This is the control named in OWASP ML Top 10 `ML06 — AI supply chain attacks`: every element that participates in producing the model's behaviour is tracked and verified before promotion.

## 2. Implementation — worked answer

### 2.1 Prompt-store schema

`prompt-store/schema.sql` (illustrative PostgreSQL):

```sql
CREATE TABLE prompt_version (
  version_id      TEXT PRIMARY KEY,                -- sha256 of the canonical JSON below
  route           TEXT NOT NULL,                   -- e.g. 'chatcat/faq'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by      TEXT NOT NULL,                   -- git author or service principal
  body            JSONB NOT NULL,                  -- {system, user, tools, output_schema, model, params, retrieval_ref}
  parent_version  TEXT REFERENCES prompt_version(version_id),
  change_note     TEXT NOT NULL                    -- required; empty string rejected
);

CREATE TABLE prompt_alias (
  route        TEXT NOT NULL,
  environment  TEXT NOT NULL,                      -- 'dev' | 'staging' | 'prod'
  version_id   TEXT NOT NULL REFERENCES prompt_version(version_id),
  set_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  set_by       TEXT NOT NULL,
  reason       TEXT NOT NULL,                      -- 'promote' | 'rollback' | 'bootstrap'
  verdict_ref  TEXT,                               -- release-gate verdict id; required for 'promote'
  PRIMARY KEY (route, environment, set_at)
);

CREATE VIEW prompt_alias_current AS
  SELECT DISTINCT ON (route, environment)
         route, environment, version_id, set_at, set_by, reason, verdict_ref
    FROM prompt_alias
    ORDER BY route, environment, set_at DESC;
```

Both tables are append-only. The `prompt_alias` row history is the record the `GOVERN 1.5` audit uses; every production behaviour change is a row.

### 2.2 Version id computation

`promptstore/version.py`:

```python
from __future__ import annotations

import hashlib
import json
from typing import Any, Mapping


CANONICAL_FIELDS = (
    "system",
    "user",
    "tools",
    "output_schema",
    "model",
    "params",
    "retrieval_ref",
)


def compute_version_id(body: Mapping[str, Any]) -> str:
    """Return the deterministic version id for a prompt body.

    The id is a sha256 over a canonical JSON serialisation of exactly the
    fields in CANONICAL_FIELDS. Any field drift (new fields, missing fields)
    raises -- we fail closed on unknown schema.
    """
    missing = [f for f in CANONICAL_FIELDS if f not in body]
    extra = [k for k in body.keys() if k not in CANONICAL_FIELDS]
    if missing or extra:
        raise ValueError(
            f"prompt body shape invalid: missing={missing!r} extra={extra!r}"
        )
    canonical = {f: body[f] for f in CANONICAL_FIELDS}
    serialised = json.dumps(
        canonical, sort_keys=True, separators=(",", ":"), ensure_ascii=False
    ).encode("utf-8")
    return "pv_" + hashlib.sha256(serialised).hexdigest()[:40]
```

The ingest path refuses an upload whose declared `version_id` does not match the recomputed one. This is a cheap tripwire against accidental "edit in place" workflows where a human updates the body but forgets to bump the id.

### 2.3 Serving-side resolution

The serving process never embeds a prompt body. It holds a route + environment pair and resolves to a version id, then to a body, with a short cache:

```python
# chatcat/serving/prompt_client.py
from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Any, Mapping


@dataclass(frozen=True)
class ResolvedPrompt:
    version_id: str
    body: Mapping[str, Any]
    resolved_at: float


class PromptClient:
    """Resolves an alias to a prompt body with a short TTL.

    The cache TTL is deliberately short (default 30s) so a rollback propagates
    in under a minute without a restart. We cache the resolved body, not just
    the version id, so a store outage does not immediately stop serving.
    """

    def __init__(self, store, ttl_seconds: float = 30.0) -> None:
        self._store = store
        self._ttl = ttl_seconds
        self._cache: dict[tuple[str, str], ResolvedPrompt] = {}

    def resolve(self, route: str, environment: str) -> ResolvedPrompt:
        key = (route, environment)
        cached = self._cache.get(key)
        now = time.monotonic()
        if cached and (now - cached.resolved_at) < self._ttl:
            return cached
        version_id = self._store.current_version_id(route, environment)
        body = self._store.get_body(version_id)
        resolved = ResolvedPrompt(version_id=version_id, body=body, resolved_at=now)
        self._cache[key] = resolved
        return resolved
```

Every served request emits an OpenTelemetry attribute `genai.prompt.version_id` so the request-level logs and traces carry the exact version used. That attribute is the join key between serving metrics and the release ledger.

### 2.4 Promotion pipeline

`.github/workflows/promote-prompt.yml`:

```yaml
name: promote-prompt
on:
  workflow_dispatch:
    inputs:
      route: { required: true, type: string }
      target_env: { required: true, type: choice, options: [staging, prod] }
      version_id: { required: true, type: string }

permissions:
  contents: read
  id-token: write

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run release-gate eval
        id: eval
        run: |
          scripts/run-eval.sh \
            --route "${{ inputs.route }}" \
            --version "${{ inputs.version_id }}" \
            --target "${{ inputs.target_env }}" \
            --out eval-verdict.json
      - name: Enforce verdict
        run: |
          jq -e '.verdict == "pass"' eval-verdict.json > /dev/null
      - name: Set alias
        run: |
          scripts/promptctl set-alias \
            --route "${{ inputs.route }}" \
            --env "${{ inputs.target_env }}" \
            --version "${{ inputs.version_id }}" \
            --reason promote \
            --verdict-ref "$(jq -r '.verdict_id' eval-verdict.json)"
      - name: Record in release ledger
        run: |
          scripts/ledger append \
            --service chatcat \
            --route "${{ inputs.route }}" \
            --strategy prompt-promote \
            --change-anchor-kind prompt-store \
            --change-anchor-ref "${{ inputs.version_id }}" \
            --status shipped \
            --gate-verdict-ref "$(jq -r '.verdict_id' eval-verdict.json)"
```

The gate script runs the evaluation suite named in the route's gate contract (defined in `mod-111 / exercise-05`'s *joint eval and SLO release gate contract*). A missing or stale verdict file is a hard fail; there is no override flag — rollback is the escape hatch, not override.

### 2.5 Rollback primitive

`promptctl rollback` flips the alias back to the previous version and writes a ledger entry with `reason=rollback`. It is a single-command operation so the runbook in `mod-108` can call it from an incident Slack channel.

```python
# promptctl/rollback.py
from __future__ import annotations

import argparse
import sys


def rollback(store, ledger, route: str, environment: str, operator: str, note: str) -> int:
    history = store.alias_history(route=route, environment=environment, limit=2)
    if len(history) < 2:
        print(f"no prior version to roll back to for {route}@{environment}", file=sys.stderr)
        return 2
    current, previous = history[0], history[1]
    if current.version_id == previous.version_id:
        print("already on the target version", file=sys.stderr)
        return 3
    store.set_alias(
        route=route,
        environment=environment,
        version_id=previous.version_id,
        operator=operator,
        reason="rollback",
        verdict_ref=previous.verdict_ref,  # carry the prior pass forward
    )
    ledger.append(
        service="chatcat",
        route=route,
        strategy="prompt-rollback",
        change_anchor_kind="prompt-store",
        change_anchor_ref=previous.version_id,
        status="shipped",
        note=note,
    )
    print(f"rolled back {route}@{environment} to {previous.version_id}")
    return 0


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--route", required=True)
    parser.add_argument("--env", dest="environment", required=True)
    parser.add_argument("--operator", required=True)
    parser.add_argument("--note", required=True, help="why we are rolling back (incident id, etc.)")
    args = parser.parse_args(argv)
    from promptstore.store import open_store
    from ledger.client import open_ledger

    with open_store() as store, open_ledger() as ledger:
        return rollback(
            store=store,
            ledger=ledger,
            route=args.route,
            environment=args.environment,
            operator=args.operator,
            note=args.note,
        )


if __name__ == "__main__":
    raise SystemExit(main())
```

Two design points worth naming:

- The rollback **reuses the previous verdict ref**. That version shipped once already; the eval that justified it is still the record. Re-running the gate during an incident just slows the mitigation.
- The rollback **requires a non-empty note**. The note is the incident id in 95 % of cases; carrying it on the ledger row is how the DORA `Time to Restore Service` metric from `mod-111 / exercise-06` closes its clock.

### 2.6 Runbook extract — `rb-prompt-rollback`

```markdown
# rb-prompt-rollback

## When to invoke
- A production route shows a measurable quality regression tied to a prompt change in the last 2 h.
- Signal sources that qualify: release-gate verdict flip, groundedness alert, structured-output-validity alert, a user-report escalation referencing output content.

## Preconditions
- You know the route (e.g. `chatcat/faq`).
- You have the incident id.

## Steps
1. Confirm the current and previous version on `promptctl status --route <route> --env prod`.
2. Execute: `promptctl rollback --route <route> --env prod --operator $USER --note <incident-id>`.
3. Expect propagation in under 60 s (serving TTL is 30 s; two refreshes).
4. Watch `chatcat_groundedness_score` and the gate-verdict dashboard for one full eval window.
5. If the regression persists, the cause is not the prompt — escalate to the retrieval-freshness runbook (exercise 02) and the feature-drift runbook (exercise 03).

## Post-incident
- Open a follow-up to fix the bad version (do NOT delete it — the ledger references it).
- Attach the ledger entry id to the incident record.
```

## 3. Validation steps

1. **Version-id determinism.** `python -m pytest promptstore/tests/test_version.py -q` must pass. The test suite asserts:
   - identical bodies yield identical ids,
   - a one-character change in `system` yields a different id,
   - a missing or extra top-level field raises `ValueError`.
2. **Alias round-trip.** Against a disposable Postgres, run `scripts/promptctl` through `set-alias → status → rollback → status` and assert the alias history has three rows with the correct `reason` fields.
3. **End-to-end promotion.** Trigger the GitHub Actions workflow with a fixture `version_id` that is known to fail the eval. Expect the job to exit non-zero at the `Enforce verdict` step and expect no `prompt_alias` row to be written.
4. **End-to-end rollback.** With a staging stack running, flip the alias forward to a deliberately-bad version. Watch the gate verdict flip to `fail`. Call `promptctl rollback` and assert the alias returns to the prior version in under 60 s (verify via the `genai.prompt.version_id` attribute on freshly-arriving spans).
5. **Audit trail.** For a random production route, `promptctl history --route <route> --env prod --days 30` must return a row per change with `set_by`, `reason`, and `verdict_ref` populated. Rows with `reason=promote` and no `verdict_ref` are a fail.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked `(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Version id covers all behaviour-affecting fields *(hard gate)* | Only the body text | Body + model id | Body + model + tools + output schema + decoding params + retrieval ref |
| 2 | Alias-based serving resolution *(hard gate)* | Serving reads a body from code or config map | Serving reads a body by id but id lives in code | Serving reads `(route, env) → version_id → body` with short TTL |
| 3 | Promotion gated by eval *(hard gate)* | No gate | Gate exists but can be overridden with a flag | Gate is required and rollback — not override — is the escape hatch |
| 4 | Rollback is a one-command primitive *(hard gate)* | Manual multi-step | One command but no ledger entry | One command, writes ledger entry, carries prior verdict ref |
| 5 | Ledger entry per change | Missing | Present but incomplete | Present with service, route, strategy, change anchor, verdict ref, status |
| 6 | Tracing / observability | No version attribute on spans | Version attribute on spans but not metrics | `genai.prompt.version_id` on spans, metrics, and logs |
| 7 | Audit queryability | No history query | History query exists but missing fields | History query returns `set_by`, `reason`, `verdict_ref` and is append-only |
| 8 | Failure modes acknowledged | No discussion | Names at least one | Names: broken eval, stale verdict, orphan version ids, cache TTL vs. incident window |
| 9 | Runbook present | Missing | Named but not stepwise | Stepwise with trigger criteria, precondition, mitigation, post-incident |
| 10 | Integration with upstream modules | Isolated | Names one | Names `mod-111 / exercise-05` gate contract and `mod-108` runbook library |

## 5. Common mistakes

- **Treating the prompt as string literal in code.** This conflates the two change paths (code vs. config) and forces every prompt change through the container-image rebuild cycle — which is slow enough that regressions outrun the rollback.
- **A content hash that excludes tool definitions or decoding params.** The template body did not change, but the behaviour did. A promotion that re-uses the previous verdict here is uncovered by eval.
- **Override flags on the promotion gate.** Every override becomes "we had to ship it hot" — eventually it is the only way anyone ships. Keep the gate strict and make rollback cheap instead.
- **Unbounded alias TTL on the serving client.** A long TTL (minutes) means rollback does not propagate during the incident window. Keep it short (≤ 60 s) even at some store-load cost.
- **Deleting bad versions.** The ledger references them; the audit trail requires them. Mark-as-withdrawn, do not delete.
- **No join key between serving metrics and prompts.** Without `genai.prompt.version_id` on spans and metrics, you cannot tell which version was serving when the regression started — rollback becomes guesswork.
- **Rollback that re-runs the gate.** Re-evaluating during an incident costs the exact minutes you were trying to save. The prior verdict is still the record.

## 6. References

- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the `GOVERN 1.5` and `MANAGE 4.1` functions name change management, monitoring, and recovery as required controls on an AI system in production. Treating the prompt as a tracked, rollback-able artifact is one implementation of those controls.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06 AI supply chain attacks` motivates hashing every behaviour-affecting input; `ML09 output integrity attack` motivates gating promotions on a structured-output validity eval.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the adversarial-techniques framework; the `AML.T0051 — LLM Prompt Injection` technique class is why prompt-template changes need the same audit controls as code changes.
- OpenTelemetry GenAI semantic conventions — introduced in the `mod-104 / exercise-01` reference solution — are the source of truth for the `genai.prompt.version_id` attribute used here.
