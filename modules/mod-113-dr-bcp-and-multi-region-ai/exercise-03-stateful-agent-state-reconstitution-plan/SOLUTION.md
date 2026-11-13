# Exercise 03 — Stateful Agent State Reconstitution Plan — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-03-stateful-agent-state-reconstitution-plan.md`.

> **Scope note.** The paired exercise prompt asks the learner to design a state-reconstitution plan for a *stateful agent* workload — a multi-turn, tool-calling agent that accumulates meaningful state across turns and (usually) across sessions. The reference solution works the plan against an illustrative agent product `scoutkit`, a fabricated travel-planning agent built on the same hosted-model stack as `chatcat` in prior exercises. The agent-specific state surfaces and failure modes are the point of the exercise; learners who chose a different illustrative agent should be scored against the rubric in §4.

## 1. Solution overview

Stateful agents have a state surface that stateless LLM calls do not:

| State surface | Example in `scoutkit` | Volatility | Who owns recovery |
|---------------|-----------------------|------------|-------------------|
| **Conversation memory** | Rolling transcript window + summary | High (per-turn) | Session store |
| **Working memory** | "Scratchpad" blob holding plan, intermediate reasoning | High | Session store |
| **Long-term memory** | User preferences, prior bookings, saved itineraries | Low (per-event) | Primary DB |
| **Tool-call state** | In-flight bookings, pending approvals, external IDs | Low but critical (correctness) | Primary DB + external systems |
| **Orchestrator state** | Node in the agent graph, retry counters, step index | High | Orchestrator store (e.g., LangGraph / durable workflow engine) |
| **Vector memory** | Embedded past turns for retrieval | Low | Vector DB |
| **Side-effect ledger** | "We charged the card", "we sent the email" | One-write-only | Side-effect ledger (append-only) |

Each of these surfaces has a distinct recovery contract. A single-RPO plan is wrong for an agent. The point of the plan is to decide, per surface, which of three reconstitution shapes applies:

1. **Resumable** — restore from a snapshot, resume forward. Example: working memory; load the scratchpad from checkpoint and continue.
2. **Replayable** — reconstruct from a replayable log of events. Example: conversation memory; replay the transcript to rebuild context.
3. **Reconciled** — compare to an external source of truth and align. Example: tool-call state; reconcile in-flight bookings against the airline's reservation system.

Choosing **resumable** where **reconciled** is required produces ghost bookings; choosing **reconciled** where **replayable** is sufficient produces user-visible latency and user re-prompting ("Can you remind me what you were helping me with?"). Choosing **replayable** where **resumable** is required produces agent loops that re-attempt actions they already completed — the worst failure mode of the three, because it drives user-visible side effects (card charges, emails) a second time.

The deliverable is a committed document `plans/scoutkit-state-reconstitution.md` plus:

- `designs/scoutkit-state-inventory.yaml` — the machine-readable inventory of state surfaces, their RPO, and their reconstitution shape.
- `scripts/reconstitute-agent-session.py` — a sample reconstitution utility that executes the plan for a single session.
- `tests/state-reconstitution-drill.md` — the drill fixture for exercising the plan.

### 1.1 Why the three-shape framing

A plain "RTO/RPO per state surface" table is necessary but not sufficient. Two surfaces with the same RTO require different *recovery procedures* depending on whether the state is resumable, replayable, or reconciled. Treating the shape explicitly is what makes the plan actionable; without it, the on-call is left to invent reconciliation at incident time.

### 1.2 The three cardinal rules of agent reconstitution

- **Rule A — No double-committed side effects.** Any action with an external observable effect (payment, external API write, email, calendar event) is gated on the side-effect ledger. Reconstitution reads the ledger before re-attempting any action with a ledger entry.
- **Rule B — Explicit resume-vs-restart choice per session.** Reconstitution does not silently "do what the agent was doing". It presents the user with a clear choice when ambiguity is high, and logs the decision.
- **Rule C — Deterministic node after reconstitution.** Reconstitution returns the agent to a *named node* in its graph (not an arbitrary resumption). The orchestrator's state identifies the node; if the node is non-idempotent, reconstitution lands on the *previous* idempotent checkpoint and re-runs forward with the side-effect gate from Rule A.

## 2. Implementation — worked answer

### 2.1 State inventory (`designs/scoutkit-state-inventory.yaml`)

```yaml
# scoutkit agent — state surface inventory
# Version: 2026-11-13
agent: scoutkit
schema_version: 1

state_surfaces:
  - id: conversation_memory
    description: Rolling transcript of the user-agent conversation (last N turns + summary).
    storage: redis cluster, cross-region replicated
    rpo_seconds: 60           # session store RPO
    rto_seconds: 120          # must be back before new turn can be served
    reconstitution_shape: replayable
    replay_source: event log (append-only Kafka topic `scoutkit.turns`)
    notes: |
      If session store is unavailable, rebuild the rolling window by replaying the
      user+assistant message pairs from the event log. The summary is re-derived by a
      summarisation call (cost ~ few tokens; cached). No external side effect; safe to
      rebuild.

  - id: working_memory
    description: Scratchpad containing the agent's current plan, hypotheses, partial
                 reasoning, and tool-result cache for this session.
    storage: redis cluster, same as conversation memory
    rpo_seconds: 60
    rto_seconds: 120
    reconstitution_shape: resumable
    checkpoint_source: orchestrator checkpoints, every node boundary
    notes: |
      Not replayable from the event log (contains derived content, not source-of-truth).
      Resume from the most recent orchestrator checkpoint. If no checkpoint exists for
      this session, rebuild lazily by re-prompting the agent with conversation memory
      only (this is the "reset to a new scratchpad" fallback).

  - id: long_term_memory
    description: User preferences, prior itineraries, saved places, loyalty programme IDs.
    storage: primary relational DB, with read replicas
    rpo_seconds: 900          # 15 min RPO; backup frequency
    rto_seconds: 1800         # 30 min RTO
    reconstitution_shape: resumable
    checkpoint_source: database backups + replica lag
    notes: |
      Standard DB recovery. Write path tolerates a stale read for the first few minutes
      after failover (user preferences do not change second-to-second).

  - id: tool_call_state
    description: External-system state the agent is managing — in-flight bookings, pending
                 approvals, external reservation IDs.
    storage: primary relational DB + external systems
    rpo_seconds: 300          # database copy
    rto_seconds: 1800
    reconstitution_shape: reconciled
    reconciliation_sources:
      - airline reservation PNR system
      - hotel property management API
      - payment processor charge ledger
    notes: |
      THE highest-risk surface. After recovery, run reconciliation against each external
      system for the reservations the agent touched in the lost window. Any discrepancy
      is logged and routed to a human for resolution. Agent must not re-attempt a
      booking without explicit reconciliation sign-off per record.

  - id: orchestrator_state
    description: Position in the agent graph: node ID, retry counters, step number, loop
                 iteration count, tool-call queue.
    storage: durable workflow engine (Temporal / LangGraph persistence)
    rpo_seconds: 5            # per-node checkpoint
    rto_seconds: 60           # engine must be back before agents can resume
    reconstitution_shape: resumable
    notes: |
      Durable-workflow engines are purpose-built for this — the engine replays the
      workflow to the last checkpoint and resumes. The point of adopting one is to
      make this surface auto-reconstituted.

  - id: vector_memory
    description: Embedded representations of prior turns and user content for retrieval.
    storage: vector DB, cross-region replicated
    rpo_seconds: 3600         # 1 hour RPO; rebuildable from source
    rto_seconds: 7200
    reconstitution_shape: resumable
    rebuild_source: source corpus (conversations + user content)
    notes: |
      If the vector DB is lost entirely (not just regionally), rebuild from source. This
      is slow (hours) but tolerable because the agent degrades gracefully to "no vector
      memory" during rebuild.

  - id: side_effect_ledger
    description: Append-only log of every external side effect the agent has committed
                 — charge IDs, booking IDs, emails sent, calendar events created.
    storage: append-only log (object storage) + cold archive
    rpo_seconds: 0            # synchronous append; must be durable before committing side effect
    rto_seconds: 0            # cannot operate without this; Rule A depends on it
    reconstitution_shape: reconciled
    reconciliation_sources:
      - external side-effect systems (same list as tool_call_state)
    notes: |
      This is THE anchor for correctness. Reconciliation confirms the ledger matches
      what external systems recorded. On conflict, the external system is authoritative
      and the ledger is corrected (and a post-mortem is written).
```

### 2.2 Reconstitution plan (`plans/scoutkit-state-reconstitution.md`)

```markdown
# scoutkit — Agent State Reconstitution Plan

**Plan ID:** `plan-reconstitution-scoutkit`.
**Version:** 2026-11-13. **Owner:** Agent Platform lead + AI SRE Director.
**Prerequisite:** Primary DR has been executed (ISCP, exercise 01). This plan is the
agent-specific step of the "Reconstitution" phase in SP 800-34's recovery process.

## 1. Pre-conditions

- Session store (Redis) has been restored or promoted (per ISCP).
- Primary DB has been restored to within RPO (per ISCP).
- Orchestrator (durable workflow engine) is online.
- Side-effect ledger is readable (this is non-negotiable; see Rule A below).

## 2. The three cardinal rules

- **Rule A.** No tool-call that produces an external side effect is executed until the
  side-effect ledger has been consulted for a prior record at that session + step +
  idempotency key. This rule is enforced in code (§2.4), not in procedure.
- **Rule B.** Any session whose state is ambiguous after reconstitution is routed into a
  "resume or restart" user dialog before the agent is allowed to continue. Ambiguity is
  defined in §3.2.
- **Rule C.** Reconstitution lands the orchestrator on an idempotent node. Non-idempotent
  nodes are never resumed; the plan rewinds to the preceding idempotent checkpoint.

## 3. Reconstitution procedure (per session)

### 3.1 Classify the session's recoverable state
For each active session in the lost window, compute a classification:

| Classification | Criterion | Reconstitution shape |
|----------------|-----------|---------------------|
| `clean` | Orchestrator checkpoint exists at an idempotent node; no in-flight side effect | Resume orchestrator |
| `pending_side_effect` | Orchestrator paused at a non-idempotent node that was in-flight | Reconcile, then resume |
| `ambiguous` | Orchestrator checkpoint missing or node-id unknown | Rule B dialog |
| `stale_long_term` | Orchestrator fine; long-term memory restored to T-900s state | Resume with "stale as of T-900s" banner |

### 3.2 Per-classification action

**`clean`:**
1. Load conversation memory (replay from `scoutkit.turns` if session-store lag > RPO).
2. Load working memory from orchestrator checkpoint.
3. Resume orchestrator at checkpointed node.
4. No user message; the agent picks up where it was.

**`pending_side_effect`:**
1. Read the side-effect ledger for the session + step.
2. If an entry exists for this step: the side effect *was* committed; resume *after* the
   step, skipping re-attempt.
3. If no entry exists: reconcile against the external system (§3.3).
4. After reconciliation, resume the orchestrator at the next idempotent node.

**`ambiguous`:**
1. Do not auto-resume. Enqueue the session for the "resume-or-restart" dialog.
2. On next user interaction, present:
   - "I was helping you with `<one-line summary from replayed transcript>`. Should I
     continue where we left off, or start fresh?"
3. On "continue": fall through to `pending_side_effect` procedure.
4. On "fresh": clear working memory + orchestrator state, preserve conversation memory
   and long-term memory. Add a transcript line "(assistant restarted at user request)".

**`stale_long_term`:**
1. Resume per `clean` with a flag on the session marking long-term memory as "possibly
   stale for the first 15 min of this session".
2. Any tool call that reads long-term memory (e.g., "look up my loyalty number") is
   routed through a re-read against the live DB and the response is marked as the fresh
   value in the ledger.

### 3.3 External-system reconciliation

For each external system the agent integrated with in the lost window:

1. Query the external system for records *tagged with this session's idempotency key*
   within the lost window.
2. For each record found:
   - If the side-effect ledger has a matching entry: no action; this is already
     reconciled (the external write + ledger write succeeded together).
   - If the side-effect ledger has no matching entry: the external write succeeded but
     our ledger lost it. Write a corrective ledger entry marked `source=external` and
     open an auto-filed audit record.
3. For each ledger entry without a matching external record:
   - The ledger thinks a side effect was committed, but the external system has no
     record. This is the dangerous case. Do not auto-retry. Enqueue for human
     reconciliation.

### 3.4 Idempotency keys

Every tool call that produces an external side effect carries an idempotency key of
the shape `sha256(session_id || step_id || plan_hash || payload_hash)`. External
systems that honour idempotency keys (most payment processors, airline NDC endpoints,
and transactional email providers) use this key. External systems that do not require
the agent to maintain an **internal** idempotency table keyed by this value and to
short-circuit retries at the agent boundary.

## 4. Reconstitution utility (`scripts/reconstitute-agent-session.py`)

See `scripts/reconstitute-agent-session.py`. The utility takes a session ID and executes
§3 for that single session. It is called from the top-level reconstitution driver (which
enumerates sessions in the lost window and dispatches in parallel, bounded).

## 5. Capacity and ordering

- Dispatch sessions in parallel with bounded concurrency (configurable, default 50).
- Within a session, steps are strictly ordered; no parallelism inside a session.
- The side-effect reconciliation against each external system is rate-limited to the
  system's declared per-caller limit.
- Expected reconstitution rate on representative traffic: 1,500 sessions/minute per
  worker; 6 workers for the modelled peak gives a 10,000-session backlog cleared in
  ~70 min.

## 6. User communication

- Users whose sessions are classified `ambiguous` receive the resume-or-restart dialog
  on their next interaction. No proactive notification.
- Users whose sessions had a `pending_side_effect` resolved by reconciliation: if the
  reconciliation produced a user-visible divergence (e.g., ledger thought a booking
  was tentative but external system says confirmed), the session opens with an
  acknowledgement transcript line and the agent surfaces the booking state explicitly.
- Users whose sessions were classified `clean`: no notification. The expected behaviour
  is that the user does not notice.

## 7. Audit trail

Every reconstitution action writes a structured record to the audit log:

```yaml
reconstitution:
  timestamp: 2026-11-13T15:04:00Z
  session_id: sess_abc123
  classification: pending_side_effect
  actions:
    - type: ledger_read
      key: sha256:...
      outcome: entry_found
    - type: resume
      node: book_hotel_confirm
```

This log is retained for the same duration as the incident audit trail and is referenced
in the incident post-mortem.

## 8. Known limitations

- **Sessions that straddle the lost window** (started before, were mid-flight during):
  reconstitution restores up to the last checkpoint; interaction from the lost window
  that wasn't checkpointed is replayed from the event log. User-visible: possible small
  repetition of a message (idempotency key protects against repeated side effects, not
  against repeated chat).
- **Non-honoured idempotency keys:** external systems that don't support idempotency
  keys still rely on the agent's internal table. If that table is lost, reconstitution
  cannot short-circuit retries safely and must escalate to §3.3 human reconciliation.
- **Orchestrator version mismatch:** if the agent graph has changed since the
  checkpoint, the node ID may no longer exist. The orchestrator must refuse to resume
  across incompatible versions and route to `ambiguous`.

## 9. Change log

| Date | Author | Change |
|------|--------|--------|
| 2026-11-13 | Agent Platform | Initial publication |
```

### 2.3 Reconstitution utility (`scripts/reconstitute-agent-session.py`)

```python
"""Reconstitute a single scoutkit agent session.

Usage:
    python reconstitute-agent-session.py --session-id sess_abc123

Invariant: Rule A — never produce a side effect without ledger pre-check.
Invariant: Rule C — resume only on an idempotent node.
"""

from __future__ import annotations

import argparse
import dataclasses
import json
import logging
import sys
from typing import Protocol

LOG = logging.getLogger("reconstitute")


@dataclasses.dataclass(frozen=True)
class Classification:
    session_id: str
    kind: str  # one of: clean, pending_side_effect, ambiguous, stale_long_term
    node_id: str | None
    checkpoint_ok: bool
    long_term_stale_window_s: int


class LedgerClient(Protocol):
    def get_entry(self, idempotency_key: str) -> dict | None: ...


class OrchestratorClient(Protocol):
    def load_checkpoint(self, session_id: str) -> dict | None: ...
    def resume(self, session_id: str, node_id: str) -> None: ...
    def is_idempotent_node(self, node_id: str) -> bool: ...


class ExternalClient(Protocol):
    def find_by_idempotency_key(self, key: str) -> dict | None: ...


def classify(session_id: str, orch: OrchestratorClient, now_s: int, lt_recovered_s: int) -> Classification:
    cp = orch.load_checkpoint(session_id)
    if cp is None:
        return Classification(session_id, "ambiguous", None, False, 0)
    node_id = cp["node_id"]
    in_flight = cp.get("in_flight_side_effect")
    if in_flight:
        return Classification(session_id, "pending_side_effect", node_id, True, 0)
    if now_s - lt_recovered_s < 900:
        return Classification(session_id, "stale_long_term", node_id, True, now_s - lt_recovered_s)
    return Classification(session_id, "clean", node_id, True, 0)


def rewind_to_idempotent(orch: OrchestratorClient, node_id: str) -> str:
    """Walk the checkpoint chain backward until we reach an idempotent node (Rule C)."""
    cur = node_id
    while cur and not orch.is_idempotent_node(cur):
        cur = orch.previous_node(cur)
    if cur is None:
        raise RuntimeError(f"No idempotent ancestor for node {node_id}")
    return cur


def reconcile_pending(session_id: str, cp: dict, ledger: LedgerClient, external_systems: dict[str, ExternalClient]) -> dict:
    """Reconcile an in-flight side effect per plan §3.3."""
    idem = cp["in_flight_side_effect"]["idempotency_key"]
    external = cp["in_flight_side_effect"]["external_system"]

    ledger_entry = ledger.get_entry(idem)
    external_record = external_systems[external].find_by_idempotency_key(idem)

    if ledger_entry and external_record:
        return {"status": "already_committed", "idempotency_key": idem}
    if ledger_entry and not external_record:
        return {"status": "ledger_only_requires_human", "idempotency_key": idem}
    if external_record and not ledger_entry:
        return {"status": "external_only_backfilled_ledger", "idempotency_key": idem,
                "external_id": external_record["id"]}
    return {"status": "not_committed_safe_to_retry", "idempotency_key": idem}


def reconstitute(session_id: str,
                 orch: OrchestratorClient,
                 ledger: LedgerClient,
                 external_systems: dict[str, ExternalClient],
                 now_s: int,
                 lt_recovered_s: int,
                 audit_sink) -> dict:
    cls = classify(session_id, orch, now_s, lt_recovered_s)
    audit = {"session_id": session_id, "classification": cls.kind, "actions": []}

    if cls.kind == "clean":
        audit["actions"].append({"type": "resume", "node": cls.node_id})
        orch.resume(session_id, cls.node_id)

    elif cls.kind == "pending_side_effect":
        cp = orch.load_checkpoint(session_id)
        reconciliation = reconcile_pending(session_id, cp, ledger, external_systems)
        audit["actions"].append({"type": "reconcile", **reconciliation})
        if reconciliation["status"] == "ledger_only_requires_human":
            audit["actions"].append({"type": "enqueue_human_reconciliation"})
            audit_sink.write(audit)
            return audit
        resume_node = rewind_to_idempotent(orch, cls.node_id)
        audit["actions"].append({"type": "resume", "node": resume_node})
        orch.resume(session_id, resume_node)

    elif cls.kind == "ambiguous":
        audit["actions"].append({"type": "enqueue_resume_or_restart_dialog"})

    elif cls.kind == "stale_long_term":
        audit["actions"].append({"type": "resume_with_lt_stale_flag",
                                 "window_s": cls.long_term_stale_window_s,
                                 "node": cls.node_id})
        orch.resume(session_id, cls.node_id)

    audit_sink.write(audit)
    return audit


def main(argv: list[str]) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--session-id", required=True)
    args = parser.parse_args(argv)
    # Dependency wiring omitted for brevity — orch, ledger, externals, audit_sink are
    # injected by the reconstitution driver.
    raise NotImplementedError("Driver wires dependencies; this module exposes reconstitute().")


if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```

### 2.4 Side-effect gate (the Rule A implementation)

The gate is a wrapper around every tool call that touches an external side effect. It is
shown here as a Python decorator for clarity:

```python
def side_effect_gated(ledger: LedgerClient):
    def decorator(fn):
        def wrapper(session_id: str, step_id: str, payload: dict, **kwargs):
            idem = compute_idempotency_key(session_id, step_id, payload)
            entry = ledger.get_entry(idem)
            if entry:
                return entry["result"]  # short-circuit; already committed
            ledger.pre_write(idem, payload)
            try:
                result = fn(session_id, step_id, payload, idempotency_key=idem, **kwargs)
            except Exception:
                ledger.mark_failed(idem)
                raise
            ledger.commit(idem, result)
            return result
        return wrapper
    return decorator
```

The gate enforces: **pre_write → external call → commit**. The ledger's `pre_write` is
durable before the external call fires; a post-event reconstitution reads `pre_write`
entries and treats them as `in_flight_side_effect` candidates for reconciliation.

### 2.5 Drill fixture (`tests/state-reconstitution-drill.md`)

```markdown
# Drill — scoutkit state reconstitution

**Cadence:** semi-annual (per `mod-113/exercise-05`).

## Scenario
A 45-min regional outage affected the primary session store. 8,400 sessions were
active during the window.

## Pre-seeded test sessions
- 50 sessions in `clean` classification
- 20 sessions in `pending_side_effect` (10 with external booking recorded, 10 without)
- 10 sessions in `ambiguous` (checkpoint missing)
- 5 sessions in `stale_long_term`

## Beats
- T+00: outage declared recovered; reconstitution plan activates.
- T+10: driver script dispatches test sessions.
- T+25: expect all `clean` resumed; `pending_side_effect` either resumed or enqueued
  for human review; `ambiguous` waiting on user dialog.
- T+35: facilitator checks:
  - Side-effect ledger shows no double commits.
  - External systems show the expected counts (10 bookings exist, not 20).
  - Audit log matches the seeded classifications.
  - No `ambiguous` session was silently resumed.

## Red-team injects
- Inject one session where the ledger says "committed" but the external system has no
  record: expect `ledger_only_requires_human` classification and human reconciliation queue.
- Inject one session where the orchestrator graph version has changed since the
  checkpoint: expect refusal to resume → `ambiguous`.
```

## 3. Validation steps

1. **Rule A unit tests.** For each side-effect-producing tool call, assert that
   (a) invocation without the gate decorator fails CI, and (b) the gate short-circuits
   when an idempotency-key match exists in the ledger.
2. **Reconstitution script dry-run.** Run `scripts/reconstitute-agent-session.py` against
   a staged test session of each classification; verify the audit record matches the
   expected shape and no external call fires when a ledger match exists.
3. **State inventory completeness.** Every state-touching call-site in the agent graph
   is annotated with the surface IDs it reads/writes; build a matrix and confirm no
   surface is referenced that is absent from `designs/scoutkit-state-inventory.yaml`.
4. **Drill the plan.** Run `tests/state-reconstitution-drill.md`. The drill must produce
   zero double-commits in the side-effect ledger. Any double-commit is a failing drill
   and a Sev-2 action item.
5. **Audit replayability.** Pick a sample of the drill's audit records and verify a
   separate auditor can step through §3 by hand and reach the same classification +
   action.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | State surfaces inventoried *(hard gate)* | Single "state" | 2-3 surfaces | ≥ 5 distinct surfaces with storage, RPO, RTO |
| 2 | Three-shape framing *(hard gate)* | Not used | Mentioned | Applied per surface with named recovery procedure |
| 3 | Side-effect ledger *(hard gate)* | Missing | Referenced | First-class, enforced by code gate (Rule A) |
| 4 | Idempotency keys *(hard gate)* | None | Mentioned | Deterministic scheme + handling for non-honouring systems |
| 5 | Resume-vs-restart dialog (Rule B) | None | Mentioned | Full flow with ambiguous classification criteria |
| 6 | Idempotent-node discipline (Rule C) | None | Mentioned | Rewind-to-idempotent implemented in the reconstitution utility |
| 7 | Reconciliation against external systems | None | Narrative | Per-system procedure with audit record |
| 8 | Classification taxonomy | None | Ad hoc | 4+ classifications with deterministic criteria |
| 9 | Audit trail of reconstitution | None | Log lines | Structured audit record retained with the incident |
| 10 | Drill fixture | None | Narrative | Pre-seeded sessions covering each classification + red-team inject |

## 5. Common mistakes

- **Treating the agent as one state blob.** The single most common failure. The
  three-shape framing is a prophylactic; without it the plan collapses into "restart
  everyone", which is wrong where side effects pending.
- **Reconciling after the retry.** Reconciliation after a retry is too late; the retry
  has already caused the duplicate. Rule A (gate before side effect) is the invariant,
  not the suggestion.
- **Silent "pending_side_effect" resumption.** Auto-resuming a session that was
  mid-side-effect without the ledger read produces the exact bug the ledger exists to
  prevent.
- **No idempotent-node discipline.** Resuming at an arbitrary node that happens to be
  "where the checkpoint says we were" re-executes the non-idempotent work. Rule C is the
  invariant that stops this.
- **No handling of orchestrator version change.** If the agent graph evolves while
  checkpoints exist, resumption across versions can land on a node that doesn't exist
  or whose semantics have changed. The orchestrator must refuse to resume across
  incompatible versions.
- **Ambiguous classification silently resumed as clean.** If classification doesn't
  default to "ambiguous" on missing data, the resumption produces user-facing confusion
  (agent appears to pick up a plan the user has forgotten).
- **Idempotency key that includes the wall-clock timestamp.** Fails the invariant: the
  same logical action gets a different key on each attempt and bypasses the ledger
  short-circuit.

## 6. References

- NIST SP 800-34 Rev. 1 §4.3 "Reconstitution" — the phase this plan operationalises for
  an agent workload.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the MEASURE and MANAGE functions name "robust and reliable AI operation" as a managed property; agent state reconstitution is a specific operationalisation. The RMF Playbook for GOVERN-6 covers third-party / external-system handling that this plan's reconciliation step maps to.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML05: model theft` + `ML09: output integrity attack` establish the vocabulary for agent output / tool-call correctness after recovery; the side-effect ledger is the integrity anchor.
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "Execution → AI tool-use" and "Impact → Erode AI model integrity" technique classes motivate Rule C (resuming on an idempotent node, not an attacker-chosen one) and Rule A (gate the side effect).
- `mod-105/exercise-...` non-deterministic failure diagnosis — the taxonomy of agent-cascade failure modes this plan's classification taxonomy reflects.
- `mod-108/exercise-05` regression-fixture back-feed — the drill fixtures surfaced here are back-fed into the regression library.
- `mod-113/exercise-01` ISCP — this plan is the agent-specific extension of the ISCP's §5 "Reconstitution" phase.
- `mod-113/exercise-02` regional-outage playbook — the playbook triggers this plan's activation.
