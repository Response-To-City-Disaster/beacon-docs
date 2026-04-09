# beacon-core

> Operational core of the Beacon emergency-response platform. Manages the complete lifecycle of city-scale emergencies — from the moment a citizen reports an incident, through the coordination of field responders, to the formal closure of the event — for the Response to City Disaster project.

[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go)](https://go.dev)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![GKE](https://img.shields.io/badge/Deployed_on-GKE_europe--west2-4285F4?logo=google-cloud)](https://cloud.google.com/kubernetes-engine)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Domains](#domains)
- [Service Dependencies](#service-dependencies)
- [Prerequisites](#prerequisites)
- [Local Development](#local-development)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Testing](#testing)
- [Docker](#docker)
- [Kubernetes / GKE Deployment](#kubernetes--gke-deployment)
- [Database Migrations](#database-migrations)
- [Project Structure](#project-structure)

---

## Overview

`beacon-core` is a Go microservice deployed on GKE (europe-west2). It acts as the **system of record** for all active emergency operations on the Beacon platform — meaning it is the single authoritative source of truth for what is happening in the field at any given moment. Every other service in the beacon ecosystem either writes data into it, reads from it, or subscribes to the events it emits.

The service is responsible for the full operational picture of an emergency:

| Domain | What it does |
|---|---|
| **Incidents** | Records and manages reported emergencies — fires, floods, accidents, SOS alerts, and more. Each incident moves through a controlled state machine (reported → approved → active → closed) with a full audit trail of every change. Incidents must be approved by staff before any further action can be taken. |
| **Dispatches** | Handles the assignment of ground-staff response teams to an approved incident. Tracks whether a responder accepts, rejects, or completes the assignment, and pushes status updates to connected clients in real time over Server-Sent Events. Dispatches that are neither accepted nor rejected within the configured timeout window are automatically recovered as timed out on the next server start. |
| **Evacuations** | Lets commanders define structured evacuation zones tied to a specific incident. Once an evacuation is active, an advisory message can be broadcast to all citizens on the platform via the event bus. |
| **Vault** | A GCS-backed document store that allows incident-related files — photos, PDFs, operational notes — to be uploaded directly to Google Cloud Storage and linked to specific incidents. Supports folders, tags, starring, soft-delete with a recycle bin, and bulk operations. |
| **Location Tracking** | Ingests batched GPS position updates from ground-staff devices during an active dispatch and stores them in ClickHouse. The resulting location track can be replayed post-incident for after-action review. |
| **SOS** | A fast-path endpoint for mobile emergency alerts. Creates a critical-severity incident that is automatically pre-approved, bypassing the normal staff-review step so response can begin immediately. |
| **Map / Geocoding** | Exposes a location-search endpoint backed by the TomTom API, used by dispatchers and the dashboard to find and validate incident locations by address. |

---

## Architecture

The service is structured around **Hexagonal Architecture (Ports & Adapters)** combined with **CQRS** (Command Query Responsibility Segregation). These two patterns together mean that business rules live in a pure, dependency-free domain layer, and all infrastructure concerns — databases, caches, HTTP clients, message queues — are plugged in from the outside as adapters.

```
┌──────────────────────────────────────────────────────────────┐
│  HTTP Layer                                                   │
│  Gorilla Mux router, JWT authentication middleware, RBAC,    │
│  request logging, CORS, OpenTelemetry tracing, Swagger UI    │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Registry Layer  (CQRS)                                       │
│  Command handlers — validate input, call domain, return      │
│  Query handlers  — call domain read path, shape response     │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Domain Layer                                                 │
│  Pure business logic. State-machine transitions, pre-        │
│  condition checks, event recording, audit logging calls.     │
│  Zero external package imports — all dependencies are        │
│  injected as interfaces.                                     │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Adapters Layer                                               │
│  ├── PostgreSQL master   (all writes)                        │
│  ├── PostgreSQL slave    (all reads — replica)               │
│  ├── Redis               (incident hot cache, 1 h TTL)       │
│  ├── ClickHouse          (location track, analytics, audit)  │
│  ├── GCS                 (vault file storage)                │
│  ├── GCP Pub/Sub         (async lifecycle event publishing)  │
│  └── HTTP clients        (beacon-iam, beacon-observability,  │
│                           TomTom geocoding API)              │
└──────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

- **Domain purity.** Nothing in `internal/domain/` imports an external package. Every external dependency (database, cache, HTTP client, event publisher) is expressed as a Go interface and injected at startup. This makes the domain logic independently testable without any infrastructure running.

- **CQRS read/write split.** All state-mutating commands write exclusively to the PostgreSQL master. All queries read exclusively from the slave replica. This enforces the separation at the code level, not just the configuration level, and prevents accidental writes from read paths.

- **Single composition root.** Every adapter, service, and handler is wired together in `cmd/server/main.go`. There is no dependency-injection framework or magic — it is plain Go. If you want to understand how any two parts of the system connect, that file is the definitive answer.

- **Chain of Responsibility for list operations.** Incident list filtering, sorting, and cursor-based pagination are each implemented as a separate chain link that can be composed independently. This follows the AIP-132 (filtering) and AIP-158 (pagination) API design guidelines.

- **Fail-open on non-critical paths.** Push-notification token resolution (via beacon-iam) and audit logging (via beacon-auditing) are designed to degrade gracefully. If either service is unavailable, the primary operation completes successfully and a warning is logged. This prevents infrastructure dependencies from blocking emergency response workflows.

---

## Domains

### Incident Lifecycle

An incident is the central entity in the system. Everything else — dispatches, evacuations, vault files, location tracks — is attached to an incident. The lifecycle enforces a deliberate approval gate to prevent unverified citizen reports from immediately triggering field operations.

```
reported ──(approve)──► active ──(escalate)──► active (higher severity)
                          │
                        (close)
                          │    blocked if any dispatch is in_progress
                          │    blocked if any evacuation is planned/active
                          ▼
                        closed
```

Every transition — creation, approval, update, escalation, closure — writes an immutable record to the `incident_events` table. This event log is the full audit timeline for an incident and is returned by `GET /v1/incidents/{id}/events`. Events carry actor metadata (who made the change, their role) and a structured diff of what changed.

SOS incidents are created with `approved: true` and `severity: critical` by the service itself, bypassing the approval step entirely.

### Dispatch Lifecycle

A dispatch is a formal assignment of a specific station's ground staff to an approved incident. The dispatcher picks a station (typically using the nearest-stations endpoint), creates the dispatch, and the assigned team sees it in their mobile app and must accept or reject it within `dispatch.timeout_minutes` (default: 2 minutes).

```
pending_acceptance
  ├──(accept)──► in_progress ──(complete)──► completed
  ├──(reject)──► rejected
  ├──(cancel)──► cancelled
  └──(timeout)─► timed_out   ← any dispatches still pending at startup are recovered here
```

While a dispatch is `in_progress`, the ground staff device streams GPS coordinates to `/v1/dispatches/{id}/location-updates`. Any client (dashboard, supervisor) that is connected to the SSE endpoint `GET /v1/dispatches/{id}/events` receives push notifications for every status transition in real time, without polling.

### Evacuation Lifecycle

An evacuation zone is attached to an approved incident and defines a geographic area that must be cleared. It can be updated as the situation evolves and cancelled if no longer needed.

```
planned ──(activate)──► active ──(complete)──► completed
  └──(cancel)──► cancelled
```

The `send-advisory` action triggers a broadcast to all active citizens on the platform. Internally, beacon-core fetches all active FCM tokens from beacon-iam and publishes an `evacuation.advisory` event to the event bus, which beacon-notification converts into push notifications.

### Vault

Files are uploaded using a two-phase approach to avoid routing large binary payloads through the API server:

1. `POST /v1/vault/files/upload` — client describes the file and receives a GCS signed URL.
2. Client uploads the file body directly to GCS using the signed URL.
3. `POST /v1/vault/files/{id}/confirm` — client notifies beacon-core that the upload is complete, making the file visible in the system.

Deleted files are moved to a recycle bin (`soft-delete`) and can be restored. Permanent deletion removes the record and the GCS object.

---

## Service Dependencies

beacon-core calls out to several other services at runtime. Understanding these dependencies matters when running locally or debugging production issues.

| Service | Transport | Why beacon-core needs it |
|---|---|---|
| **beacon-iam** | HTTP REST | Three distinct uses: (1) validates the JWT on every inbound request; (2) fetches the ground-staff roster for a given station when creating a dispatch; (3) fetches active mobile FCM tokens when publishing push-notification events. IAM is a hard dependency — if it is unreachable, incident creation and dispatch creation will fail. FCM token resolution fails open. |
| **beacon-observability** | HTTP REST | Provides the geographic coordinates and metadata for all emergency stations (fire stations, hospitals, etc.). Used exclusively by the nearest-stations and station-availability dispatch endpoints to rank stations by distance from an incident location. |
| **beacon-event-bus** | GCP Pub/Sub | Receives asynchronous lifecycle events for every significant state change: `incident.created`, `incident.approved`, `incident.escalated`, `incident.closed`, `dispatch.created`, `dispatch.accepted`, `dispatch.completed`, `evacuation.advisory`, and others. Downstream consumers include beacon-notification (push alerts) and beacon-dashboard (live updates). The event bus is optional — set `event_bus.enabled: false` to disable it locally. |
| **beacon-auditing** | ClickHouse TCP | Writes a structured audit record for every mutating operation (who, what, when, resource ID). Configured separately from the main ClickHouse instance. Fails open — if the auditing client cannot connect at startup, the service continues without audit logging and logs an error. |
| **Google Cloud Storage** | GCS SDK | All vault files are stored in the `beacon-vault` bucket under GCP project `beacon-477817`. The service account email configured in `storage.service_account_email` must have `roles/storage.objectAdmin` on the bucket. |
| **TomTom API** | HTTP REST | Handles forward geocoding for the `/v1/map/search` endpoint. Requires a valid API key in `tomtom.api_key`. Requests time out after `tomtom.timeout` (default 10 s). |

---

## Prerequisites

| Tool | Minimum version | Purpose |
|---|---|---|
| Go | 1.24 | Build and run the service |
| PostgreSQL | 12 | Primary datastore for incidents, dispatches, evacuations, vault metadata |
| Redis | 6 | In-memory cache for incident hot reads (TTL 1 hour) |
| ClickHouse | 22 | Location tracking timeseries and incident analytics aggregates |
| Docker | 20.10 | Container image builds and running dependencies locally |
| `golang-migrate` | latest | Apply and roll back database schema migrations |
| `swag` | latest | Regenerate the Swagger/OpenAPI documentation from source annotations |
| `golangci-lint` | latest | Static analysis and linting |

Install the Go tooling (`swag`, `golangci-lint`) in one step:

```bash
make install-tools
```

ClickHouse and the GCS/IAM/event-bus integrations are not required for basic local development — they can be disabled in `config.yaml` (see [Configuration](#configuration)).

---

## Local Development

### 1. Clone

```bash
git clone https://github.com/Response-To-City-Disaster/beacon-core.git
cd beacon-core
```

### 2. Start infrastructure dependencies

The service requires PostgreSQL and Redis at minimum. ClickHouse is only needed if you are working on location tracking or analytics.

```bash
# PostgreSQL — primary datastore
docker run -d --name beacon-pg \
  -e POSTGRES_USER=core \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=core \
  -p 5432:5432 \
  postgres:15-alpine

# Redis — incident cache
docker run -d --name beacon-redis \
  -p 6379:6379 \
  redis:7-alpine

# ClickHouse — only needed for location tracking and analytics
docker run -d --name beacon-ch \
  -e CLICKHOUSE_USER=myuser \
  -e CLICKHOUSE_PASSWORD=MyStrongPassword123 \
  -p 9000:9000 \
  clickhouse/clickhouse-server:latest
```

### 3. Run database migrations

This applies all pending SQL migrations to your local PostgreSQL instance:

```bash
make migrate-up
```

The migration tool defaults to `postgresql://postgres:secret@localhost:5432/core?sslmode=disable`. Override with the `DB_URL` environment variable if your local credentials differ.

### 4. Configure

The service reads `config.yaml` from the working directory at startup. Copy the default and edit it for your local environment:

```bash
cp config.yaml config.local.yaml
```

At minimum, update:
- `database.master.host` and `database.slave.host` → `localhost`
- `redis.host` → `localhost`
- `clickhouse.host` → `localhost` (or remove if not using it)
- `event_bus.enabled` → `false` (unless you have GCP credentials configured)
- `auditing` block → point to your local ClickHouse or leave the host empty to disable

Then tell the service to use your local config:

```bash
CONFIG_PATH=config.local.yaml make run
```

### 5. Run

```bash
make run
```

The HTTP server starts on `http://localhost:8080`. You should see `"server starting"` in the logs followed by confirmation that each dependency connected successfully.

### 6. Swagger UI

The Swagger UI is available at `http://localhost:8080/swagger/index.html` and documents every endpoint including request/response schemas. Regenerate it after changing handler annotations:

```bash
make swagger
```

### Hot reload

If you have [`air`](https://github.com/air-verse/air) installed, the service will automatically restart on file changes:

```bash
make dev   # uses air if found, falls back to go run
```

---

## Configuration

Configuration is loaded from `config.yaml` using [Viper](https://github.com/spf13/viper). Every key can be overridden at runtime by an environment variable. The override convention is: uppercase the key, replace `.` with `_`, and prefix with `BEACON_`. For example, `database.master.host` becomes `BEACON_DATABASE_MASTER_HOST`.

This means you never need to modify `config.yaml` in deployed environments — all secrets and environment-specific values are injected via environment variables or Kubernetes Secrets (see [Kubernetes / GKE Deployment](#kubernetes--gke-deployment)).

```yaml
server:
  port: 8080        # Port the HTTP server listens on
  timeout: 30s      # Read, write, and idle timeout for HTTP connections

database:
  master:           # Handles all writes (INSERT, UPDATE, DELETE)
    host: ""
    port: 5432
    user: ""
    password: ""
    dbname: "core"
    max_open_conns: 25   # Maximum concurrent connections to the master
    max_idle_conns: 5    # Connections held open when idle
  slave:            # Handles all reads (SELECT) — replica of master
    host: ""
    port: 5432
    user: ""
    password: ""
    dbname: "core"
    max_open_conns: 50   # Higher limit — reads are more frequent than writes
    max_idle_conns: 10

redis:
  host: ""
  port: 6379
  password: ""
  db: 0
  ttl: 3600         # How long incident records are cached, in seconds (default: 1 hour)

clickhouse:
  host: ""
  port: 9000
  username: ""
  password: ""
  dbname: "core"    # Stores location track data and analytics aggregates

storage:
  bucket_name: ""              # GCS bucket for vault files
  project_id: ""               # GCP project that owns the bucket
  service_account_email: ""    # Service account with storage.objectAdmin on the bucket

iam:
  url: ""               # Base URL of the beacon-iam service
  service_username: ""  # Machine account credentials beacon-core uses to call IAM
  service_password: ""

notification:
  url: ""   # Base URL of beacon-notification (used for direct notification calls if any)

observability:
  url: ""         # Base URL of beacon-observability
  country_id: 0   # Country scope for station queries

dispatch:
  timeout_minutes: 2   # How long a dispatch can remain pending before it times out

tomtom:
  api_key: ""    # TomTom developer API key
  timeout: 10s   # Maximum time to wait for a TomTom geocoding response

event_bus:
  project_id: ""        # GCP project hosting the Pub/Sub topics
  credentials_file: ""  # Path to a service account JSON key. Leave empty to use
                        # Application Default Credentials (recommended on GKE)
  enabled: true         # Set to false to disable all event publishing (useful locally)

auditing:
  host: ""
  port: 9000
  database: "beacon_audit"   # ClickHouse database for the audit trail
  username: ""
  password: ""
  secure: false   # Set to true if the ClickHouse instance uses TLS

logging:
  level: "info"      # Verbosity: debug | info | warn | error
  format: "console"  # Output format: console (human-readable) | json (structured, for production)
```

---

## API Reference

All application endpoints are prefixed with `/v1`. The two health endpoints (`/health`, `/ready`) are exempt from authentication and are used by Kubernetes probes.

Every other request must carry a valid JWT in the `Authorization: Bearer <token>` header. The token is validated against beacon-iam on every request — there is no local token cache. The JWT claims determine the caller's role, which controls what they are allowed to do.

### Role matrix

Roles are hierarchical in practice, though they are enforced as discrete checks at each route:

| Role | What they can do |
|---|---|
| `citizen` | Report incidents, submit SOS alerts, and read any publicly accessible data (incidents, evacuations, vault files they have access to). Cannot approve, close, dispatch, or escalate. |
| `ground_staff` | Everything a citizen can do, plus accept or reject a dispatch assigned to them, and submit GPS location updates during an active dispatch. |
| `ert_member` | Everything a ground_staff can do, plus create and cancel dispatches, create and manage evacuations, approve incidents, close incidents, escalate incidents, and record incident updates. |
| `admin` | Full access to all operations across all domains. |

---

### Health

These endpoints are intentionally lightweight and require no authentication. They are used by Kubernetes liveness and readiness probes to determine whether to route traffic to a pod.

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Returns `200 OK` if the process is alive. Used as a liveness probe — if this fails, the pod is restarted. |
| `GET` | `/ready` | Returns `200 OK` if the service has finished initializing and is ready to handle requests. Used as a readiness probe — Kubernetes will not route traffic to a pod until this passes. |

---

### Incidents

The incident is the primary entity in beacon-core. All dispatch and evacuation activity is attached to an incident. An incident cannot receive dispatches or evacuations until it has been approved by a staff member.

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/incidents` | citizen, ground_staff, ert_member, admin | Creates a new incident in `reported` state. Requires title, description, type, severity, location coordinates, and the reporting user's ID. The incident is not visible to field operations until it is approved. |
| `GET` | `/v1/incidents` | all | Returns a paginated, filterable list of incidents. Supports filtering by status, severity, type, and approval state. Results are sorted and paginated using a cursor-based scheme — use `page_token` from the previous response to fetch the next page. See [List Filters](#list-filters) below. |
| `GET` | `/v1/incidents/{id}` | all | Returns the full details of a single incident by its UUID, including location, severity, status, all attached vault file references, and the dispatching metadata. Reads from the Redis cache first; falls back to the PostgreSQL slave on a cache miss. |
| `PUT` | `/v1/incidents/{id}` | ground_staff, ert_member, admin | Updates mutable fields on an existing incident (title, description, type, severity, location, affected radius). Validates that any status transition follows the permitted state machine. Records a diff of changed fields in the event log. |
| `POST` | `/v1/incidents/{id}/approve` | ground_staff, ert_member, admin | Marks an incident as approved, enabling dispatches and evacuations to be created against it. Also triggers an `incident.approved` event to the event bus, which notifies subscribed citizens via push notification. Only has effect once — re-approving an already-approved incident returns an error. |
| `POST` | `/v1/incidents/{id}/close` | ground_staff, ert_member, admin | Closes an incident. This is a terminal state — the incident cannot be reopened. The request is rejected if any dispatches are still `in_progress` or any evacuations are still `planned` or `active`. The caller must resolve those first. |
| `POST` | `/v1/incidents/{id}/escalate` | ground_staff, ert_member, admin | Raises the severity level of an active incident (e.g., from `high` to `critical`). Records the previous and new severity in the event log and publishes an `incident.escalated` event to alert subscribed citizens. Can only be applied to open incidents. |
| `POST` | `/v1/incidents/{id}/update` | ground_staff, ert_member, admin | Records a situational update on an incident (e.g., "fire contained to ground floor") without changing its status or severity. If `should_alert: true` is passed in the request body, the update message is published to the event bus and forwarded as a push notification to active citizens. |
| `GET` | `/v1/incidents/{id}/events` | all | Returns the paginated event history for an incident — a chronological log of every action taken: who did what, when, and what changed. Each event includes actor metadata and a structured field-level diff. This is the source of truth for the incident timeline shown in the dashboard. |
| `GET` | `/v1/incidents/metrics` | all | Returns aggregate counts of incidents broken down by status, severity, and type. Used by the dashboard summary cards to give commanders a snapshot of the current operational picture. |
| `GET` | `/v1/analytics` | all | Returns monthly trend data covering incident volumes, response times, and severity distributions over the past N months (default: 8). Used to populate the analytics charts in the dashboard. |

#### List filters

These query parameters apply to `GET /v1/incidents`:

| Parameter | Type | Description |
|---|---|---|
| `status` | string | Filter to incidents in a specific state: `reported`, `active`, or `closed`. |
| `severity` | string | Filter by severity: `low`, `medium`, `high`, or `critical`. |
| `type` | string | Filter by incident type: `fire`, `flood`, `accident`, `sos`, or other configured types. |
| `approved` | bool | `true` to show only approved incidents; `false` for pending-approval only. Omit to return all. |
| `is_internal` | bool | `true` to include internal-only incidents (not visible to citizens). Omit for default visibility. |
| `sort_by` | string | Field to sort by: `created_at` (default), `severity`, or `status`. |
| `sort_order` | string | `asc` or `desc`. Defaults to `desc` (newest first). |
| `page_size` | int | Number of results per page. Defaults to 20. |
| `page_token` | string | Opaque cursor returned in the previous response. Pass this to retrieve the next page. Omit to start from the beginning. |

---

### SOS

The SOS endpoint is a dedicated fast path for emergency alerts originating from mobile devices. It bypasses the normal incident creation and approval flow.

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/incidents/sos` | all authenticated users | Creates a new incident with type `sos`, severity `critical`, and `approved: true` in a single operation. The incident is immediately visible to dispatchers and eligible for dispatch without any staff review. If `has_location: true` is provided along with latitude/longitude, the coordinates are attached to the incident. An `incident.created` event is published immediately to notify all active response teams. |

---

### Dispatches

A dispatch is the mechanism by which an ERT member assigns a ground-staff team to respond to an approved incident. The dispatcher selects a station (using the nearest-stations endpoint), creates the dispatch, and the assigned team's mobile app receives an immediate push notification asking them to accept or reject.

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/dispatches` | ert_member, admin | Creates a new dispatch assignment linking an approved incident to a specific station and role (e.g., `fire_brigade`). Validates that the incident exists and is approved before creating. The dispatch starts in `pending_acceptance` state and a `dispatch.created` event is sent to the assigned team via the event bus. |
| `GET` | `/v1/dispatches` | ground_staff, ert_member, admin | Returns a list of all dispatches across all incidents, ordered by creation time. Useful for the operations dashboard to see the full field activity at a glance. |
| `GET` | `/v1/dispatches/{id}` | ground_staff, ert_member, admin | Returns the full details of a single dispatch including its current status, the assigned station, the incident it is linked to, timestamps for each transition, and any notes left by the dispatcher. |
| `POST` | `/v1/dispatches/{id}/accept` | ground_staff, ert_member, admin | The assigned responder confirms they are responding. Transitions the dispatch from `pending_acceptance` to `in_progress` and records an `accepted` event on the parent incident's timeline. Publishes a real-time SSE event to any connected listeners. |
| `POST` | `/v1/dispatches/{id}/reject` | ground_staff, ert_member, admin | The assigned responder declines the assignment (e.g., station is at capacity). Transitions to `rejected`. The dispatcher will typically create a new dispatch to a different station in response. Records a `dispatch_rejected` event on the parent incident — this event is not visible to citizens. |
| `POST` | `/v1/dispatches/{id}/complete` | ert_member, admin | Marks the dispatch as completed once the response team has finished their work at the scene. Transitions to `completed` and publishes a `dispatch.completed` event. This is a prerequisite for closing the parent incident if this was the last active dispatch. |
| `POST` | `/v1/dispatches/{id}/cancel` | ert_member, admin | Cancels a dispatch that is no longer needed before it has been completed. Can be applied to dispatches in `pending_acceptance` or `in_progress` state. |
| `GET` | `/v1/dispatches/{id}/events` | ground_staff, ert_member, admin | Opens a **Server-Sent Events stream** for real-time dispatch status updates. The connection stays open and the server pushes a message whenever the dispatch transitions to a new state. Clients should reconnect on disconnect using the `Last-Event-ID` header. Use `curl -N` to test from the terminal. |
| `GET` | `/v1/dispatches/{id}/location-track` | ert_member, admin | Returns the full GPS breadcrumb trail for a dispatch — the sequence of coordinates submitted by the ground-staff device during the response. Useful for after-action review and route analysis. Data is read from ClickHouse. |
| `POST` | `/v1/dispatches/{id}/location-updates` | ground_staff | Accepts a batch of GPS coordinates from a ground-staff device during an active dispatch. Coordinates are written to ClickHouse in bulk. The mobile app is expected to call this endpoint periodically (e.g., every 5–10 seconds) while the dispatch is `in_progress`. |
| `GET` | `/v1/incidents/{id}/dispatches` | all | Lists all dispatches ever created for a given incident, across all statuses. Useful to see the full history of response assignments for an incident, including those that were rejected or timed out. |
| `GET` | `/v1/incidents/{id}/stations/{affiliation_id}/availability` | ground_staff, ert_member, admin | Queries beacon-iam for the number of available staff members at a specific station for a given role. Used by the dispatch creation UI to show whether a station has capacity before assigning. |
| `GET` | `/v1/incidents/{id}/nearest-stations` | ground_staff, ert_member, admin | Fetches all emergency stations from beacon-observability, computes their distance from the incident's location, and returns a ranked list ordered by proximity. This is the primary tool a dispatcher uses to decide which station to assign. |

---

### Evacuations

An evacuation defines a zone that must be cleared in connection with an active incident. It can be created, updated as the perimeter changes, and eventually completed or cancelled. The advisory broadcast is the mechanism for notifying civilians in the affected area.

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/evacuations` | ert_member, admin | Creates a new evacuation zone linked to an approved incident. Requires zone boundaries, a description, and optionally a transport mode (e.g., `foot`, `vehicle`). The evacuation starts in `planned` state. |
| `GET` | `/v1/evacuations` | all | Returns a paginated list of all evacuations across the system, regardless of incident. Supports basic filtering by status. |
| `GET` | `/v1/evacuations/{id}` | all | Returns the full details of a single evacuation including its zone geometry, current status, the parent incident, and any advisory messages that have been sent. |
| `PUT` | `/v1/evacuations/{id}` | ert_member, admin | Updates the mutable properties of an evacuation — zone boundaries, description, transport mode, or advisory text — as the situation on the ground evolves. |
| `POST` | `/v1/evacuations/{id}/cancel` | ert_member, admin | Cancels an evacuation that is no longer required. Records the cancellation on the parent incident's event timeline. |
| `POST` | `/v1/evacuations/{id}/send-advisory` | ert_member, admin | Broadcasts a public safety advisory to all citizens on the platform. Internally, beacon-core fetches all active FCM device tokens from beacon-iam and publishes an `evacuation.advisory` event to the event bus, which beacon-notification converts into push notifications. The advisory message is recorded on the incident timeline. |
| `DELETE` | `/v1/evacuations/{id}` | ert_member, admin | Permanently removes an evacuation record. Should only be used to clean up erroneous entries — prefer `cancel` for operational withdrawals. |
| `GET` | `/v1/incidents/{id}/evacuations` | all | Lists all evacuations attached to a specific incident, ordered by creation time. Useful for getting the full picture of evacuation activity for an incident. |

---

### Vault

The Vault is an incident-scoped document store backed by Google Cloud Storage. It is organized as a folder hierarchy and supports all the file management operations a team needs during and after an incident: uploading evidence, annotating files with tags, archiving completed work, and permanently deleting sensitive material.

Uploads use a two-phase model so that large files (photos, video clips) never pass through beacon-core's memory — the client gets a signed URL and uploads directly to GCS.

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/vault/stats` | Returns aggregate storage statistics for the current user's vault: total files, total size, number of starred items, and number of items in the bin. |
| `GET` | `/v1/vault/root` | Lists the contents of the root folder — top-level folders and loose files that have not been placed in a subfolder. |
| `GET` | `/v1/vault/starred` | Returns all files that the current user has starred, regardless of which folder they are in. Useful for quick access to frequently referenced documents. |
| `GET` | `/v1/vault/bin` | Lists all soft-deleted files. Items in the bin can be restored or permanently deleted. |
| `POST` | `/v1/vault/folders` | Creates a new folder. Folders can be nested inside other folders by specifying a `parent_id`. |
| `GET` | `/v1/vault/folders/{id}` | Returns the contents of a specific folder — its child folders and the files directly inside it. |
| `POST` | `/v1/vault/files/upload` | Phase 1 of the upload flow. The client sends file metadata (name, MIME type, size) and receives back a GCS signed upload URL and a file ID. The actual file bytes are sent directly to the GCS URL, not through this API. |
| `POST` | `/v1/vault/files/{id}/confirm` | Phase 2 of the upload flow. Called after the client has successfully uploaded the file bytes to GCS. This makes the file visible in the vault and links it to the folder specified during upload initiation. |
| `GET` | `/v1/vault/files/{id}` | Returns the full metadata for a file: name, MIME type, size, folder, tags, star status, upload timestamp, and a signed GCS download URL valid for a limited time. |
| `PATCH` | `/v1/vault/files/{id}` | Updates editable metadata on an existing file, such as the display name or description. Does not affect the underlying GCS object. |
| `DELETE` | `/v1/vault/files/{id}` | Moves the file to the recycle bin (soft delete). The file is no longer returned in normal folder listings but can be restored from `/v1/vault/bin`. |
| `POST` | `/v1/vault/files/{id}/move` | Moves the file to a different folder within the vault. Requires the destination `folder_id` in the request body. |
| `POST` | `/v1/vault/files/{id}/star` | Stars a file, marking it as important. Starred files appear in `/v1/vault/starred`. |
| `DELETE` | `/v1/vault/files/{id}/star` | Removes the star from a file. |
| `POST` | `/v1/vault/files/{id}/restore` | Restores a soft-deleted file from the bin back to its original folder. If the folder no longer exists, the file is placed in the root. |
| `DELETE` | `/v1/vault/files/{id}/permanent` | Permanently deletes a file from the bin. This deletes the database record and the GCS object. Irreversible. |
| `GET` | `/v1/vault/files/{id}/tags` | Returns the list of tags applied to a file. Tags are free-form string labels and can be used to categorize evidence (e.g., `"photo"`, `"report"`, `"urgent"`). |
| `POST` | `/v1/vault/files/{id}/tags` | Adds a new tag to a file. |
| `DELETE` | `/v1/vault/files/{id}/tags/{tagId}` | Removes a specific tag from a file by its tag ID. |
| `POST` | `/v1/vault/bulk/delete` | Soft-deletes multiple files in a single request. Accepts an array of file IDs. |
| `POST` | `/v1/vault/bulk/move` | Moves multiple files to a target folder in a single request. |
| `POST` | `/v1/vault/bulk/star` | Stars multiple files in a single request. |
| `POST` | `/v1/vault/bulk/restore` | Restores multiple soft-deleted files from the bin in a single request. |
| `POST` | `/v1/incidents/{id}/attachments` | Links an existing vault file to a specific incident. The file must already exist in the vault. After linking, it appears in the incident's attachments list and is returned as part of `GET /v1/incidents/{id}`. |
| `GET` | `/v1/incidents/{id}/attachments` | Lists all vault files that have been explicitly attached to an incident. This is distinct from the incident's vault folder — it is a curated list of key evidence files. |
| `GET` | `/v1/incidents/{id}/vault-folder` | Returns the GCS folder dedicated to this incident, creating it automatically if it does not yet exist. All uploads associated with an incident should be placed here for organizational consistency. |

---

### Map

| Method | Path | Roles | Description |
|---|---|---|---|
| `GET` | `/v1/map/search?query=<address>` | all authenticated users | Geocodes a free-text address or place name using the TomTom API and returns a list of matching locations with their coordinates, formatted addresses, and confidence scores. Used by the incident reporting form and the dispatch dashboard to translate a human-readable address into coordinates for the incident record. |

---

### Example requests

```bash
# Report a new incident
curl -X POST https://beacon-tcd.tech/api/core/v1/incidents \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "fire",
    "severity": "high",
    "title": "Building fire on O Connell St",
    "description": "Smoke visible from upper floors, occupants evacuating",
    "location": { "latitude": 53.3498, "longitude": -6.2603 },
    "affected_radius": 0.3,
    "reported_by": "user-uuid"
  }'

# Approve the incident (staff member in control room)
curl -X POST https://beacon-tcd.tech/api/core/v1/incidents/<id>/approve \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Verified by control room — fire confirmed by camera feed" }'

# Find the nearest fire station to dispatch
curl "https://beacon-tcd.tech/api/core/v1/incidents/<id>/nearest-stations" \
  -H "Authorization: Bearer <token>"

# Create a dispatch to the chosen station
curl -X POST https://beacon-tcd.tech/api/core/v1/dispatches \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "incident_id": "<incident-id>",
    "affiliation_id": "<station-id>",
    "role": "fire_brigade",
    "notes": "Bring breathing apparatus, multiple floors affected"
  }'

# Stream live dispatch status updates (keep connection open)
curl -N -H "Authorization: Bearer <token>" \
  https://beacon-tcd.tech/api/core/v1/dispatches/<id>/events

# Report an SOS alert from a mobile device
curl -X POST https://beacon-tcd.tech/api/core/v1/incidents/sos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "latitude": 53.3498,
    "longitude": -6.2603,
    "has_location": true,
    "description": "Person trapped, need immediate assistance"
  }'

# Send a public evacuation advisory
curl -X POST https://beacon-tcd.tech/api/core/v1/evacuations/<id>/send-advisory \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "message": "Evacuate the area bounded by O Connell St and Abbey St immediately. Move north." }'
```

---

## Testing

### Unit tests

Unit tests cover domain logic — state machine transitions, validation rules, service method behaviour — using mock implementations of the repository and publisher interfaces. No infrastructure is required.

```bash
make test
# Runs with -race flag and generates coverage.out + coverage.html
```

View the coverage report in a browser:

```bash
go tool cover -html=coverage.out
```

### Run with race detector

```bash
go test -v -race ./...
```

### Integration tests

Integration tests verify CRUD operations against a real PostgreSQL database. They exercise the full stack from the service layer down through the Postgres adapters. They are skipped automatically in CI when `SKIP_INTEGRATION_TESTS=true` is set — use this if the database is not available in your pipeline.

```bash
# Against the cloud database (requires network access)
make test-crud-integration

# Against a local database
DB_HOST=localhost DB_PORT=5432 DB_NAME=core DB_USER=core DB_PASSWORD=secret \
  make test-crud-integration
```

---

## Docker

### Build

```bash
make docker-build
```

The `Dockerfile` uses a two-stage build. The builder stage uses `golang:1.24-alpine` to compile the binary with CGO disabled and dead-code stripping (`-ldflags="-w -s"`). The runtime stage uses `gcr.io/distroless/static:nonroot` — a minimal image with no shell, no package manager, and no OS libraries, running as a non-root user. The final image contains only the compiled binary and `config.yaml`.

This results in a small, hardened image with a minimal attack surface.

Private Go modules (other `Response-To-City-Disaster` packages) are fetched at build time using a GitHub token. Pass it as a build argument:

```bash
docker build --build-arg GITHUB_TOKEN=<token> -t beacon-core:latest .
```

In CI, this token is injected from the repository's secrets store and never written to disk.

### Run locally

```bash
make docker-run
# or with explicit env vars:
docker run --rm -p 8080:8080 \
  -e BEACON_DATABASE_MASTER_HOST=host.docker.internal \
  -e BEACON_DATABASE_SLAVE_HOST=host.docker.internal \
  -e BEACON_DATABASE_MASTER_USER=core \
  -e BEACON_DATABASE_MASTER_PASSWORD=secret \
  -e BEACON_DATABASE_MASTER_DBNAME=core \
  -e BEACON_EVENT_BUS_ENABLED=false \
  beacon-core:latest
```

Use `host.docker.internal` to reach services running on your host machine from inside the container.

---

## Kubernetes / GKE Deployment

The service runs in the `staging` and `production` namespaces on a GKE cluster in `europe-west2`. Container images are built with git SHA tags and pushed to GCP Artifact Registry. Kubernetes Secrets (managed outside of source control) inject all sensitive configuration values at pod startup.

### One-time setup

Before the first deploy, configure Docker authentication with Artifact Registry and create the Kubernetes Secret in the target namespace:

```bash
# Authenticate Docker with GCP Artifact Registry
make gcp-auth GCP_PROJECT_ID=beacon-477817

# Create the Kubernetes Secret with all required credentials
kubectl create secret generic beacon-core-secrets -n staging \
  --from-literal=db-master-host=<host> \
  --from-literal=db-slave-host=<host> \
  --from-literal=db-user=<user> \
  --from-literal=db-password=<password> \
  --from-literal=clickhouse-host=<host> \
  --from-literal=clickhouse-port=9000 \
  --from-literal=clickhouse-username=<user> \
  --from-literal=clickhouse-password=<password> \
  --from-literal=clickhouse-dbname=core
```

### Deploy

```bash
# Build the image, tag it with the current git SHA, and push to Artifact Registry
make gke-build-push GCP_PROJECT_ID=beacon-477817 GKE_CLUSTER_NAME=beacon-cluster

# Render the k8s/ manifests with the new image tag and apply them, then wait for rollout
make gke-deploy GCP_PROJECT_ID=beacon-477817 GKE_CLUSTER_NAME=beacon-cluster NAMESPACE=staging
```

The deploy target uses `kubectl rollout status` to block until all pods are healthy or the 5-minute timeout is exceeded.

### Day-to-day operations

```bash
make gke-logs                     # Tail live stdout/stderr logs from the deployment
make gke-pods                     # List pods and their current status
make gke-status                   # Show the service's external IP address
make gke-restart                  # Trigger a rolling restart (no downtime)
make gke-scale REPLICAS=3         # Scale the number of running pods
make gke-describe                 # Describe the deployment for debugging
```

### Kubernetes manifests

| File | Purpose |
|---|---|
| `k8s/deployment.yml` | Pod template with resource limits, health probes, and all env-var bindings from the Secret |
| `k8s/service.yml` | Kubernetes Service that exposes the pods to the cluster and load balancer |
| `k8s/secret.yml` | Secret manifest template — values are never committed; populated by CI at deploy time |
| `k8s/istio.yml` | Istio VirtualService for traffic routing and canary rollout policy |

**Resource limits per pod:**

| | Request | Limit |
|---|---|---|
| CPU | 20m | 50m |
| Memory | 75Mi | 100Mi |

The low memory footprint is intentional — the Go binary is small and the service does not hold large in-memory state (the SSE broker holds open channel handles, but each is lightweight).

**Health probes:**

| Probe | Endpoint | Initial delay | Period | Failure threshold |
|---|---|---|---|---|
| Liveness | `GET /health` | 30 s | 10 s | 3 consecutive failures → pod restart |
| Readiness | `GET /ready` | 10 s | 5 s | 3 consecutive failures → remove from load balancer |

The 30-second liveness initial delay gives the service time to establish its database and Redis connections before Kubernetes starts checking.

---

## Database Migrations

Schema migrations are managed with [`golang-migrate`](https://github.com/golang-migrate/migrate). Every change to the PostgreSQL schema lives in a numbered `.up.sql` / `.down.sql` pair under `migrations/`. Never modify an already-applied migration — always add a new numbered file.

```
migrations/
├── 000001_create_incidents_table           — core incidents table with location and status
├── 000002_add_approved_to_incidents        — adds the approval gate column
├── 000003_create_incident_events_table     — immutable event log for incident timeline
├── 000004_create_dispatches_table          — dispatch assignments and status tracking
├── 000005_create_dispatch_members_table    — junction table for staff members per dispatch
├── 000006_create_evacuations_table         — evacuation zones linked to incidents
├── 000007_create_vault_tables              — vault file and folder metadata
├── 000008_add_event_metadata               — structured actor/entity metadata on events
├── 000009_fix_leaked_dispatch_members      — data fix for orphaned dispatch member rows
├── 000009_extend_evacuations_advisory      — adds advisory text fields to evacuations
├── 000010_add_transport_mode_to_evacuations — transport mode column for evacuation zones
├── 000011_simplify_evacuations             — schema cleanup and normalization
├── 000012_add_is_internal_to_incidents     — flag for internal-only (non-citizen-visible) incidents
└── clickhouse/001_staff_location_updates   — ClickHouse table for GPS location track ingestion
```

```bash
# Apply all pending migrations (local)
make migrate-up

# Roll back the most recent migration (local)
make migrate-down

# Apply migrations against the cloud database
make migrate-up-cloud

# Roll back against the cloud database
make migrate-down-cloud
```

The `cmd/migrate` binary is a standalone runner used in CI pipelines to apply migrations as part of the deployment process, before the new application version starts.

---

## Project Structure

```
beacon-core/
├── cmd/
│   ├── server/main.go        # Composition root. Reads config, initialises all
│   │                         # dependencies, wires adapters → services → handlers,
│   │                         # starts the HTTP server, and handles graceful shutdown.
│   └── migrate/main.go       # Standalone migration runner used in CI pipelines.
│
├── internal/
│   ├── domain/               # The heart of the service. Pure Go — no external
│   │   │                     # package imports. All infra is injected as interfaces.
│   │   ├── incident/         # Incident state machine, approval gate, event recording,
│   │   │                     # cache management, audit calls, analytics.
│   │   ├── dispatch/         # Dispatch lifecycle, staff assignment, SSE event
│   │   │                     # publishing, timeout tracking, crash recovery.
│   │   ├── evacuation/       # Evacuation zone management, advisory broadcast logic.
│   │   ├── location/         # GPS breadcrumb ingestion and track query service.
│   │   ├── vault/            # Vault domain model and GCS interaction interfaces.
│   │   ├── route/            # Route domain (future).
│   │   └── mapfeature/       # Map feature domain (future).
│   │
│   ├── http/
│   │   ├── router.go         # Defines all routes, applies RBAC middleware
│   │   │                     # sub-routers per role group. The authoritative
│   │   │                     # list of every endpoint in the service.
│   │   ├── middleware.go     # Logging, CORS, OpenTelemetry tracing, and the
│   │   │                     # JWT authentication middleware.
│   │   └── handlers/         # One file per domain. Translates HTTP requests into
│   │                         # CQRS command/query calls and formats responses.
│   │
│   ├── registry/
│   │   ├── commands/         # One handler struct per write operation. Validates
│   │   │                     # input, calls the domain service, returns the result.
│   │   └── queries/          # One handler struct per read operation. Always reads
│   │                         # from the slave repo or cache.
│   │
│   ├── adapters/
│   │   ├── database/         # PostgreSQL master and slave repository implementations
│   │   │                     # for incidents, dispatches, and evacuations.
│   │   ├── db/postgres/      # Vault repository (separate adapter package).
│   │   ├── clickhouse/       # Location track repository backed by ClickHouse.
│   │   ├── eventbus/         # GCP Pub/Sub publisher adapters — one per domain,
│   │   │                     # maps domain structs to event bus message payloads.
│   │   ├── microservices/    # HTTP clients for beacon-iam, beacon-observability,
│   │   │                     # and TomTom. Each implements a domain interface.
│   │   └── storage/gcs/      # Google Cloud Storage adapter for vault file operations.
│   │
│   └── sse/
│       └── broker.go         # In-memory pub/sub broker for dispatch SSE events.
│                             # Keyed by dispatch ID. Not distributed — single instance only.
│
├── pkg/                      # Shared packages used across the service.
│   ├── auth/                 # JWT parsing, user claim extraction, role constants,
│   │                         # and the HTTP authentication middleware.
│   ├── config/               # Viper-based config loader with struct binding.
│   ├── errors/               # Typed error constructors (BadRequest, NotFound,
│   │                         # Conflict, InvalidTransition, etc.).
│   ├── logger/               # Zerolog-based structured logger wrapper.
│   ├── postgres/             # Opens and validates the master/slave connection pools.
│   ├── redis/                # Redis client initialisation with TTL configuration.
│   ├── clickhouse/           # ClickHouse TCP connection setup.
│   ├── tracing/              # OpenTelemetry trace context middleware.
│   └── response/             # Standardised HTTP JSON response helpers (Success,
│                             # Created, Error, etc.) used by all handlers.
│
├── migrations/               # golang-migrate SQL files. One numbered pair per change.
├── k8s/                      # Kubernetes deployment, service, secret, and Istio manifests.
├── docs/                     # Auto-generated Swagger/OpenAPI documentation (do not edit).
├── tests/
│   ├── unit/                 # Unit tests for domain logic using mock repositories.
│   └── integration/          # Integration tests requiring a live PostgreSQL database.
│
├── Dockerfile                # Two-stage build: alpine builder → distroless runtime.
├── Makefile                  # All developer, test, build, and deployment targets.
└── config.yaml               # Default configuration template. No secrets committed here.
```

---

## Contributing

1. Branch from `main`: `git checkout -b feature/<short-description>`
2. Run `make check` before opening a PR — this runs `fmt`, `vet`, `lint`, and the full test suite.
3. Run `make setup-hooks` once after cloning to install the post-commit hook. It automatically reformats code and regenerates Swagger docs after each commit so those never drift.
4. Keep changes focused. If a PR touches the domain layer and adds a new adapter, split them unless they are logically inseparable.
5. Open a pull request against `main`. CI will run the same `make check` pipeline.

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
