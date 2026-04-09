# Beacon Platform — System Context Overview

> This file describes how beacon-core fits into the wider beacon-* ecosystem.

---

## What Is the Beacon Platform?

Beacon is a city-scale emergency response platform. It connects citizens reporting incidents, ground-staff responders in the field, emergency response team (ERT) commanders, and city administrators into a single coordinated workflow.

---

## Where beacon-core Sits

**beacon-core is the system of record for all active emergency operations.** It is the authoritative source of truth for incidents, dispatches, evacuations, and field documents. Every other service either feeds data into it or reacts to events it emits.

```
Citizens / Mobile App
        │
        │  report incident / SOS
        ▼
  beacon-core  ──────────────────► beacon-event-bus (GCP Pub/Sub)
   (this service)                         │
        │                                 ├──► beacon-notification  (push alerts)
        │  calls at runtime               ├──► beacon-dashboard     (live ops view)
        ├──► beacon-iam                   └──► beacon-scheduler     (timed reminders)
        │     (auth + staff roster
        │      + FCM push tokens)
        │
        └──► beacon-observability
              (emergency station
               locations + capacities)
```

---

## Key Dependencies

### beacon-iam

beacon-core cannot function without beacon-iam. It relies on it for three distinct things:

1. **Authentication** — every inbound HTTP request carries a JWT that beacon-core validates against beacon-iam's public keys.
2. **Staff roster** — when a dispatcher creates a new dispatch assignment, beacon-core queries beacon-iam for available ground-staff members at the relevant station.
3. **Push-notification recipients** — when publishing lifecycle events (incident approved, dispatch accepted, evacuation advisory), beacon-core fetches active FCM device tokens from beacon-iam to include in the event payload.

If beacon-iam is unavailable, incident creation and dispatch creation will fail. FCM token resolution fails open (events are published without recipients), but the dispatch workflow is hard-blocked.

### beacon-observability

beacon-core calls beacon-observability to retrieve the geographic coordinates and metadata of emergency stations. This is used by the "nearest stations" feature — when a commander wants to dispatch a team to an incident, beacon-core queries observability for all stations, computes distances, and returns a ranked list. Without beacon-observability, the nearest-stations endpoint returns an error and dispatchers must locate stations manually.

---

## What beacon-core Emits

beacon-core publishes domain events to **beacon-event-bus** (Google Cloud Pub/Sub). Downstream consumers include:

| Event | Typical consumers |
|---|---|
| `incident.created`, `incident.approved` | beacon-notification (citizen push alert), beacon-dashboard |
| `incident.escalated`, `incident.closed` | beacon-notification, beacon-dashboard |
| `dispatch.created`, `dispatch.accepted`, `dispatch.completed` | beacon-dashboard, beacon-scheduler |
| `evacuation.advisory` | beacon-notification (broadcast to all citizens) |

---

## Data Stores Owned by beacon-core

| Store | Technology | What lives there |
|---|---|---|
| Primary DB | PostgreSQL (master + slave) | Incidents, dispatches, evacuations, vault metadata |
| Cache | Redis | Hot incident records (1-hour TTL) |
| Analytics | ClickHouse | Staff GPS location tracks, incident analytics aggregates |
| File storage | Google Cloud Storage (`beacon-vault` bucket) | Incident evidence files and documents |
| Audit trail | ClickHouse (`beacon_audit` DB, via beacon-auditing) | Immutable record of every state mutation |

---

*Generated as part of the beacon documentation workflow — 2026-04-09.*
