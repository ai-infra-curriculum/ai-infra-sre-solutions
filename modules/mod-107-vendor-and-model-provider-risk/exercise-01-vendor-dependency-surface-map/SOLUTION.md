# Exercise 01 — Vendor Dependency Surface Map (Solution)

## 1. Solution overview

A vendor dependency surface map inventories every external third-party that sits
on a request path, data path, or control path of the AI system. The goal is not
a procurement spreadsheet; the goal is a reliability artifact that lets
on-call and architecture reviewers answer three questions in under a minute:

1. If vendor `X` is degraded right now, which of our user-facing surfaces break,
   and how?
2. Where are we single-threaded on one provider, and is that deliberate?
3. For each external dependency, which internal owner is accountable for
   the contract, the quota, the auth material, and the incident response?

The surface map is the base artifact every other exercise in this module
plugs into: fallback contracts (ex-02), case-study replay (ex-03),
status-page ingestion (ex-04), SLA/SLO reconciliation (ex-05), and vendor
scorecards (ex-06) all key off the same inventory. Treat it as a living
file in the repo, not a one-off audit.

The schema below is intentionally close to standards practitioners already
use: the NIST AI RMF `MAP` function asks organizations to catalogue
third-party AI components and their context of use, and the OWASP ML Top 10
risks `ML06: AI Supply Chain Attacks` and `ML05: Model Theft` both assume the
team can enumerate which external models/components they consume.

## 2. Worked implementation

### 2.1 Dependency record schema

Each external dependency is one YAML record. Fields marked `(required)` are
the minimum for a row to count as populated.

```yaml
# vendor-surface-map/openai-chat-completions.yaml
id: openai-chat-completions                       # (required) stable slug
display_name: "OpenAI Chat Completions API"       # (required)
vendor: openai                                    # (required) short vendor key
category: model-provider                          # (required) see taxonomy below
criticality: tier-1                               # (required) tier-1 | tier-2 | tier-3
owner_team: inference-platform                    # (required) accountable team
secondary_contacts:
  - team: security
    reason: data egress, DPA owner
  - team: finance
    reason: committed-spend / enterprise contract

# What it does for us, in one sentence an SRE on another team can read.
purpose: "Primary LLM provider for the product chat surface and agentic eval harness."

# Where it sits in the request graph.
request_path:
  surfaces_impacted:
    - product-chat
    - agent-eval-harness
  entry_points:
    - service: chat-gateway
      endpoints:
        - path: /v1/chat/completions
          method: POST
          upstream_host: api.openai.com
          auth: bearer-token (vault path: secret/data/openai/prod)
          egress_cidr: via egress-proxy-pool-a
  blocking: true        # Request fails user-visibly if this hop fails
  streaming: true       # Response is SSE/stream
  tool_calling: true    # Request uses function/tool call schema

# What leaves our boundary.
data_flow:
  payload_classes:
    - user-chat-content
    - system-prompt (contains redacted internal docs)
  pii_present: possible         # user free text; relies on upstream redaction
  training_opt_out: true        # vendor setting confirmed — link in references
  retention: "30 days per vendor policy"   # verify quarterly
  region_pinning: "us only"     # per DPA; needs verification each contract cycle

# How we know it's up or down.
telemetry:
  our_signals:
    - name: chat_gateway_openai_error_rate
      type: prometheus
      alert: openai_error_budget_burn
    - name: chat_gateway_openai_latency_p95_ms
      type: prometheus
  vendor_signals:
    - name: openai-status
      url: https://status.openai.com/
      ingestion: see exercise-04 pipeline
      components_watched:
        - API
        - ChatGPT      # correlated with API-side regressions in some incidents

# Contractual and quota context.
commercials:
  contract: enterprise                 # enterprise | pay-as-you-go | free
  committed_spend_per_month_usd: "<verify>"
  rate_limit:
    scope: org-level TPM + RPM, per model
    headroom_target: "<=70% utilization at p95"
  sla_availability: "see vendor terms"  # do not paste a number without link
  credit_remedy: "service credits only, no direct SLO alignment"
  data_processing_addendum: signed, vault path: contracts/openai/dpa-v3.pdf

# Failure modes we have actually observed or planned for.
known_failure_modes:
  - id: regional-degradation
    symptom: elevated 5xx and timeouts on api.openai.com
    detection_lag_seconds_target: 60
    mitigation: fallback to secondary provider (see ex-02 contract)
  - id: model-deprecation
    symptom: 400s referencing deprecated model id
    detection: pre-flight model probe job
    mitigation: pinned model alias + migration ticket cadence
  - id: rate-limit-exhaustion
    symptom: 429 with retry-after header
    detection: TPM/RPM saturation alert
    mitigation: queue-shaping + cross-provider spill

# Concentration-risk markers (feeds exercise-06 scorecard).
concentration_risk:
  share_of_tier_1_traffic_pct: 85     # 85% of tier-1 chat traffic flows here
  substitutable_in_minutes: 15        # time to shift majority of traffic to fallback
  exclusive_capabilities:
    - "logprobs on this model family"  # feature we would lose on fallback
  lock_in_factors:
    - fine-tuned-models hosted here
    - vendor-native evals

links:
  runbooks:
    - docs/runbooks/openai-degraded.md
  past_incidents:
    - incidents/INC-<n>: <title>        # link entries from exercise-03
  vendor_docs:
    - https://platform.openai.com/docs
```

Every dependency — not just LLM providers — gets a record. The taxonomy below
makes sure the map covers the full surface, not just the obvious one.

### 2.2 Category taxonomy

| Category | Typical examples | Why it belongs on the map |
|---|---|---|
| `model-provider` | Hosted chat / completion / embedding APIs | Primary request-path dependency |
| `inference-platform` | Managed model hosting, vLLM-as-a-service, GPU cloud for your own weights | Owns the serving substrate; outage looks identical to a model-provider outage |
| `foundation-model-weights` | Hugging Face hub, model zoo, licensed weight distribution | Supply-chain surface; maps to OWASP ML06 |
| `vector-store` | Managed vector DB, hybrid search | Retrieval failure causes silent quality collapse, not hard errors |
| `embeddings-provider` | Hosted embedding APIs | Same failure mode as model-provider but usually blast-radius is retrieval, not generation |
| `evaluation-service` | Hosted eval / judge / guardrail APIs | Non-blocking in normal operation; blocking at release gates |
| `safety-filter` | Content moderation, toxicity, PII detection | Legal/safety obligations; failure often means fail-closed and user-visible refusals |
| `observability` | Telemetry pipelines, LLM-trace SaaS, APM | Not request-blocking, but losing it blinds incident response |
| `identity-and-secrets` | Vault-as-a-service, IdP, secrets manager | Loss breaks every auth'd upstream call simultaneously |
| `cdn-and-edge` | Edge router, API gateway SaaS | Fronts all providers; single blast radius for many vendors |
| `data-ingress-egress` | Managed Kafka, object storage, streaming ETL | Gates training data / RAG freshness |

### 2.3 Criticality tiers (what "tier-1" means)

Record every dependency, but prioritize deep fields for tier-1. Suggested
cuts, which an SRE lead signs off:

- `tier-1` — If this vendor is down, a user-visible surface fails or refuses.
  The team has an alert, a runbook, a named primary owner, and a tested
  fallback or documented graceful degradation.
- `tier-2` — Degrades a surface but has either a built-in retry window, an
  async retry path, or a background job that can backfill.
- `tier-3` — Internal tooling, dev loop, batch jobs. On-call is informed but
  not paged.

Tiering is a decision, not a measurement. Record who decided and when.

### 2.4 Worked example output

A populated map for a small inference stack looks like:

```
vendor-surface-map/
├── README.md                              # how to read, who owns
├── index.yaml                             # machine-readable list
├── model-provider/
│   ├── openai-chat-completions.yaml
│   ├── anthropic-messages.yaml            # tier-1 fallback for chat
│   └── openai-embeddings.yaml
├── inference-platform/
│   └── self-hosted-vllm-cluster.yaml
├── vector-store/
│   └── managed-vector-db.yaml
├── evaluation-service/
│   └── hosted-judge-api.yaml
├── safety-filter/
│   └── moderation-api.yaml
├── observability/
│   ├── llm-trace-saas.yaml
│   └── apm-vendor.yaml
└── identity-and-secrets/
    └── cloud-secrets-manager.yaml
```

The `index.yaml` is the contract any script or dashboard can consume:

```yaml
version: 1
generated_at: "<set by CI job>"
dependencies:
  - id: openai-chat-completions
    tier: 1
    owner: inference-platform
    surfaces: [product-chat, agent-eval-harness]
    path: model-provider/openai-chat-completions.yaml
  - id: anthropic-messages
    tier: 1
    owner: inference-platform
    surfaces: [product-chat]
    role: fallback
    path: model-provider/anthropic-messages.yaml
  - id: openai-embeddings
    tier: 2
    owner: retrieval-platform
    surfaces: [rag-pipeline]
    path: model-provider/openai-embeddings.yaml
  - id: managed-vector-db
    tier: 2
    owner: retrieval-platform
    surfaces: [rag-pipeline]
    path: vector-store/managed-vector-db.yaml
  - id: hosted-judge-api
    tier: 3
    owner: ml-quality
    surfaces: [release-gates]
    path: evaluation-service/hosted-judge-api.yaml
  - id: moderation-api
    tier: 1
    owner: trust-and-safety
    surfaces: [product-chat]
    role: guardrail
    path: safety-filter/moderation-api.yaml
  - id: cloud-secrets-manager
    tier: 1
    owner: platform-security
    surfaces: [all]
    role: transversal
    path: identity-and-secrets/cloud-secrets-manager.yaml
```

### 2.5 Mapping to standards

- **NIST AI RMF (`MAP` function, `MAP 4.1` subcategory on third-party risks)**:
  the surface map is the evidence artifact for "approaches for mapping AI
  technology and legal risks of its components — including third-party AI
  technologies — are in place." Each dependency record is one row of that
  evidence. The `GOVERN` function (`GOVERN 6.1`, `GOVERN 6.2`) expects a
  policy that assigns organizational accountability for third-party risks;
  `owner_team` and `secondary_contacts` satisfy that.
- **OWASP Machine Learning Security Top 10**: `ML05 Model Theft` and `ML06 AI
  Supply Chain Attacks` are the two risks the surface map helps bound. The
  map tells you which model weights you pulled from where, and which hosted
  models would leak prompts or outputs if the provider were compromised.
- **MITRE ATLAS**: the `AML.T0010 ML Supply Chain Compromise` technique and
  the `AML.TA0002 Reconnaissance` tactic page both assume defenders can
  enumerate upstream model sources. Having the surface map is the
  pre-condition for ATLAS-style threat modeling of the request path.

### 2.6 Walkthrough — building the map from scratch

1. **List every outbound call the production AI services make.** Start from
   egress firewall logs or the egress proxy allowlist; cross-check against
   service IaC (Terraform/Helm) for hard-coded endpoints and against the
   secrets store for credentials that imply an upstream. The union of those
   three sources is the first candidate list. Nothing in-cluster to-in-cluster
   counts yet.
2. **Collapse to logical dependencies.** `api.openai.com/v1/chat/completions`
   and `api.openai.com/v1/embeddings` are two dependencies, not one, because
   they fail independently, have different quotas, and are owned by different
   teams downstream of you.
3. **Interview the owning team for each candidate.** The questions are the
   fields in the schema. If a field cannot be answered, that is a finding;
   record it and open a ticket.
4. **Assign tiers.** Walk each entry with the on-call lead for the surface it
   affects. Tier assignment is the single most consequential field: it drives
   paging, scorecards, and whether a fallback contract (ex-02) is required.
5. **Rank concentration risk.** For each tier-1 vendor, compute
   `share_of_tier_1_traffic_pct` from the last 28 days of traffic and note
   the `substitutable_in_minutes`. Anything above 75% share with a
   substitution window greater than an hour is a concentration finding.
6. **Wire the map into CI.** Add a lint job that fails if
   (a) any service has a secret in vault whose path is not referenced by
   any record, (b) any record is missing a required field, or
   (c) any record has not been reviewed in the last quarter.
7. **Review quarterly.** The reviewers are the owning team lead plus the
   SRE lead for the affected surface. A dated signature line on each file
   is sufficient evidence.

### 2.7 Common pitfalls the schema deliberately rejects

- **"Vendor" rows that are really feature flags.** If `anthropic-messages`
  is in the record set but has zero production traffic and no tested
  fallback plan, it is an aspiration, not a dependency. Mark it
  `role: planned` and exclude it from tier counts.
- **Observability vendors recorded as tier-3.** Losing your trace backend
  mid-incident is a tier-1 problem for incident response even if users do
  not see it. Treat transversal vendors (observability, identity, egress)
  as tier-1 by default and demand justification for lower tiers.
- **One record per cloud provider.** "AWS" is not a dependency; the specific
  managed service with its own SLA, region, and owner team is.

## 3. Validation steps

A learner can self-check without any custom tooling:

1. **Schema lint.** Run a YAML linter (`yamllint`) on every record and the
   index. All records must parse.
2. **Required-fields check.** Write a short script (Python or `yq`) that
   fails if any file is missing `id`, `vendor`, `category`, `criticality`,
   or `owner_team`:
   ```bash
   # macro-level check, no external deps
   yq eval 'select(.id == null or .vendor == null or .category == null \
     or .criticality == null or .owner_team == null) | .id // input_filename' \
     vendor-surface-map/**/*.yaml
   ```
3. **Index consistency.** Every file under category directories must appear
   in `index.yaml`, and every path listed in `index.yaml` must exist on disk.
4. **Egress cross-check.** For a real deployment, diff the egress allowlist
   against `request_path.entry_points[].upstream_host`. Mismatches are
   findings.
5. **Tier sanity.** No more than ~5 tier-1 rows is the common healthy range
   for a small platform. If the map has 15 tier-1 rows, the exercise is
   not failed, but the learner should flag it: paging on 15 vendors at once
   is not a reliability posture.
6. **Peer review.** A teammate who did not author the map must be able to
   read a single file and answer the three overview questions
   (Section 1) without Slack lookups. This is the final gate.

## 4. Rubric

| Dimension | 0 — missing | 1 — partial | 2 — complete |
|---|---|---|---|
| **Coverage** | Only model-provider rows exist | Model-provider + at least one of {vector-store, observability, identity} | All applicable categories from Section 2.2 are either represented or explicitly marked N/A with reason |
| **Required fields** | Missing >1 required field across map | All required fields on tier-1; tier-2/3 partially populated | All required fields on all rows; `<verify>` or an explicit `<!-- cite-primary-source: … -->` placeholder used where actual value is unknown |
| **Tiering** | No tiers, or everything is tier-1 | Tiers assigned but no decision log | Tiers assigned, decision owner recorded, criteria documented in the README |
| **Data flow** | No `data_flow` block | `payload_classes` named but `pii_present`, `training_opt_out`, or `retention` not addressed | All data-flow fields present; opt-out and retention are each linked to a vendor doc or contract path |
| **Failure modes** | None listed | 1 failure mode per tier-1 vendor, detection-only | Each tier-1 vendor has ≥2 failure modes with detection AND mitigation; each mitigation is either a runbook link or an ex-02 fallback-contract reference |
| **Concentration risk** | Not addressed | `share_of_tier_1_traffic_pct` set but no substitution plan | Share, substitution window, exclusive capabilities, and lock-in factors all populated for tier-1 |
| **Standards traceability** | No reference to NIST AI RMF, OWASP ML Top 10, or MITRE ATLAS | At least one reference in the README | README cites the specific NIST AI RMF subcategory (`MAP 4.1` and/or `GOVERN 6.1`), the applicable OWASP ML Top 10 risks, and the relevant MITRE ATLAS techniques, with URLs |
| **Operability** | Not in CI | Lint runs locally | Lint runs in CI, blocks merge on missing required fields and stale review dates |

Passing score: ≥ 2 on **Required fields**, **Tiering**, **Failure modes**,
and **Operability**, plus an average of ≥ 1.5 across all dimensions.

## 5. Common mistakes

- **Treating the map as documentation, not code.** If the map is not linted
  in CI and not consumed by at least one downstream artifact (status
  ingestion in ex-04, scorecard in ex-06), it rots within a quarter.
- **Confusing SLA with SLO in `commercials.sla_availability`.** The vendor
  SLA is a billing instrument, not a reliability target. Keep it factual
  here; reconcile it against user-facing SLOs in exercise-05.
- **Omitting the observability stack.** When the LLM provider is down and
  the trace backend is also degraded, the surface map must have already
  predicted that. Including observability on the map at tier-1 is the
  cheapest way to prevent that oversight.
- **Padding `known_failure_modes` with hypotheticals.** List only failure
  modes that are either observed in production, in a vendor status-page
  archive, or in a public case study (see exercise-03). Hypotheticals
  dilute the signal.
- **One mega-file.** A single `vendors.yaml` with every record collapses
  the ownership story. One file per record, one owner per file, one PR
  per change.
- **Tiering by spend.** Spend is useful for exercise-06; it is a weak
  proxy for criticality. A $200/month safety filter can be tier-1 when a
  $20k/month embedding backfill job is tier-3.

## 6. References

- **NIST AI Risk Management Framework (AI RMF 1.0)** — the `MAP` and
  `GOVERN` functions define third-party risk mapping and governance
  expectations this exercise maps to. See
  <https://www.nist.gov/itl/ai-risk-management-framework> for the current
  framework and companion Playbook. Cite the specific subcategory
  (`MAP 4.1`, `GOVERN 6.1`) in the map's README when attesting compliance.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  `ML05 Model Theft` and `ML06 AI Supply Chain Attacks` are the two risks
  most directly supported by a complete surface map.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The surface map is the
  precondition for threat-modeling with the `AML.T0010 ML Supply Chain
  Compromise` technique and the `AML.TA0002 Reconnaissance` tactic.
- **Downstream artifacts in this module.** The map is consumed by
  `exercise-02-provider-fallback-contract-authoring-with-litellm-shape`
  (one fallback contract per tier-1 model-provider row),
  `exercise-04-provider-status-ingestion-into-incident-pipeline` (uses
  `telemetry.vendor_signals`),
  `exercise-05-sla-vs-slo-reconciliation-and-error-budget-sharing` (uses
  `commercials.sla_availability`), and
  `exercise-06-vendor-eval-scorecard-for-single-vs-multi-provider`
  (uses `concentration_risk`).
