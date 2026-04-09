# beacon-core

> Operational core of the Beacon emergency-response platform. Manages the full lifecycle of city-scale incidents — from first report through dispatch, evacuation, and closure — for the Response to City Disaster project.

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

`beacon-core` is a Go microservice running on GKE (europe-west2). It is the system of record for all active emergency operations on the Beacon platform. Every other service in the ecosystem either feeds data into it or reacts to the events it emits.

**What it manages:**

| Domain | Responsibility |
|---|---|
| Incidents | Reported emergencies — fire, flood, SOS, etc. Full state-machine lifecycle with approval gate, escalation, and audit trail. |
| Dispatches | Assignment of ground-staff teams to approved incidents. Accepts, rejects, completes, or times out. Real-time status via SSE. |
| Evacuations | Structured evacuation zones attached to an incident. Advisory broadcast to all active citizens over GCP Pub/Sub. |
| Vault | GCS-backed document store for incident evidence, attachments, and operational files. Supports folder hierarchy, tags, soft-delete, and bulk operations. |
| Location Tracking | GPS breadcrumb ingestion from ground staff during an active dispatch, stored in ClickHouse for post-incident analysis. |
| SOS | Fast-path incident creation for mobile emergency alerts — auto-approved, severity critical, no admin review required. |
| Map / Geocoding | TomTom-powered address and location search used by dispatchers to locate incidents. |

---

## Architecture

The service follows **Hexagonal Architecture (Ports & Adapters)** with **CQRS** (Command Query Responsibility Segregation).

```
┌──────────────────────────────────────────────────────────────┐
│  HTTP Layer  — Gorilla Mux, JWT middleware, RBAC, Swagger     │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Registry Layer  — CQRS command handlers (write)              │
│                          query handlers  (read)               │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Domain Layer  — pure business logic, repository interfaces,  │
│                  state-machine rules, no external imports      │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│  Adapters Layer                                               │
│  ├── PostgreSQL master  (writes)                              │
│  ├── PostgreSQL slave   (reads)                               │
│  ├── Redis              (incident cache, TTL 1 h)             │
│  ├── ClickHouse         (location track, analytics, audit)    │
│  ├── GCS                (vault file storage)                  │
│  ├── GCP Pub/Sub        (event bus — lifecycle events)        │
│  └── HTTP clients       (beacon-iam, beacon-observability,    │
│                          TomTom)                              │
└──────────────────────────────────────────────────────────────┘
```

**Key design principles:**

- **Domain purity** — no external package imports in `internal/domain/`; all dependencies injected as interfaces.
- **CQRS read/write split** — command handlers write to the PostgreSQL master; query handlers read from the slave replica.
- **Chain of Responsibility** — incident list filtering, sorting, and pagination implemented as composable chain links (AIP-132 / AIP-158 compliant).
- **Dependency injection** — all wiring is done in a single composition root: `cmd/server/main.go`.
- **Fail-open non-critical paths** — FCM token resolution and audit logging degrade gracefully on dependency failure; the main operation is never blocked.

---

## Domains

### Incident Lifecycle

```
reported ──(approve)──► active ──(escalate)──► active (higher severity)
                          │
                      (close) — blocked if active dispatches or evacuations exist
                          │
                        closed
```

Every state transition writes an immutable event record to `incident_events`, visible in the timeline returned by `GET /v1/incidents/{id}/events`. SOS incidents are created pre-approved and bypass the approval step.

### Dispatch Lifecycle

```
created (pending_acceptance)
  ├──(accept)──► in_progress ──(complete)──► completed
  ├──(reject)──► rejected
  ├──(cancel)──► cancelled
  └──(timeout)─► timed_out          ← recovered automatically at startup
```

Dispatch status changes are pushed in real time to connected clients over Server-Sent Events at `GET /v1/dispatches/{id}/events`.

### Evacuation Lifecycle

```
planned ──(activate)──► active ──(complete)──► completed
  └──(cancel)──► cancelled
```

`POST /v1/evacuations/{id}/send-advisory` broadcasts an advisory message to all active citizens via the event bus.

### Vault

A two-phase upload model (initiate → confirm) allows clients to upload directly to GCS without routing binary data through the API server. Files support soft-delete (bin/restore) and permanent deletion.

---

## Service Dependencies

| Service | Transport | What beacon-core needs |
|---|---|---|
| **beacon-iam** | HTTP REST | JWT validation on every request; ground-staff roster for dispatch creation; active FCM push tokens for notifications |
| **beacon-observability** | HTTP REST | Emergency station coordinates for nearest-station dispatch queries |
| **beacon-event-bus** | GCP Pub/Sub | Async publish of all lifecycle events (`incident.*`, `dispatch.*`, `evacuation.*`) |
| **beacon-auditing** | ClickHouse (TCP) | Immutable audit trail of every state mutation |
| **Google Cloud Storage** | GCS SDK | Vault file storage (`beacon-vault` bucket, project `beacon-477817`) |
| **TomTom API** | HTTP REST | Address search for `GET /v1/map/search` |

All service URLs are configured via `config.yaml` (see [Configuration](#configuration)).

---

## Prerequisites

| Tool | Minimum version | Purpose |
|---|---|---|
| Go | 1.24 | Build and run |
| PostgreSQL | 12 | Primary datastore |
| Redis | 6 | Incident cache |
| ClickHouse | 22 | Location tracking and analytics |
| Docker | 20.10 | Container builds |
| `golang-migrate` | latest | Run database migrations |
| `swag` | latest | Regenerate Swagger docs |
| `golangci-lint` | latest | Linting |

Install Go tooling in one step:

```bash
make install-tools
```

---

## Local Development

### 1. Clone

```bash
git clone https://github.com/Response-To-City-Disaster/beacon-core.git
cd beacon-core
```

### 2. Start dependencies

```bash
# PostgreSQL
docker run -d --name beacon-pg \
  -e POSTGRES_USER=core -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=core \
  -p 5432:5432 postgres:15-alpine

# Redis
docker run -d --name beacon-redis -p 6379:6379 redis:7-alpine

# ClickHouse (optional — needed for location tracking)
docker run -d --name beacon-ch \
  -e CLICKHOUSE_USER=myuser -e CLICKHOUSE_PASSWORD=MyStrongPassword123 \
  -p 9000:9000 clickhouse/clickhouse-server:latest
```

### 3. Run migrations

```bash
make migrate-up
```

### 4. Configure

Copy and edit `config.yaml`:

```bash
cp config.yaml config.local.yaml
```

Point `database.master.host`, `redis.host`, and `clickhouse.host` at your local instances. Set `event_bus.enabled: false` and `auditing` fields if those services are not available locally.

### 5. Run

```bash
make run
```

The server starts on `http://localhost:8080`.

### 6. Swagger UI

```bash
make swagger
# open http://localhost:8080/swagger/index.html
```

### Hot reload

```bash
make dev    # uses `air` if installed, falls back to `go run`
```

---

## Configuration

All configuration is loaded from `config.yaml` (Viper). Every key can be overridden by an environment variable using the `BEACON_` prefix and `_` as the level separator (e.g. `database.master.host` → `BEACON_DATABASE_MASTER_HOST`).

```yaml
server:
  port: 8080
  timeout: 30s

database:
  master:
    host: ""
    port: 5432
    user: ""
    password: ""
    dbname: "core"
    max_open_conns: 25
    max_idle_conns: 5
  slave:
    host: ""
    port: 5432
    user: ""
    password: ""
    dbname: "core"
    max_open_conns: 50
    max_idle_conns: 10

redis:
  host: ""
  port: 6379
  password: ""
  db: 0
  ttl: 3600           # Cache TTL in seconds

clickhouse:
  host: ""
  port: 9000
  username: ""
  password: ""
  dbname: "core"

storage:
  bucket_name: ""
  project_id: ""
  service_account_email: ""

iam:
  url: ""
  service_username: ""
  service_password: ""

notification:
  url: ""

observability:
  url: ""
  country_id: 0

dispatch:
  timeout_minutes: 2

tomtom:
  api_key: ""
  timeout: 10s

event_bus:
  project_id: ""
  credentials_file: ""  # empty = Application Default Credentials
  enabled: true

auditing:
  host: ""
  port: 9000
  database: "beacon_audit"
  username: ""
  password: ""
  secure: false

logging:
  level: "info"         # debug | info | warn | error
  format: "console"     # console | json
```

In Kubernetes, sensitive values (`db-master-host`, `db-password`, `clickhouse-password`, etc.) are sourced from the `beacon-core-secrets` Kubernetes Secret — never stored in the image or config file.

---

## API Reference

All endpoints are under `/v1`. Authentication is required on every endpoint except `/health` and `/ready`. The JWT is validated against **beacon-iam**.

### Role matrix

| Role | Permissions |
|---|---|
| `citizen` | Create incidents, report SOS, read all public data |
| `ground_staff` | Accept/reject dispatches, submit GPS location updates |
| `ert_member` | All of the above + create dispatches and evacuations, approve/close incidents |
| `admin` | Full access |

---

### Health

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/health` | — | Liveness probe |
| `GET` | `/ready` | — | Readiness probe |

---

### Incidents

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/incidents` | citizen + | Create a new incident |
| `GET` | `/v1/incidents` | all | List incidents (filterable, sortable, paginated) |
| `GET` | `/v1/incidents/{id}` | all | Get incident by ID |
| `PUT` | `/v1/incidents/{id}` | staff + | Update incident fields |
| `POST` | `/v1/incidents/{id}/approve` | staff + | Approve a reported incident |
| `POST` | `/v1/incidents/{id}/close` | staff + | Close incident (blocked if active dispatches/evacuations exist) |
| `POST` | `/v1/incidents/{id}/escalate` | staff + | Escalate to higher severity |
| `POST` | `/v1/incidents/{id}/update` | staff + | Record an update and optionally alert citizens |
| `GET` | `/v1/incidents/{id}/events` | all | Paginated event history (audit timeline) |
| `GET` | `/v1/incidents/metrics` | all | Aggregate counts by status/severity/type |
| `GET` | `/v1/analytics` | all | Monthly trend analytics |

**List filters** (`GET /v1/incidents`):

| Parameter | Type | Description |
|---|---|---|
| `status` | string | `reported` \| `active` \| `closed` |
| `severity` | string | `low` \| `medium` \| `high` \| `critical` |
| `type` | string | `fire` \| `flood` \| `accident` \| `sos` \| … |
| `approved` | bool | Filter by approval state |
| `is_internal` | bool | Filter internal-only incidents |
| `sort_by` | string | `created_at` \| `severity` \| `status` |
| `sort_order` | string | `asc` \| `desc` |
| `page_size` | int | Results per page (default: 20) |
| `page_token` | string | Cursor for next page |

---

### SOS

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/incidents/sos` | all | Report an emergency SOS alert (auto-approved, critical severity) |

---

### Dispatches

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/dispatches` | ert_member, admin | Create a dispatch for an approved incident |
| `GET` | `/v1/dispatches` | staff + | List all dispatches |
| `GET` | `/v1/dispatches/{id}` | staff + | Get dispatch by ID |
| `POST` | `/v1/dispatches/{id}/accept` | ground_staff + | Accept a pending dispatch |
| `POST` | `/v1/dispatches/{id}/reject` | ground_staff + | Reject a pending dispatch |
| `POST` | `/v1/dispatches/{id}/complete` | ert_member, admin | Mark dispatch as completed |
| `POST` | `/v1/dispatches/{id}/cancel` | ert_member, admin | Cancel a dispatch |
| `GET` | `/v1/dispatches/{id}/events` | staff + | **SSE stream** — real-time status updates |
| `GET` | `/v1/dispatches/{id}/location-track` | ert_member, admin | Full GPS track for a completed dispatch |
| `POST` | `/v1/dispatches/{id}/location-updates` | ground_staff | Batch-ingest GPS updates during an active dispatch |
| `GET` | `/v1/incidents/{id}/dispatches` | all | List all dispatches for an incident |
| `GET` | `/v1/incidents/{id}/stations/{affiliation_id}/availability` | staff + | Staff availability at a station |
| `GET` | `/v1/incidents/{id}/nearest-stations` | staff + | Ranked nearest stations to the incident |

---

### Evacuations

| Method | Path | Roles | Description |
|---|---|---|---|
| `POST` | `/v1/evacuations` | ert_member, admin | Create an evacuation zone |
| `GET` | `/v1/evacuations` | all | List all evacuations |
| `GET` | `/v1/evacuations/{id}` | all | Get evacuation by ID |
| `PUT` | `/v1/evacuations/{id}` | ert_member, admin | Update evacuation details |
| `POST` | `/v1/evacuations/{id}/cancel` | ert_member, admin | Cancel an evacuation |
| `POST` | `/v1/evacuations/{id}/send-advisory` | ert_member, admin | Broadcast advisory to all active citizens |
| `DELETE` | `/v1/evacuations/{id}` | ert_member, admin | Delete an evacuation |
| `GET` | `/v1/incidents/{id}/evacuations` | all | List evacuations for an incident |

---

### Vault

All vault endpoints require authentication. Citizens get read access; staff and above can upload and manage.

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/vault/stats` | Storage usage statistics |
| `GET` | `/v1/vault/root` | Root folder contents |
| `GET` | `/v1/vault/starred` | Starred files |
| `GET` | `/v1/vault/bin` | Soft-deleted files |
| `POST` | `/v1/vault/folders` | Create a folder |
| `GET` | `/v1/vault/folders/{id}` | Get folder contents |
| `POST` | `/v1/vault/files/upload` | Initiate a GCS upload (returns signed URL) |
| `POST` | `/v1/vault/files/{id}/confirm` | Confirm upload complete |
| `GET` | `/v1/vault/files/{id}` | Get file metadata |
| `PATCH` | `/v1/vault/files/{id}` | Update file metadata |
| `DELETE` | `/v1/vault/files/{id}` | Soft delete |
| `POST` | `/v1/vault/files/{id}/move` | Move to another folder |
| `POST` | `/v1/vault/files/{id}/star` | Star a file |
| `DELETE` | `/v1/vault/files/{id}/star` | Unstar a file |
| `POST` | `/v1/vault/files/{id}/restore` | Restore from bin |
| `DELETE` | `/v1/vault/files/{id}/permanent` | Permanently delete |
| `GET` | `/v1/vault/files/{id}/tags` | List tags |
| `POST` | `/v1/vault/files/{id}/tags` | Add tag |
| `DELETE` | `/v1/vault/files/{id}/tags/{tagId}` | Remove tag |
| `POST` | `/v1/vault/bulk/delete` | Bulk soft delete |
| `POST` | `/v1/vault/bulk/move` | Bulk move |
| `POST` | `/v1/vault/bulk/star` | Bulk star |
| `POST` | `/v1/vault/bulk/restore` | Bulk restore |
| `POST` | `/v1/incidents/{id}/attachments` | Attach a vault file to an incident |
| `GET` | `/v1/incidents/{id}/attachments` | List attachments for an incident |
| `GET` | `/v1/incidents/{id}/vault-folder` | Get or create the incident's vault folder |

---

### Map

| Method | Path | Roles | Description |
|---|---|---|---|
| `GET` | `/v1/map/search?query=<address>` | all | Geocode an address or place name via TomTom |

---

### Example requests

```bash
# Create an incident
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

# Approve it (staff)
curl -X POST https://beacon-tcd.tech/api/core/v1/incidents/<id>/approve \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message": "Verified by control room"}'

# Create a dispatch to the nearest station
curl -X POST https://beacon-tcd.tech/api/core/v1/dispatches \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "incident_id": "<incident-id>",
    "affiliation_id": "<station-id>",
    "role": "fire_brigade",
    "notes": "Bring breathing apparatus"
  }'

# Stream dispatch status in real time
curl -N -H "Authorization: Bearer <token>" \
  https://beacon-tcd.tech/api/core/v1/dispatches/<id>/events

# Report SOS
curl -X POST https://beacon-tcd.tech/api/core/v1/incidents/sos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "latitude": 53.3498,
    "longitude": -6.2603,
    "has_location": true,
    "description": "Help needed immediately"
  }'
```

---

## Testing

### Unit tests

```bash
make test
# generates coverage.html
```

### Run with race detector

```bash
go test -v -race ./...
```

### Integration tests

Integration tests run against a live CloudSQL instance. They are skipped automatically in CI if `SKIP_INTEGRATION_TESTS=true`.

```bash
make test-crud-integration
```

To run against a local database:

```bash
DB_HOST=localhost DB_PORT=5432 DB_NAME=core DB_USER=core DB_PASSWORD=secret \
  make test-crud-integration
```

---

## Docker

### Build

```bash
make docker-build
```

The image uses a multi-stage build (`golang:1.24-alpine` builder → `gcr.io/distroless/static:nonroot` runtime). The final image runs as a non-root user and contains only the compiled binary and `config.yaml`.

Private Go modules are resolved at build time via `GITHUB_TOKEN`:

```bash
docker build --build-arg GITHUB_TOKEN=<token> -t beacon-core:latest .
```

### Run locally

```bash
make docker-run
# or
docker run --rm -p 8080:8080 \
  -e BEACON_DATABASE_MASTER_HOST=host.docker.internal \
  -e BEACON_DATABASE_SLAVE_HOST=host.docker.internal \
  -e BEACON_DATABASE_MASTER_USER=core \
  -e BEACON_DATABASE_MASTER_PASSWORD=secret \
  -e BEACON_DATABASE_MASTER_DBNAME=core \
  beacon-core:latest
```

---

## Kubernetes / GKE Deployment

The service runs in the `staging` / `production` namespaces on GKE (europe-west2). Images are stored in GCP Artifact Registry.

### One-time setup

```bash
# Authenticate Docker with Artifact Registry
make gcp-auth GCP_PROJECT_ID=beacon-477817

# Create the secrets in the target namespace
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
# Build, tag, and push to Artifact Registry
make gke-build-push GCP_PROJECT_ID=beacon-477817 GKE_CLUSTER_NAME=beacon-cluster

# Apply manifests and wait for rollout
make gke-deploy GCP_PROJECT_ID=beacon-477817 GKE_CLUSTER_NAME=beacon-cluster NAMESPACE=staging
```

### Useful operations

```bash
make gke-logs      # tail live logs
make gke-pods      # list running pods
make gke-status    # show service IP
make gke-restart   # rolling restart
make gke-scale REPLICAS=3   # scale horizontally
```

### Kubernetes manifests

| File | Purpose |
|---|---|
| `k8s/deployment.yml` | Deployment with resource limits, liveness/readiness probes, and env-var injection from secrets |
| `k8s/service.yml` | Service definition |
| `k8s/secret.yml` | Secret template (values populated at deploy time, never committed) |
| `k8s/istio.yml` | Istio virtual service / traffic policy |

**Resource limits** (per pod):

| | Request | Limit |
|---|---|---|
| CPU | 20m | 50m |
| Memory | 75Mi | 100Mi |

**Health probes:**

- Liveness: `GET /health` — initial delay 30 s, period 10 s
- Readiness: `GET /ready` — initial delay 10 s, period 5 s

---

## Database Migrations

Migrations are managed with [`golang-migrate`](https://github.com/golang-migrate/migrate) and live in `migrations/`. Each migration has a `.up.sql` and `.down.sql` pair.

```
migrations/
├── 000001_create_incidents_table
├── 000002_add_approved_to_incidents
├── 000003_create_incident_events_table
├── 000004_create_dispatches_table
├── 000005_create_dispatch_members_table
├── 000006_create_evacuations_table
├── 000007_create_vault_tables
├── 000008_add_event_metadata
├── 000009_fix_leaked_dispatch_members
├── 000009_extend_evacuations_advisory
├── 000010_add_transport_mode_to_evacuations
├── 000011_simplify_evacuations
├── 000012_add_is_internal_to_incidents
└── clickhouse/001_staff_location_updates
```

```bash
# Local
make migrate-up
make migrate-down

# Against cloud database
make migrate-up-cloud
make migrate-down-cloud
```

The `cmd/migrate` binary is also available as a standalone migration runner used in CI pipelines.

---

## Project Structure

```
beacon-core/
├── cmd/
│   ├── server/main.go          # Composition root — all wiring lives here
│   └── migrate/main.go         # Standalone migration runner
├── internal/
│   ├── domain/                 # Pure business logic — no external dependencies
│   │   ├── incident/           # Incident state machine, event recording, metrics
│   │   ├── dispatch/           # Dispatch lifecycle, timeout recovery, SSE events
│   │   ├── evacuation/         # Evacuation zones, advisory broadcast
│   │   ├── location/           # GPS track service
│   │   ├── vault/              # Vault domain model
│   │   ├── route/              # Route domain (planned)
│   │   └── mapfeature/         # Map feature domain (planned)
│   ├── http/
│   │   ├── router.go           # All routes with RBAC middleware applied
│   │   ├── middleware.go        # Logging, CORS, tracing, auth
│   │   └── handlers/           # HTTP ↔ CQRS translation (one file per domain)
│   ├── registry/
│   │   ├── commands/           # Write operations — one handler per command
│   │   └── queries/            # Read operations — one handler per query
│   ├── adapters/
│   │   ├── database/           # PostgreSQL master/slave repositories
│   │   ├── db/postgres/        # Vault repository
│   │   ├── clickhouse/         # Location track repository
│   │   ├── eventbus/           # GCP Pub/Sub publisher adapters
│   │   ├── microservices/      # HTTP clients: IAM, Observability, TomTom
│   │   └── storage/gcs/        # Google Cloud Storage adapter
│   └── sse/
│       └── broker.go           # In-memory SSE broker for dispatch events
├── pkg/                        # Shared packages
│   ├── auth/                   # JWT parsing, role extraction, middleware
│   ├── config/                 # Viper config loader
│   ├── errors/                 # Typed error constructors
│   ├── logger/                 # Zerolog wrapper
│   ├── postgres/               # Master/slave connection pool setup
│   ├── redis/                  # Redis client setup
│   ├── clickhouse/             # ClickHouse connection setup
│   ├── tracing/                # OpenTelemetry trace middleware
│   └── response/               # Standard HTTP response helpers
├── migrations/                 # golang-migrate SQL files
├── k8s/                        # Kubernetes manifests
├── docs/                       # Swagger-generated API docs
├── tests/
│   ├── unit/                   # Unit tests
│   └── integration/            # Integration tests (require live database)
├── Dockerfile                  # Multi-stage build → distroless runtime
├── Makefile                    # All developer and deployment targets
└── config.yaml                 # Default configuration (no secrets)
```

---

## Contributing

1. Branch from `main`: `git checkout -b feature/<name>`
2. Run `make check` (fmt + vet + lint + test) before pushing
3. Swagger docs are regenerated automatically via the post-commit hook — run `make setup-hooks` once after cloning
4. Open a pull request against `main`

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
