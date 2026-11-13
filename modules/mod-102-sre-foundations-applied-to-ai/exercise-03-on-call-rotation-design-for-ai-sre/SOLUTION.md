# Exercise 03 — On-Call Rotation Design for an AI SRE Team

> Reference solution. The learner is asked to design a sustainable on-call
> rotation for a small AI SRE team (say 6–8 engineers) supporting a 24×7
> model-backed service. "AI-specific" means they have to account for
> (a) failure modes classical SRE doesn't see (quality burns, provider
> degradation, GPU-driver incidents) and (b) the *skill load* of being on-call
> on an AI stack — the on-call must be able to act on model behavior, not
> just infrastructure.

## 1. Solution overview

The Site Reliability Workbook, Ch. 8 ("On-Call") codifies the classical rules:
follow-the-sun where possible; cap primary on-call at ≤ 25% of working hours
on average; two-person on-call per shift (primary + secondary); compensate
on-call; protect post-page recovery time. Those rules survive onto AI SRE
teams unchanged. Three things do have to change:

1. **Skill-gating the rotation.** Not everyone on an AI SRE team can safely
   roll back a model, re-pin a prompt template, or interpret a vLLM metric
   storm on their first week. A tiered rotation (shadow → secondary →
   primary) is required.
2. **Expanded runbook surface.** The on-call runbooks have to cover classical
   infra incidents (GPU driver crash, NCCL timeout, node pressure) and
   AI-specific ones (quality burn, provider degradation, prompt regression,
   retriever drift, unsafe-output report).
3. **Handoff carries *model and config state*.** A classical handoff says
   "the DB primary is still flapping." An AI SRE handoff also has to say
   "we're pinned on model v23.4 after yesterday's quality burn, and the
   fallback provider is receiving 20% of `/chat` traffic." Without that, the
   next shift makes the wrong call.

The worked answer below is a complete rotation design for a team of 7
engineers supporting a global 24×7 inference service.

## 2. Implementation — worked answer

### 2.1 Context (stated by the learner at the top of their submission)

- **Team size:** 7 engineers, two regions (4 in US-Pacific, 3 in EU-Central).
- **Service:** production LLM inference platform fronting `/chat`, `/agent`,
  `/embed`. Multi-tenant internal, external customers on `/chat`.
- **Coverage requirement:** 24×7 primary + secondary.
- **Vendor posture:** one primary external model provider + one secondary
  (configured, tested monthly).

### 2.2 Rotation shape

**Follow-the-sun primary, overlapping secondary.**

| Shift | Hours (UTC) | Primary pool | Secondary pool |
|---|---|---|---|
| Americas | 16:00 – 00:00 UTC | 4 US-Pacific engineers | US-Pacific peer |
| EMEA | 08:00 – 16:00 UTC | 3 EU-Central engineers | EU peer |
| Overnight / early | 00:00 – 08:00 UTC | US-Pacific primary (paid overnight) *or* EMEA secondary | reciprocal |

- **Primary shift length:** 7 days, Mon-to-Mon (long enough to see a weekly
  pattern, short enough not to burn people).
- **Secondary shift length:** 7 days, offset by 3 days from the primary so
  the same engineer is never primary + secondary at the same time.
- **On-call load target:** ≤ 25% of working hours on average (Workbook,
  Ch. 8). With 7 engineers across 2 shifts, this is achievable.
- **Follow-the-sun handoff:** daily 15-minute shift handoff at 16:00 UTC
  (EMEA → Americas) and 08:00 UTC (Americas → EMEA).

### 2.3 Tiered on-call (skill-gating)

Classical SRE has one on-call tier; on an AI stack we need three.

| Tier | Called for | Prerequisite skills | Who |
|---|---|---|---|
| **Shadow** | Receives all pages, does not act. 4 weeks minimum. | None. | Any new hire or team-transfer. |
| **Secondary** | Receives pages, assists primary, is the primary's reviewer for any irreversible action. | Completed shadow, has done ≥ 5 runbook dry-runs including a model rollback and a provider failover. | 6 of 7 engineers. |
| **Primary** | Pages go here. Has pre-authorized rollback / failover / degrade authority per the error-budget policy. | Completed secondary, has handled ≥ 2 real incidents as secondary, has run one incident as primary in a simulated chaos day. | 5 of 7 engineers. |

- The on-call rotation is drawn from the **primary pool** only. Growing the
  pool is a hiring-plan objective, not an incident-time improvisation.
- Pager escalation path: primary → secondary (5-min) → on-call manager
  (15-min) → engineering director (30-min).

### 2.4 Runbook inventory (the surface the on-call is expected to be fluent on)

The rotation only works if the runbooks exist and are *rehearsed*. Minimum
inventory:

**Infrastructure runbooks (classical):**
- GPU driver crash / unrecoverable CUDA error.
- Node pressure (OOM-killer activation, disk pressure on /tmp or model
  cache).
- NCCL / collective timeout on multi-GPU deployments.
- Autoscaler stuck (HPA / KEDA / custom scaler).
- Load-balancer / ingress misroute.

**AI-specific runbooks:**
- Quality burn — reproducible offline (= rollback path).
- Quality burn — not reproducible offline (= quality incident escalation).
- Provider degradation — confirmed.
- Provider degradation — suspected (triage).
- Unsafe-output report (route to safety-incident procedure, which can
  supersede the error-budget policy per Exercise 02).
- KV-cache eviction storm / prefix-cache miss storm (vLLM metrics path).
- Prompt-template regression (pin + roll back + open issue to prompt-owner).
- Retriever / index regression (pin index + re-run eval).
- Token-cost blow-out (apply degradation levers per error-budget policy).

### 2.5 Shift handoff artifact

Handoff is a **written** document, filled in at every shift change. Required
fields:

1. **Open incidents** (ID, status, who owns, next action).
2. **Current error-budget state** per SLO.
3. **Active policy actions** — anything pinned, degraded, failed-over.
   Must include: pinned model version, pinned prompt template version,
   pinned retriever index version, traffic split across providers.
4. **Known risky changes in flight** — model promotion scheduled, prompt
   change proposed, infra maintenance window.
5. **Handoff reader sign-off** (name, timestamp).

The handoff artifact is enforced because the on-call surface for an AI
service is *stateful* in ways classical services aren't — a shift that
inherits a pinned model but doesn't know it is a shift that will approve
a model promotion on top of a frozen rollback.

### 2.6 Sustainability guardrails

- **Hard cap: 2 pages / shift.** If pages exceed cap for two consecutive
  weeks, the service is paged into a reliability-debt review, not the next
  on-call's sleep.
- **No code work on pager day.** Primary's calendar is cleared of scheduled
  commitments. Project work is their choice but not an expectation.
- **Post-incident day off (or equivalent).** Any page that extends past
  midnight local, or any incident > 2 h, triggers a next-day recovery day.
- **Compensation.** On-call is paid or time-banked at the team's established
  rate (reference the org's own policy; this is not optional). (Workbook,
  Ch. 8.)
- **Monthly chaos day.** One scheduled chaos exercise per month (see
  mod-109) that includes an AI-specific failure (provider failover, prompt
  rollback, retriever swap). Rotates through runbooks so each one is
  rehearsed quarterly at minimum.

### 2.7 Decision rationale

- **Tiered on-call solves the "can everyone roll back a model" problem.** If
  the rotation is flat, the weakest primary determines what the on-call can
  safely do. Tiers make it explicit: being on-call means you can pull the
  pre-authorized levers from the error-budget policy. New hires *will* page;
  they just do it as shadow / secondary first.
- **Written handoff is non-negotiable on AI SRE teams.** A verbal "yeah the
  model got rolled back last night" from the outgoing shift is how model
  promotions land on top of frozen rollbacks. Written, with named fields.
- **Monthly chaos day, not only after an incident.** A runbook that is only
  rehearsed after an incident is a runbook that rotted silently for 6
  months. The AI-stack runbook surface is wide (classical + AI); it rotates
  through at ≥ quarterly or it is not trustworthy.
- **Hard cap on pages.** The Workbook's rule (≤ 25% of working hours) is
  necessary but not sufficient — page *count* also matters, because every
  page fragments sleep. 2/shift is a defensible operational cap; above it,
  the service — not the person — is the problem.
- **No AI-specific innovations where they aren't needed.** Compensation,
  handoff, follow-the-sun, and secondary coverage all come from the
  Workbook. The AI SRE delta is tier-gating + runbook surface + stateful
  handoff artifact. Don't reinvent what isn't broken.

## 3. Validation steps

1. **Coverage math works.** The schedule actually covers 24 × 7 primary +
   secondary without any 7-engineer being on-call more than ~25% of their
   working hours.
2. **No single point of knowledge.** At least 3 engineers per region are
   primary-capable.
3. **Runbook inventory is enumerated.** Minimum AI-specific runbooks:
   provider degradation, quality burn (both branches), prompt regression,
   retriever regression, unsafe-output, cost blow-out.
4. **Handoff is a written artifact with named fields.** Not "a slack
   message."
5. **Compensation and recovery are explicit.** Both on-call compensation
   policy and post-incident recovery are named.
6. **Interacts with the error-budget policy.** The on-call is the actor for
   the pre-authorized actions in Exercise 02; those actions must be the
   same lever names in both documents.
7. **Chaos rehearsal cadence is specified.** Without it the runbooks
   atrophy.

## 4. Rubric / review checklist

Score each row independently; passing bar is **14/20** with no row at 0.

| Row | 0 | 2 | 4 |
|---|---|---|---|
| **Schedule math** | Shift coverage incomplete or burns people > 25%. | Coverage complete, load unclear. | 24×7 primary + secondary with explicit load math ≤ 25%. |
| **Tier-gating** | Flat rotation. | Shadow period mentioned. | Shadow / secondary / primary with named prerequisites. |
| **Runbook surface** | < 3 AI-specific runbooks listed. | 3–4 listed. | Complete inventory: provider degradation, quality burn (both branches), prompt, retriever, unsafe-output, cost. |
| **Handoff artifact** | Verbal / informal. | Written, free-form. | Written, with required fields including pinned model / prompt / retriever / traffic split. |
| **Sustainability** | No compensation or load caps. | Compensation only. | Compensation + page caps + post-incident recovery + chaos cadence. |

## 5. Common mistakes

- **Flat rotation.** Treats "on-call" as a uniform skill. On an AI stack the
  action surface differs by engineer; without tiers, either the weakest
  primary dictates authority, or the strongest primary burns out.
- **Overcounting people.** A 7-person team cannot sustainably maintain 2×2
  coverage without the follow-the-sun / regional structure or vendor-backed
  follow-the-sun. The arithmetic has to actually close.
- **No runbooks for AI-specific failure modes.** The on-call is then
  improvising at 3 a.m. against a quality burn.
- **Verbal handoff only.** Stateful configurations (pinned model, pinned
  prompt, traffic split) are invisible to the next shift, so the next shift
  undoes them.
- **No chaos / rehearsal cadence.** Runbooks that are only used during real
  incidents rot. On an AI stack with high change velocity this happens in
  weeks, not quarters.
- **Treating compensation as optional.** Workbook explicitly: on-call is
  compensated. Not optional.
- **Rotation disconnected from the error-budget policy.** The primary
  on-call is *the* actor for pre-authorized rollbacks and failovers. If the
  rotation design doesn't name that, Exercise 02 doesn't work.

## 6. References

- *The Site Reliability Workbook*, Ch. 8 "On-Call" (load caps, two-person
  shift, compensation, handoff hygiene) and Ch. 9 "Incident Response" —
  <https://sre.google/workbook/table-of-contents/>.
- Google SRE Book, Ch. 11 "Being On-Call" —
  <https://sre.google/sre-book/being-on-call/>.
- NIST AI Risk Management Framework, `GOVERN` and `MANAGE` functions —
  grounds the need for pre-authorized response roles —
  <https://www.nist.gov/itl/ai-risk-management-framework>.
- OWASP Machine Learning Security Top 10 (motivates the unsafe-output
  runbook) — <https://owasp.org/www-project-machine-learning-security-top-10/>.
- MITRE ATLAS — threat-informed scenarios for the chaos-day cadence —
  <https://atlas.mitre.org/>.
