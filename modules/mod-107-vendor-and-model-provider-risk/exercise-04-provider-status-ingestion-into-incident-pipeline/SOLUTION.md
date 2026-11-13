# Exercise 04 — Provider Status Ingestion Into Incident Pipeline (Solution)

## 1. Solution overview

The exercise is to wire external model-provider status feeds into the
same incident pipeline that handles internal pages, so that:

- A vendor-declared impact becomes a first-class incident timeline
  event in the team's incident system, not a Slack message someone
  might miss.
- Each impact event is correlated to the internal services that depend
  on that vendor (via the vendor dependency surface map from ex-01),
  so the on-call sees which of their surfaces are at risk the moment
  the vendor flips.
- Status-page scraping, polling, or webhook handling is centralized
  and observable so the ingestion itself does not become a silent
  dependency.

Major providers publish status in multiple shapes. Status pages
hosted on the Atlassian Statuspage platform (used by OpenAI, Anthropic,
and others at the time of writing) expose a standard JSON API and an
optional webhook mechanism. Google Cloud's status site publishes an
Atom/RSS feed and a JSON dashboard. The ingestion pipeline normalizes
these to one internal event schema.

The pipeline is intentionally small: three stages (fetch, normalize,
dispatch) with explicit idempotency and backoff. Reliability of the
ingestion is itself in scope: if the ingester silently fails, the team
loses a signal they think they have.

## 2. Worked implementation

### 2.1 Architecture at a glance

```
+-------------------+        +------------------+        +--------------+
|  Provider feeds   |  pull  |   Ingester       |  push  |  Incident    |
|  (Statuspage API, |------->|  (fetch +        |------->|  pipeline    |
|   RSS/Atom, webhook)       |   normalize)     |        |  (PagerDuty/ |
+-------------------+        |                  |        |   Incident.io|
                             |    de-dupe cache |        |   Opsgenie)  |
                             +------------------+        +--------------+
                                       |
                                       | emit metrics + logs
                                       v
                               +----------------+
                               | Prometheus/OTel|
                               +----------------+
```

### 2.2 Internal event schema

The one schema every provider adapter normalizes to.

```yaml
# incident-pipeline/schemas/vendor-impact-event.yaml
kind: VendorImpactEvent
version: 1
required:
  - event_id            # deterministic id for idempotency
  - vendor              # matches ex-01 vendor key
  - vendor_incident_id  # vendor's own id for de-dupe across polls
  - status              # investigating | identified | monitoring | resolved
  - impact              # none | minor | major | critical
  - components          # list of vendor component names ("API", "ChatGPT", ...)
  - started_at          # ISO 8601 UTC
  - updated_at          # ISO 8601 UTC
  - resolved_at         # ISO 8601 UTC, nullable
  - title
  - body                # last-update text from the vendor
  - source_url
  - affects_tier1       # computed after cross-ref with vendor surface map
  - affected_surfaces   # list of internal surfaces, from surface map
```

`event_id` is `sha256(vendor + ":" + vendor_incident_id + ":" + updated_at)`,
so polling the same unchanged incident twice produces the same id and
is suppressed by the downstream pipeline.

### 2.3 Provider adapters

Two adapters cover ~every major provider used in practice today. Add
one per provider; they all emit the same normalized event.

#### 2.3.1 Statuspage JSON adapter (OpenAI, Anthropic, and others)

Atlassian Statuspage exposes a stable public JSON API; the canonical
URL pattern is `https://<status-host>/api/v2/incidents.json` for
historical listing and `.../incidents/unresolved.json` for currently
open. The payload fields used below (`id`, `name`, `status`, `impact`,
`incident_updates`, `components`, `started_at`, `updated_at`,
`resolved_at`) are documented at
<https://developer.statuspage.io/> (confirm against the host you
target; small hosts occasionally gate the API).

```python
# ingester/adapters/statuspage.py
"""
Pulls the Statuspage API for a vendor and emits VendorImpactEvent
records. One instance per vendor.
"""
import hashlib
import os
import time
from dataclasses import dataclass, asdict
from typing import Iterator, Optional

import httpx


STATUS_TO_INTERNAL = {
    "investigating": "investigating",
    "identified":    "identified",
    "monitoring":    "monitoring",
    "resolved":      "resolved",
    "postmortem":    "resolved",
}

IMPACT_TO_INTERNAL = {
    "none":     "none",
    "minor":    "minor",
    "major":    "major",
    "critical": "critical",
    "maintenance": "none",   # we filter these out downstream
}


@dataclass
class VendorImpactEvent:
    event_id: str
    vendor: str
    vendor_incident_id: str
    status: str
    impact: str
    components: list
    started_at: str
    updated_at: str
    resolved_at: Optional[str]
    title: str
    body: str
    source_url: str
    affects_tier1: bool = False
    affected_surfaces: Optional[list] = None


class StatuspageAdapter:
    def __init__(
        self,
        vendor: str,
        base_url: str,
        timeout_s: float = 10.0,
        user_agent: str = "vendor-status-ingester/1.0 (sre@example.com)",
    ):
        self.vendor = vendor
        self.base_url = base_url.rstrip("/")
        self.client = httpx.Client(
            timeout=timeout_s,
            headers={"User-Agent": user_agent, "Accept": "application/json"},
        )

    def fetch_open(self) -> list[dict]:
        r = self.client.get(f"{self.base_url}/api/v2/incidents/unresolved.json")
        r.raise_for_status()
        return r.json().get("incidents", [])

    def fetch_recent(self) -> list[dict]:
        # Historical endpoint returns most-recent-first; cap for memory
        r = self.client.get(f"{self.base_url}/api/v2/incidents.json")
        r.raise_for_status()
        return r.json().get("incidents", [])[:50]

    def to_events(self, incidents: list[dict]) -> Iterator[VendorImpactEvent]:
        for inc in incidents:
            impact = IMPACT_TO_INTERNAL.get(inc.get("impact", "none"), "none")
            if impact == "none":
                continue  # filter maintenance / marketing updates
            status = STATUS_TO_INTERNAL.get(inc.get("status", ""), "investigating")
            vendor_id = inc["id"]
            updated_at = inc["updated_at"]
            event_id = hashlib.sha256(
                f"{self.vendor}:{vendor_id}:{updated_at}".encode()
            ).hexdigest()
            last_update_body = ""
            updates = inc.get("incident_updates") or []
            if updates:
                last_update_body = updates[0].get("body", "")  # newest-first
            components = [c["name"] for c in inc.get("components", [])]
            yield VendorImpactEvent(
                event_id=event_id,
                vendor=self.vendor,
                vendor_incident_id=vendor_id,
                status=status,
                impact=impact,
                components=components,
                started_at=inc.get("started_at") or inc["created_at"],
                updated_at=updated_at,
                resolved_at=inc.get("resolved_at"),
                title=inc.get("name", ""),
                body=last_update_body,
                source_url=inc.get("shortlink") or f"{self.base_url}/incidents/{vendor_id}",
            )
```

#### 2.3.2 Atom/RSS adapter (Google Cloud / Vertex AI / Gemini)

Google Cloud's status site publishes a JSON dashboard and an RSS feed.
A generic Atom/RSS adapter covers Google Cloud and any other provider
that only exposes a feed.

```python
# ingester/adapters/atom.py
"""
Minimal Atom/RSS adapter. Feed items are normalized to the same
VendorImpactEvent shape as Statuspage. Status / impact mapping is
coarser because feeds do not carry structured severity.
"""
import hashlib
from typing import Iterator, Optional

import feedparser
from .statuspage import VendorImpactEvent


KEYWORD_TO_IMPACT = {
    # Conservative: unless the vendor explicitly says "major" / "severe",
    # treat as minor and let the downstream rules decide.
    "resolved":  "none",
    "degraded":  "minor",
    "incident":  "minor",
    "outage":    "major",
    "severe":    "critical",
    "disruption": "major",
}


class AtomAdapter:
    def __init__(self, vendor: str, feed_url: str):
        self.vendor = vendor
        self.feed_url = feed_url

    def to_events(self) -> Iterator[VendorImpactEvent]:
        feed = feedparser.parse(self.feed_url)
        for entry in feed.entries:
            title_lower = (entry.get("title") or "").lower()
            impact = "minor"
            for kw, mapped in KEYWORD_TO_IMPACT.items():
                if kw in title_lower:
                    impact = mapped
            if impact == "none":
                continue
            status = "resolved" if "resolved" in title_lower else "investigating"
            vendor_id = entry.get("id") or entry.get("link") or entry.get("title")
            updated_at = entry.get("updated") or entry.get("published")
            event_id = hashlib.sha256(
                f"{self.vendor}:{vendor_id}:{updated_at}".encode()
            ).hexdigest()
            yield VendorImpactEvent(
                event_id=event_id,
                vendor=self.vendor,
                vendor_incident_id=vendor_id,
                status=status,
                impact=impact,
                components=[],       # feeds don't expose structured components
                started_at=entry.get("published") or updated_at,
                updated_at=updated_at,
                resolved_at=None,
                title=entry.get("title", ""),
                body=entry.get("summary", ""),
                source_url=entry.get("link", ""),
            )
```

### 2.3.3 Webhook receiver (optional, for providers that offer it)

Statuspage supports outbound webhooks on incident updates (documented
at <https://support.atlassian.com/statuspage/>). When a vendor exposes
webhooks for your subscription, prefer them over polling: lower latency
and lower rate-limit risk. Build a small HTTP receiver that
authenticates, validates signature if provided, and emits the same
`VendorImpactEvent`. Fall back to polling even when webhooks are
configured — webhook delivery is best-effort.

### 2.4 Correlation with the vendor dependency surface map

Events arrive without context. Correlation turns them into actionable
signals by looking up the vendor and affected components in the ex-01
surface map and tagging `affects_tier1` and `affected_surfaces`.

```python
# ingester/correlate.py
import yaml
from pathlib import Path

def load_surface_map(root: Path) -> dict:
    index = yaml.safe_load((root / "index.yaml").read_text())
    by_vendor = {}
    for dep in index["dependencies"]:
        path = root / dep["path"]
        full = yaml.safe_load(path.read_text())
        by_vendor.setdefault(full["vendor"], []).append({
            "id": full["id"],
            "tier": int(full["criticality"].split("-")[1]),
            "surfaces": full.get("request_path", {}).get("surfaces_impacted", []),
            "components_watched": [
                sig.get("components_watched", [])
                for sig in full.get("telemetry", {}).get("vendor_signals", [])
            ],
        })
    return by_vendor


def enrich(event, by_vendor):
    deps = by_vendor.get(event.vendor, [])
    if not deps:
        return event  # unknown vendor — pass through flagged elsewhere
    # Match vendor incident components to any dependency's watched set.
    relevant = []
    for d in deps:
        watched = {c for cs in d["components_watched"] for c in cs}
        if not watched or any(c in watched for c in event.components):
            relevant.append(d)
    if not relevant:
        return event
    event.affects_tier1 = any(d["tier"] == 1 for d in relevant)
    event.affected_surfaces = sorted({s for d in relevant for s in d["surfaces"]})
    return event
```

### 2.5 Dispatch to the incident pipeline

The dispatcher is the single integration point with the team's incident
system. The example below targets PagerDuty's Events API v2, which
accepts `trigger`, `acknowledge`, and `resolve` actions keyed on a
`dedup_key`. The same primitives exist in incident.io, Opsgenie,
Grafana Incident, etc. — rebind the function body, keep the shape.

```python
# ingester/dispatch.py
import os, httpx

PAGERDUTY_URL = "https://events.pagerduty.com/v2/enqueue"

SEVERITY = {
    "critical": "critical",
    "major":    "error",
    "minor":    "warning",
    "none":     "info",
}

ROUTING_TIER1 = os.environ["PD_ROUTING_KEY_TIER1"]   # pages on-call
ROUTING_FYI   = os.environ["PD_ROUTING_KEY_FYI"]     # informational channel


def dispatch(event) -> None:
    action = "resolve" if event.status == "resolved" else "trigger"
    routing = ROUTING_TIER1 if event.affects_tier1 else ROUTING_FYI
    payload = {
        "routing_key": routing,
        "event_action": action,
        "dedup_key": f"vendor:{event.vendor}:{event.vendor_incident_id}",
        "payload": {
            "summary": f"[{event.vendor}] {event.title}",
            "source": event.vendor,
            "severity": SEVERITY[event.impact],
            "component": ",".join(event.components) or "unspecified",
            "custom_details": {
                "status": event.status,
                "impact": event.impact,
                "started_at": event.started_at,
                "updated_at": event.updated_at,
                "resolved_at": event.resolved_at,
                "body": event.body,
                "source_url": event.source_url,
                "affects_tier1": event.affects_tier1,
                "affected_surfaces": event.affected_surfaces or [],
            },
            "links": [{"href": event.source_url, "text": "vendor status page"}],
        },
    }
    r = httpx.post(PAGERDUTY_URL, json=payload, timeout=10)
    r.raise_for_status()
```

### 2.6 The loop and observability

```python
# ingester/main.py
import json, logging, os, time
from pathlib import Path

from prometheus_client import start_http_server, Counter, Gauge, Histogram

from .adapters.statuspage import StatuspageAdapter
from .adapters.atom import AtomAdapter
from .correlate import load_surface_map, enrich
from .dispatch import dispatch

log = logging.getLogger("ingester")
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

INGESTER_POLL = Counter("vendor_status_poll_total", "Polls by vendor and outcome", ["vendor", "outcome"])
INGESTER_LAG  = Gauge("vendor_status_last_success_seconds", "Seconds since last successful fetch", ["vendor"])
INGESTER_EVT  = Counter("vendor_status_events_total", "Emitted events", ["vendor", "impact", "affects_tier1"])
FETCH_HIST    = Histogram("vendor_status_fetch_seconds", "Fetch latency", ["vendor"])

SEEN_PATH = Path(os.environ.get("INGESTER_STATE", "/var/lib/ingester/seen.json"))


def load_seen() -> set[str]:
    if SEEN_PATH.exists():
        return set(json.loads(SEEN_PATH.read_text()))
    return set()


def save_seen(seen: set[str]) -> None:
    SEEN_PATH.parent.mkdir(parents=True, exist_ok=True)
    SEEN_PATH.write_text(json.dumps(sorted(seen)))


def run(adapters, surface_map_root: Path, interval_s: int = 30):
    by_vendor = load_surface_map(surface_map_root)
    seen = load_seen()
    start_http_server(9108)
    while True:
        for adapter in adapters:
            vendor = adapter.vendor
            with FETCH_HIST.labels(vendor=vendor).time():
                try:
                    if isinstance(adapter, StatuspageAdapter):
                        incidents = adapter.fetch_open() + adapter.fetch_recent()
                        events = list(adapter.to_events(incidents))
                    else:
                        events = list(adapter.to_events())
                    INGESTER_POLL.labels(vendor=vendor, outcome="ok").inc()
                    INGESTER_LAG.labels(vendor=vendor).set(0)
                except Exception as e:
                    log.exception("fetch failed for %s: %s", vendor, e)
                    INGESTER_POLL.labels(vendor=vendor, outcome="error").inc()
                    continue
            for event in events:
                if event.event_id in seen:
                    continue
                enriched = enrich(event, by_vendor)
                try:
                    dispatch(enriched)
                    seen.add(event.event_id)
                    INGESTER_EVT.labels(
                        vendor=vendor,
                        impact=enriched.impact,
                        affects_tier1=str(enriched.affects_tier1),
                    ).inc()
                except Exception as e:
                    log.exception("dispatch failed for %s: %s", event.event_id, e)
            save_seen(seen)
        # bump lag gauges for any vendor not polled this cycle
        for vendor in {a.vendor for a in adapters}:
            INGESTER_LAG.labels(vendor=vendor).inc(interval_s)
        time.sleep(interval_s)


if __name__ == "__main__":
    # Wire adapters per the surface map's vendor keys.
    adapters = [
        StatuspageAdapter(vendor="openai",    base_url="https://status.openai.com"),
        StatuspageAdapter(vendor="anthropic", base_url="https://status.anthropic.com"),
        AtomAdapter(vendor="google",          feed_url="https://status.cloud.google.com/en/feed.atom"),
    ]
    run(adapters, surface_map_root=Path(os.environ["SURFACE_MAP_ROOT"]))
```

### 2.7 Config and deployment

- **Where it runs.** A small always-on service (one replica is fine
  for polling-only; add a second in a different region for webhook
  durability). Deploy away from the same vendor's compute if possible
  so you do not lose your ingester when the vendor region goes dark.
- **Backoff.** On vendor-side 429 or 5xx, exponential backoff with
  jitter, capped at the vendor's documented rate limit. Statuspage
  public endpoints tolerate low-rate polling; do not go below 15 s
  per host.
- **Secrets.** No vendor-side secrets needed for public status
  endpoints; only the incident-pipeline routing keys are sensitive.
  Store them in the same secrets manager referenced by the surface
  map.
- **State.** Idempotency cache (`seen.json` or a small key-value
  store) survives restarts. If lost, dedup is still provided by the
  incident pipeline's `dedup_key`.

### 2.8 Alerting on the ingester itself

```yaml
# monitoring/vendor-status-ingester.rules.yaml
groups:
- name: vendor-status-ingester
  rules:
  - alert: VendorStatusIngesterStale
    expr: vendor_status_last_success_seconds > 300
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Vendor status ingester stale for {{ $labels.vendor }}"
      runbook: docs/runbooks/vendor-status-ingester.md

  - alert: VendorStatusIngesterErrorRate
    expr: |
      sum(rate(vendor_status_poll_total{outcome="error"}[10m]))
        / sum(rate(vendor_status_poll_total[10m])) > 0.2
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Vendor status ingester error rate >20% for 15m"
      runbook: docs/runbooks/vendor-status-ingester.md
```

The ingester not working looks exactly like no vendor having
incidents. The staleness alert is the only thing preventing that
silent failure.

## 3. Validation steps

### 3.1 Unit

- Feed the Statuspage adapter a fixture JSON copied from the real
  endpoint (saved to `tests/fixtures/openai-unresolved.json`). Assert
  the adapter emits one event per incident, with stable `event_id`
  across repeat runs of the same fixture.
- Feed the Atom adapter a fixture `feed.atom`. Assert keyword →
  impact mapping behaves.
- `enrich` test: mock surface map; event for a known vendor and
  component yields `affects_tier1=True` with the expected surfaces.
- Idempotency test: emit same event twice, assert `dispatch` is
  called once (seen-set).

### 3.2 Integration (local)

- Point the ingester at a real public status page in a sandbox
  environment with a non-production routing key. Verify an event
  appears in the incident pipeline within one poll interval.
- Roll back a seen-set entry, re-run, confirm duplicate dispatch is
  suppressed by the incident pipeline's `dedup_key` (ingester cache +
  downstream dedup form a belt-and-braces).

### 3.3 Drill

- In a sandbox environment, run a local webserver that mimics the
  Statuspage JSON format and flips a mocked incident through
  `investigating → identified → monitoring → resolved`. Confirm the
  incident-pipeline timeline shows a single incident with four
  updates, not four incidents.

### 3.4 Observability sanity

- Inspect `vendor_status_last_success_seconds{vendor=...}` for each
  adapter; it must reset to zero at every successful poll.
- Inspect `vendor_status_events_total{affects_tier1="true"}` after a
  staged drill; it must increment.
- Trigger the staleness alert by pointing the Statuspage adapter at
  an unreachable host; confirm the alert fires in under its `for`
  duration + poll interval.

## 4. Rubric

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Coverage** | Only one provider format supported | Statuspage-style only | Statuspage + feed + webhook path documented or implemented |
| **Normalization** | Downstream sees provider-specific fields | One schema but ad-hoc | Explicit `VendorImpactEvent` schema with versioning |
| **Correlation** | Dispatches every vendor update | Tags vendor only | Tags `affects_tier1` and `affected_surfaces` by looking up ex-01 surface map |
| **Idempotency** | Duplicate dispatches on restart | Local cache but no dedup key downstream | Both: deterministic `event_id` + incident-pipeline `dedup_key` |
| **Severity routing** | Everything pages, or nothing pages | Pages on all tier-1 vendors flat | Severity-aware + tier-1 routing; FYI channel for non-tier-1 |
| **Self-observability** | None | Logs | Metrics + staleness alert + error-rate alert with runbook |
| **Backoff** | No backoff on vendor 429/5xx | Fixed backoff | Exponential + jitter, respects vendor rate limit |
| **Tests** | None | Unit for one adapter | Unit for all adapters + correlation + idempotency + drill script |
| **Runbook** | Absent | Short "what it does" | Troubleshooting for stale, error-rate, and false-page scenarios |

Passing: average ≥ 1.5 with no 0s on **Normalization**, **Correlation**,
**Idempotency**, **Self-observability**.

## 5. Common mistakes

- **Treating every status-page update as a new incident.** The
  Statuspage API returns the same incident across polls with
  mutating `incident_updates`. Keying `event_id` on `updated_at` is
  intentional — one event per update — but `dedup_key` must be keyed
  on `vendor_incident_id` so the incident pipeline threads them into
  one incident.
- **Paging on `maintenance`.** Maintenance windows will wake the team
  at 3am. Filter them in the adapter (we treat `impact=maintenance`
  as `none`) unless the team explicitly wants scheduled windows in
  incident timelines.
- **No staleness alert.** If the ingester dies, the on-call does not
  know. The `last_success_seconds` gauge is the cheapest way to make
  that visible.
- **Ingester hosted next to the provider.** Running the OpenAI
  adapter in the same cloud region you also page about means a
  regional outage can take out both the adapter and the surface it
  monitors. Deploy elsewhere.
- **Matching on free text.** Correlating via `components` is reliable;
  matching the incident `title` with regex is not. Keep
  `components_watched` in the surface map and match on it.
- **Credentialing public endpoints.** Statuspage public endpoints
  need no auth; adding a bearer token leaks credentials to the
  vendor's CDN and will get your IP rate-limited faster than
  no-auth polling.
- **Falling back to polling without upper-bounding frequency.**
  Lost webhook + aggressive polling = your IP in the vendor's
  rate-limit list. Cap polling at 15–30 s per host.
- **Losing the body text.** The vendor's last-update body is the
  single most valuable piece of context for the on-call; strip only
  HTML, not content.

## 6. References

- **Atlassian Statuspage developer documentation** —
  <https://developer.statuspage.io/> for the public API shapes
  (`incidents`, `incidents/unresolved`, webhooks). Field names used
  in the adapter (`id`, `name`, `status`, `impact`, `components`,
  `incident_updates`, `started_at`, `updated_at`, `resolved_at`,
  `shortlink`) come from there — verify against the live API when
  implementing.
- **OpenAI Status** — <https://status.openai.com/>.
  **Anthropic Status** — <https://status.anthropic.com/>.
  **Google Cloud Status** — <https://status.cloud.google.com/> (RSS
  at `/en/feed.atom`). These are the three endpoints the worked
  example wires.
- **PagerDuty Events API v2** —
  <https://developer.pagerduty.com/docs/events-api-v2/trigger-events/>
  for `routing_key`, `event_action`, `dedup_key` semantics. incident.io,
  Opsgenie, and Grafana Incident expose equivalent primitives — the
  dispatch adapter is portable across them.
- **NIST AI Risk Management Framework** —
  <https://www.nist.gov/itl/ai-risk-management-framework>. `MANAGE
  4.1` (incident response procedures) and `MEASURE 2.6` (monitoring
  of third-party components) are the subcategories this pipeline is
  evidence for.
- **OWASP Machine Learning Security Top 10** —
  <https://owasp.org/www-project-machine-learning-security-top-10/>.
  `ML06 AI Supply Chain Attacks` motivates treating provider status
  as a monitored input, not an assumed-available service.
- **MITRE ATLAS** — <https://atlas.mitre.org/>. The ingester is a
  detection surface for `AML.T0010 ML Supply Chain Compromise`-style
  disruption events.
- **Upstream: exercise-01 vendor surface map** (correlation),
  **exercise-02 fallback contract** (the ingester's `affects_tier1`
  tag feeds the automatic cutover decision),
  **exercise-03 case study** (every event here is one row in a
  future case study's Timeline).
