# Exercise 04 — Data Residency-Constrained Failover Design — Reference Solution

Paired learning file: `lessons/mod-113-dr-bcp-and-multi-region-ai/exercises/exercise-04-data-residency-constrained-failover-design.md`.

> **Scope note.** The paired exercise prompt asks the learner to design a multi-region failover for an AI workload that operates under data-residency constraints — where personal data about users in region X cannot leave region X, where logs containing user content have jurisdictional constraints, and where "just fail over to another region" is not a legal option. The reference solution works the design against the same fabricated `chatcat` product with an added residency scope: the illustrative chatcat deployment serves EU users under GDPR and under the data-localisation expectations typical for an EU deployment, plus US users under a weaker baseline. The design generalises; the rubric scores whichever jurisdictional assumption the learner adopts.

## 1. Solution overview

A residency-constrained failover is a design exercise with three cardinal constraints that make it different from the generic two-region HA in exercise 01:

1. **Personal data cannot migrate across the residency boundary.** The failover target for an EU user cannot be a US region (or a region the controller has not catalogued as permissible under the SCC / transfer mechanism in use).
2. **Within the residency boundary, failover must still work.** The customer-visible contract is not "we're available unless there is a single-region outage in your residency"; the contract is "we're available within your residency even on regional failure".
3. **The hosted-model provider's regional layout is a separate constraint.** The provider may operate regional endpoints that match or do not match our residency boundaries. A model endpoint that advertises "EU" may or may not meet the controller's transfer-mechanism posture; it is the controller's job to attest.

The design therefore has to produce a **failover topology per residency scope**, not one global topology. For `chatcat` with EU + US scopes, the design produces:

- EU scope: active-active across two EU regions, plus a warm third EU region for the tail.
- US scope: active-active across two US regions.
- A strict routing invariant: a request's user-residency tag determines the serving pool; traffic cannot cross-flow across scopes even on failure.
- A hosted-model provider pair per scope: EU-approved provider endpoints only for EU scope; US provider endpoints for US scope.
- A judge pipeline per scope, never cross-scope.

The deliverable is a committed document `designs/chatcat-residency-failover.md` plus:

- `designs/residency-topology.yaml` — machine-readable topology.
- `designs/routing-invariants.md` — the invariants that make the topology defensible.
- `controls/residency-controls.md` — controls that enforce the invariants (admission policies, provider contract clauses, telemetry).
- `audits/residency-attestations.md` — the attestation set that an auditor expects.

### 1.1 Decision rationale — why one topology per scope is the right default

Alternative A: one global topology with "data classification tags" filtering what crosses. This is the pattern most teams reach for first because it lets them share infrastructure. In practice it has two systemic failure modes:

- The classification tag must be present and correct on every record, for every lifecycle stage, in every component. A missing tag defaults to "least restrictive", which is the opposite of the residency posture.
- Debug and incident operations frequently need to read records across scopes; the design becomes "no cross-flow except for incident operators", and the operator path becomes the compliance gap.

Alternative B: fully disjoint stacks per scope. Expensive but simple. The design chooses this where it is cheap (the retrieval index, the session store) and compromises to shared control plane where it is expensive and defensible (deployment automation, metric aggregation on aggregate-only telemetry).

Alternative C: shared control plane, residency-isolated data plane. The chosen design. Control plane (CI/CD, metrics aggregation, dashboards) lives in a residency-neutral region and never touches request content. Data plane (serving, session, retrieval, judge, side-effect ledger) is strictly per-scope.

### 1.2 The residency scope definition

A residency scope is defined by (a) a set of legal/regulatory boundaries, (b) a set of acceptable cloud regions, and (c) a set of acceptable hosted-model-provider endpoints. For `chatcat`:

| Scope | Boundary | Cloud regions | Hosted-model endpoints | Notes |
|-------|----------|---------------|------------------------|-------|
| `eu-gdpr` | EU/EEA | `eu-west-1`, `eu-west-3`, `eu-central-1` | Provider A EU endpoint; Provider B EU endpoint | GDPR applies; transfer mechanism assumed Standard Contractual Clauses + supplementary measures |
| `us-general` | US | `us-east-1`, `us-west-2`, `us-east-2` | Provider A US endpoint; Provider B US endpoint | No federal residency law; state-level (CCPA, etc.) applies at the controller side; no transfer out-of-US without explicit user consent |

Future scopes (e.g., `uk-dpa`, `apac-sg`, `apac-jp`) are added by extending the table and the topology; the invariants in §2.3 do not change.

## 2. Implementation — worked answer

### 2.1 Topology (`designs/residency-topology.yaml`)

```yaml
# chatcat residency-constrained topology
version: 2026-11-13
scopes:
  - id: eu-gdpr
    primary_regions: [eu-west-1, eu-west-3]
    tertiary_region: eu-central-1         # warm, for both-primary-regions impairment
    serving_strategy: active-active
    retrieval:
      storage: vector-db
      replication: cross-region within scope
      snapshot_frequency_hours: 1
      cross_scope_replication: FORBIDDEN
    session_store:
      storage: redis-enterprise
      replication: active-active within scope
      cross_scope_replication: FORBIDDEN
    primary_model_endpoint:
      provider: A
      endpoint: eu-paris
      attestation: see audits/residency-attestations.md#provider-a-eu
    fallback_model_endpoint:
      provider: B
      endpoint: eu-frankfurt
      attestation: see audits/residency-attestations.md#provider-b-eu
    judge_pipeline:
      location: eu-west-1
      snapshot_bucket: eu-west-1
      cross_scope_replication: FORBIDDEN
    side_effect_ledger:
      storage: eu-west-1
      cross_scope_replication: FORBIDDEN

  - id: us-general
    primary_regions: [us-east-1, us-west-2]
    tertiary_region: us-east-2
    serving_strategy: active-active
    retrieval:
      storage: vector-db
      replication: cross-region within scope
      snapshot_frequency_hours: 1
      cross_scope_replication: FORBIDDEN
    session_store:
      storage: redis-enterprise
      replication: active-active within scope
      cross_scope_replication: FORBIDDEN
    primary_model_endpoint:
      provider: A
      endpoint: us-east
      attestation: see audits/residency-attestations.md#provider-a-us
    fallback_model_endpoint:
      provider: B
      endpoint: us-west
      attestation: see audits/residency-attestations.md#provider-b-us
    judge_pipeline:
      location: us-east-1
      snapshot_bucket: us-east-1
      cross_scope_replication: FORBIDDEN
    side_effect_ledger:
      storage: us-east-1
      cross_scope_replication: FORBIDDEN

control_plane:
  location: us-east-1                # control-plane region is residency-neutral; sees no user content
  components: [ci-cd, observability-aggregation, dashboards]
  telemetry_contract:
    metric_labels_allow: [service, region, scope, canary]
    metric_labels_forbid: [user_id, session_id, email, ip, content_hash]
    log_contract: aggregates and cardinality only — no payload, no PII
    trace_contract: per-scope traces stay in scope; aggregated spans flow out with payload redacted
```

### 2.2 Routing invariants (`designs/routing-invariants.md`)

```markdown
# Residency routing invariants for chatcat

These invariants are enforceable and tested. A violation is a P0 incident.

## Invariant 1 — Scope determination at ingress

Every request is tagged with a `scope` label at the ingress gateway. The label is
derived from, in priority order:
1. Authenticated identity's declared residency scope (from the identity service).
2. For anonymous flows: the originating region as inferred from the public IP and
   verified against a residency-matcher table.
3. Default: reject the request with a 400. There is no silent default to a scope.

The `scope` label is immutable for the lifetime of the request and session.

## Invariant 2 — No cross-scope routing

A request tagged `eu-gdpr` is served only by `eu-gdpr` serving pods. The weighted DNS
and the L7 router are configured such that a `scope=eu-gdpr` request routed to a
`us-general` pool is a 500 at the router, not a successful request. This is enforced
in the router config; it is tested in CI.

## Invariant 3 — No cross-scope state

- Session store: a `scope=eu-gdpr` session key cannot be read from the `us-general`
  cluster. This is enforced by the session-store client using a per-scope credential
  pair.
- Retrieval index: separate index per scope; the retrieval client requires a scope tag
  that matches its configured scope.
- Judge pipeline: judge samples carry the scope label and the pipeline refuses
  samples whose label does not match its own scope.
- Side-effect ledger: one ledger per scope; the gate's ledger client is scope-bound.

## Invariant 4 — Failover stays within scope

A regional outage in `eu-west-1` causes traffic to shift to `eu-west-3` (and `eu-central-1`
if both EU primaries are impaired). It does NOT cause traffic to shift to `us-east-1`
under any condition. The DNS and the router configuration explicitly cannot route
`eu-gdpr` to any US region, and a drill is run quarterly to confirm.

## Invariant 5 — Control plane is observational-only

The control-plane region can:
- Receive aggregate metrics with the allowed label set.
- Receive redacted logs (counts, aggregates, metric samples).
- Trigger deployments through the CI/CD pipeline.

The control plane cannot:
- Read user content from any scope.
- Read a session, a transcript, a prompt payload, or a tool-call body.
- Store an identifier that is PII (email, user-id, etc.) unless the field is on the
  short, catalogued allowlist (service name, region, scope, canary, image SHA).

Dashboards that show "sample query" or "example output" are not built. If a sample is
required, it is provided in-scope via the per-scope debug tooling.

## Invariant 6 — Provider-endpoint binding

A `scope=eu-gdpr` request is served by one of the `eu-gdpr` catalogued provider
endpoints and no other. A provider endpoint that is not in the `audits/residency-
attestations.md` catalogue for the scope is refused at the provider-client layer.

## Invariant 7 — Backup, DR, and incident access

- Backups of scope data are stored in-scope.
- DR drills that recover in-scope data recover to an in-scope region.
- Incident responders accessing in-scope data do so from an in-scope bastion with
  audited access; cross-scope access is a separate, logged exception path.
```

### 2.3 Controls (`controls/residency-controls.md`)

```markdown
# Residency controls for chatcat

| Control ID | Name | Invariant | Enforcement | Evidence |
|------------|------|-----------|-------------|----------|
| RC-01 | Ingress scope tagger | 1 | Admission gate on the API gateway | Router config + unit test |
| RC-02 | Scope-bound DNS | 2, 4 | Weighted DNS records are scope-partitioned | Terraform + DNS audit log |
| RC-03 | Scope-bound L7 router | 2 | Envoy config rejects cross-scope routes | Envoy config + integration test |
| RC-04 | Scope-bound session credentials | 3 | Per-scope KMS keys | Session-store audit log |
| RC-05 | Scope-bound retrieval client | 3 | Retrieval service has scope at startup | Service config + startup assertion |
| RC-06 | Scope-bound judge | 3 | Judge sample schema includes `scope`; mismatched samples dropped | Judge log + per-scope snapshot |
| RC-07 | Scope-bound ledger | 3 | Side-effect gate (Rule A from exercise 03) is scope-bound | Ledger client code + audit |
| RC-08 | Control-plane label allowlist | 5 | Observability collectors drop forbidden labels | Collector config + sampling audit |
| RC-09 | Content-free logging | 5 | Log emitters must pass a schema check; payload fields rejected | Log pipeline test |
| RC-10 | Provider endpoint catalogue | 6 | Provider client refuses endpoints not in the catalogue for the scope | Client code + endpoint list |
| RC-11 | In-scope backup | 7 | Backup job runs within the scope; cross-scope copy denied by KMS | Backup bucket policy |
| RC-12 | Audited cross-scope access | 7 | Jump-host with step-up MFA + ticket required for cross-scope read | Access audit log |
| RC-13 | Residency failover drill | 2, 4 | Quarterly drill confirms cross-scope failover is refused | `mod-113/exercise-05` drill log |

## Technical implementation of RC-02 and RC-03

- **Weighted DNS.** `eu-gdpr` uses DNS record set `eu.chatcat.example.com` with endpoints
  in EU regions only. `us-general` uses `us.chatcat.example.com` with endpoints in US
  regions only. The client resolves to its scope's hostname; the two hostnames have no
  shared endpoints.
- **L7 router.** Envoy is configured with a listener per scope and a route table that
  asserts `scope == route.scope`. A request with no scope tag or a mismatched scope is
  rejected at the listener with 400 (bad request) and a telemetry tag
  `residency_violation=true`. A rising rate of `residency_violation=true` is an alert.

## Technical implementation of RC-08 and RC-09

- **Collector allowlist.** The Prometheus scrape config and the OpenTelemetry collector
  strip any label not in the allowlist and increment a `residency_label_stripped_total`
  counter per labelled-field. The counter has an alert if it rises (indicates a
  misbehaving emitter).
- **Log schema check.** Logs flow through a central logging pipeline that validates
  each record against a schema. A record with a payload / PII field triggers the
  pipeline to drop the record and increment a counter. Dropped records are visible in
  the per-emitter alert.
```

### 2.4 Attestations (`audits/residency-attestations.md`)

```markdown
# Residency attestations for chatcat

These attestations are the evidence an auditor expects; they are also what the plan
owner must refresh on the triggers in `mod-113/exercise-01` policy §Maintenance.

## Provider A — EU endpoint attestation

- **Contract clause:** provider-A contract §7 specifies EU-resident processing for the
  `eu-paris` endpoint.
- **Sub-processor list:** provider-A's sub-processor list dated `<date>` lists no
  sub-processor outside the EU for this endpoint.
- **Data-processing agreement:** DPA executed `<date>`, review cadence `<cadence>`.
- **Prompt-and-response retention:** per contract, 0 days (no retention).
- **Status-page region label:** provider-A status page reports `eu-paris` as a distinct
  region; outage events on `eu-paris` are not conflated with `eu-ireland`.

## Provider A — US endpoint attestation

(Same shape, US version.)

## Provider B — EU and US (fallback pair)

(Same shape, with the additional clause about takeover during fallback.)

## Our own in-scope attestations

For each scope:
- Cloud provider regions in use and a reference to the provider's residency documentation.
- KMS key configuration confirming keys are scope-bound (no cross-scope key use).
- Backup bucket policies confirming no cross-scope replication.
- Logging and telemetry allowlists.
- The last residency failover drill (date, scope, result).

## Review cadence
- Annual on each attestation.
- Triggered on: sub-processor list update from a provider; provider endpoint catalogue
  change; cloud provider region change; `mod-107` vendor-risk register movement.
```

### 2.5 Failover design per scope

The failover topology for one scope (`eu-gdpr`) looks like this:

```
                       ┌──────────────────────────────────────────────┐
                       │                 eu-gdpr scope                │
                       │                                              │
   (EU client) ──DNS──> │    eu.chatcat.example.com                   │
                       │       │                                      │
                       │       ├── eu-west-1  (active, weight 50)     │
                       │       ├── eu-west-3  (active, weight 50)     │
                       │       └── eu-central-1 (warm, weight 0)      │
                       │                                              │
                       │    retrieval: eu-west-1 (primary),           │
                       │               eu-west-3 (replica, lag ≤ 1h)  │
                       │                                              │
                       │    session:   eu-west-1 <-> eu-west-3        │
                       │                                              │
                       │    judge:     eu-west-1 (primary)            │
                       │                                              │
                       │    model:     provider-A eu-paris (primary)  │
                       │               provider-B eu-frankfurt (fb)   │
                       └──────────────────────────────────────────────┘
```

Within-scope failover actions:

1. **Single-region impairment (`eu-west-1` down).** DNS re-weights to `eu-west-3`
   (100%) and `eu-central-1` (warm → online at 50%). Retrieval replica is promoted in
   `eu-west-3`. Session failover is automatic (active-active). The provider path is
   unaffected unless `eu-paris` is also down.
2. **Both primaries impaired.** `eu-central-1` carries all traffic. Capacity in
   `eu-central-1` is pre-reserved to the SLO.
3. **All three EU regions impaired.** Degraded mode (see `mod-113/exercise-02` §6). Not
   failover to the US; there is no user-serving posture that includes EU traffic
   traversing into a US region for `eu-gdpr`. If leadership wants a "last-resort
   degraded mode" that crosses scope, it is treated as a separate, explicit, legally-
   reviewed carve-out with explicit user consent at that moment. The design's default is
   "degrade and unavailable, not misrouted".
4. **Primary provider impaired.** Pin to `eu-frankfurt` fallback per exercise 02 §4.

## 3. Validation steps

1. **Static validation of routing invariants.**
   - For each invariant in §2.2, there is an integration test in CI that confirms the
     corresponding control's enforcement. Example: send a synthetic `scope=eu-gdpr`
     request to `us.chatcat.example.com` and assert it returns 400.
2. **Attestation freshness.** Every attestation in `audits/residency-attestations.md`
   has a `last_reviewed_on` date; the CI pipeline fails if any date is older than the
   policy's review cadence.
3. **Scope-boundary drill.** Quarterly drill (`mod-113/exercise-05`) attempts a
   cross-scope read from an in-scope operator bastion; expected result is deny +
   audit-log entry.
4. **Label allowlist proof.** Sample 1,000 recent metrics from the collector and
   confirm no label outside the allowlist appears; a `residency_label_stripped_total`
   counter proves the control is firing when a misbehaving emitter is introduced.
5. **DR drill within scope.** For each scope, run the regional-failover drill from
   `mod-113/exercise-02` and confirm the failover stays within the scope. Cross-scope
   failover is a drill-failure.

## 4. Rubric / review checklist

Score each dimension 0 / 1 / 2. Pass at ≥ 14 / 20 with no zero in dimensions marked
`(hard gate)`.

| # | Dimension | 0 | 1 | 2 |
|---|-----------|---|---|---|
| 1 | Residency scope defined *(hard gate)* | Not defined | Named but not enumerated | Scope with boundary, regions, provider endpoints enumerated |
| 2 | Topology per scope *(hard gate)* | One global topology | Scopes mentioned | Separate active-active topology per scope with tertiary region |
| 3 | Routing invariants *(hard gate)* | None | A couple | 6+ enforceable invariants with tests |
| 4 | Ingress scope tagging *(hard gate)* | Missing | Mentioned | Detailed priority order with reject-on-default |
| 5 | State isolation per scope | Missing | Narrative | Per-scope session, retrieval, judge, ledger with credentials |
| 6 | Provider endpoint binding | Generic provider failover | Per-scope endpoint named | Per-scope endpoint with attestation reference |
| 7 | Control-plane isolation | Not addressed | Mentioned | Label allowlist + log-schema enforcement + evidence |
| 8 | Attestation set | Missing | List | Dated attestations with review cadence and trigger events |
| 9 | Failover without crossing scope | Silently crosses | Mentions limit | Explicit: all-EU-down degrades to unavailable, not cross-scope |
| 10 | Testing and drill | None | Narrative | Quarterly drill fixture confirming cross-scope is refused |

## 5. Common mistakes

- **Silent fallback to the other scope.** The topology "we fail from EU to US if things
  get really bad" is the single most dangerous residency mistake. The default must be
  "unavailable within scope" over "available across scope". If an enterprise contract
  requires cross-scope fallback, that is a separate, legally-reviewed, user-consented
  carve-out — not a default.
- **Classification-tag design.** A single topology filtered by classification tags
  defaults wrong on missing tags and collapses the compliance posture to the classifier's
  accuracy. The reference design prefers a disjoint data plane and a residency-neutral
  control plane.
- **Metric labels with PII.** Dashboards built on `user_id` or `email` labels will leak
  PII to the central observability store. The allowlist-only pattern (RC-08) is cheap
  and the omission is systemic.
- **Provider endpoint drift.** "Our provider has EU presence" is a sales statement, not
  a compliance attestation. The catalogue must be specific endpoints and specific
  contract clauses, and it must be refreshed on sub-processor changes.
- **Logs with request payloads.** The incident-debug pattern of "log the request
  payload on 5xx" quietly undoes the residency posture. Logs must be aggregate-only; a
  payload capture for debug requires an in-scope step-up flow.
- **Cross-scope DR copy.** Shipping backups to a cheaper region is a residency
  violation. Backups stay in scope.
- **Judge pipeline cross-scope.** A single judge pipeline that scores samples from both
  scopes requires the samples to flow across scope. The reference design runs one
  judge per scope and aggregates only the deltas.
- **Incident-time improvisation.** Cross-scope reads during an incident are a known
  need; the design must handle them deliberately (RC-12) so they are logged and
  reviewed, not improvised through shared credentials.
- **Treating control-plane as unconstrained.** CI/CD and dashboards sit in a
  residency-neutral region on purpose — they see metadata, not content. Dashboards
  that render user content break the design.

## 6. References

- NIST SP 800-34 Rev. 1 §3.4 "Create contingency strategies" — the step this exercise
  fulfils under residency constraints.
- NIST AI Risk Management Framework — [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the GOVERN function ("GOVERN 6: addressing AI risks that arise from third-party entities") and the MAP function ("MAP 4: risks are mapped") grounds this design's explicit treatment of hosted-model provider endpoints as named, catalogued dependencies.
- OWASP Machine Learning Security Top 10 — [owasp.org/www-project-machine-learning-security-top-10/](https://owasp.org/www-project-machine-learning-security-top-10/) — `ML06: ML supply chain` motivates the provider-endpoint catalogue and attestation refresh cadence; `ML02: data poisoning` motivates in-scope storage for the retrieval index (cross-scope poisoning is an analogous class).
- MITRE ATLAS — [atlas.mitre.org](https://atlas.mitre.org/) — the "Resource development → acquire external services" and "Exfiltration → ML inference API access" technique classes justify (a) the strict provider-endpoint binding (RC-10) and (b) the aggregate-only telemetry constraint (RC-08/09).
- `mod-107` vendor-and-provider-risk register — the register the attestation set in §2.4 cross-references.
- `mod-108/exercise-06` legal-compliance handoff — the handoff that invokes this
  design's controls when a cross-scope access becomes necessary.
- `mod-113/exercise-01` ISCP — the plan this failover design parameterises per scope.
- `mod-113/exercise-02` regional-outage playbook — the playbook that runs per scope;
  not cross-scope.
