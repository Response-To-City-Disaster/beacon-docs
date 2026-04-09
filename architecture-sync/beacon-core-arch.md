# beacon-core: Architecture Guide for New Engineers

> Written by a senior engineer who has spent too many late nights in this codebase.
> Read this before you touch anything.

---

## The Big Picture

**beacon-core is the operational brain of the city emergency-response platform.**

When a fire breaks out, a flood hits, or someone presses SOS on their phone, it ends up here. This service owns the full lifecycle of city-scale emergencies from the moment a citizen taps "Report Incident" to the moment a commander types "Incident Closed."

Concretely, it manages five things:

| Domain | What it is |
|---|---|
| **Incidents** | The emergency records themselves — fire, flood, SOS, etc. They move through a state machine: `reported → approved → active → closed`. |
| **Dispatches** | Assignments of ground-staff teams to approved incidents. A dispatcher creates one, a ground-staff member accepts/rejects it, and it eventually completes. Real-time status streams over SSE. |
| **Evacuations** | Structured evacuation zones attached to an incident. Commanders can broadcast advisory messages to all active citizens. |
| **Vault** | A GCS-backed document store. Evidence photos, PDFs, and files can be attached to an incident. Think of it as a structured Google Drive scoped to emergencies. |
| **Location Tracking** | GPS breadcrumbs from ground staff during a dispatch, stored in ClickHouse for post-incident analysis. |

If you remember nothing else: **you cannot dispatch responders, order an evacuation, or close an incident unless it has first been approved.** That approval gate runs through all three domains and is enforced at the service layer.

---

## The Tour: 5 Files That Run This Service

### 1. `cmd/server/main.go` — The Composition Root

This is the most important file in the repository. It is *long*, and that is intentional. Every adapter, every service, every handler, and every cross-domain wire-up lives here. If you want to understand how two pieces of the system connect to each other, start here.

It also runs `dispatchService.RecoverExpiredDispatches()` at startup — a crash recovery step that cleans up dispatches that timed out while the server was offline. Do not reorder things in this file without reading the full dependency chain first.

### 2. `internal/domain/incident/service.go` — The Core Business Logic

The `incident.Service` struct is where all incident state transitions are enforced. Before you can close an incident, it checks for active dispatches (`DispatchChecker`) and active evacuations (`EvacuationChecker`). Before a dispatch can be created, the incident must be approved. This file is where all those rules live.

It also handles audit logging (via `beacon-auditing`), Redis cache invalidation, event publishing to the event bus, and recording the structured event history (the audit trail visible to citizens). If you need to understand how any incident mutation works end-to-end, this is the file.

### 3. `internal/http/router.go` — The Full API Surface

All HTTP routes, their required roles, and which handler they delegate to are defined here. The role-based access control (RBAC) is set up as middleware sub-routers — there is no annotation magic, it is explicit. Before adding a new endpoint, read this file to understand which `auth.RequireRole(...)` group it should belong to.

Roles in play: `Admin`, `GroundStaff`, `ERTMember`, `Citizen`. Citizens can report incidents and read data. Ground staff can accept/reject dispatches. ERT members and admins can create dispatches and evacuations.

### 4. `internal/registry/` — The CQRS Layer

Every user action is a named **command** (`registry/commands/`) or **query** (`registry/queries/`). These handlers are thin wrappers — they translate the HTTP request into a domain call and back. If you need to add a new operation, this is where you add the handler; the domain service is where you add the logic.

The separation matters: commands mutate state, queries read it. Queries always go to the Postgres *slave* replica; commands go to the *master*. Do not accidentally call a master repo from a query handler.

### 5. `internal/sse/broker.go` — Real-Time Dispatch Events

Ground staff track dispatch status (accepted, rejected, completed, timed-out) over Server-Sent Events. The `Broker` is a simple in-memory pub/sub keyed by dispatch ID. It is fast and stateless per connection, which is good.

The catch: it is **in-memory only**. See the Coupling Risks section below.

---

## The Neighborhood: Who We Talk To

The graph of service dependencies, derived from `go.mod`, `config.yaml`, and the adapters layer:

```
                        ┌─────────────────┐
                        │   beacon-core   │
                        └────────┬────────┘
         ┌──────────────┬────────┴──────────┬────────────────┐
         ▼              ▼                   ▼                ▼
   beacon-iam   beacon-observability  beacon-event-bus  beacon-auditing
   (HTTP REST)    (HTTP REST)          (GCP Pub/Sub)    (ClickHouse)
         │
         │  also: Google Cloud Storage (Vault bucket)
         │  also: TomTom API (geocoding / map search)
```

| Service | What we need from it | Where the call lives |
|---|---|---|
| **beacon-iam** | (1) JWT validation on every request. (2) Ground-staff roster for dispatch availability. (3) Active FCM tokens for push notifications. | `internal/adapters/microservices/iam_client.go` |
| **beacon-observability** | List of emergency stations with coordinates — used to find the nearest station to an incident when creating a dispatch. | `internal/adapters/microservices/observability_client.go` |
| **beacon-event-bus** | Async publish of lifecycle events (`incident.created`, `incident.approved`, `dispatch.accepted`, `evacuation.advisory`, etc.) consumed by `beacon-notification` and `beacon-dashboard`. | `internal/adapters/eventbus/` |
| **beacon-auditing** | Structured audit log of every state mutation (who changed what, when). Stored in ClickHouse. | `cmd/server/main.go` — injected directly into services via `SetAuditClient()` |
| **GCS** (`beacon-vault` bucket) | File storage for the Vault feature — upload, move, star, soft-delete. | `internal/adapters/storage/gcs/gcs_client.go` |
| **TomTom** | Location/address search used by the `/v1/map/search` endpoint. | `internal/adapters/microservices/tomtom_geocoding_client.go` |

**Important:** beacon-core calls beacon-iam on *every dispatch creation* (to check staff availability) and on *every incident/evacuation event publication* (to fetch FCM push tokens). If IAM is degraded, these paths degrade with it. FCM token resolution is designed to fail open — it logs a warning and continues without recipients. Staff availability checks are not fail-open; a dispatch creation will error if IAM is unreachable.

---

## Pro-Tips: Don't Break These Things

These are the areas flagged during code analysis where a small, innocent-looking change can cascade into production failures.

### 1. Cross-Domain Adapters in `main.go` — The Hidden Coupling Web

Three anonymous structs defined in `main.go` stitch the domains together:

- `incidentApprovalAdapter` — lets dispatch and evacuation services check if an incident is approved before allowing any action
- `dispatchActivityAdapter` — lets the incident service block closure when active dispatches exist
- `evacuationActivityAdapter` — lets the incident service block closure when active evacuations exist

These adapters implement interfaces defined *in the domain packages*. If you change `incident.SlaveRepository`, `dispatch.SlaveRepository`, or `evacuation.SlaveRepository`, you will silently break one of these adapters. The compiler will catch it, but only if you run `go build`. **Always build before pushing.**

### 2. The In-Memory SSE Broker Cannot Scale Horizontally

`internal/sse/broker.go` stores all live subscriptions in a Go `map` protected by a `sync.RWMutex`. This means:

- If you run two instances of beacon-core (e.g., for load balancing), a ground-staff client connected to instance A will miss events published by instance B.
- Do not add horizontal scaling before replacing this with a Redis pub/sub or similar distributed mechanism.

### 3. Dispatch Timeout Recovery at Startup

`dispatchService.RecoverExpiredDispatches(context.Background())` runs **synchronously** during startup, before the HTTP server starts accepting connections. It scans for dispatches that are still `pending_acceptance` or `in_progress` past their timeout window and marks them as timed out. 

If you change the dispatch `timeout_minutes` config value, you may inadvertently time-out dispatches that were still valid. If you change the startup sequence order in `main.go`, you risk either crashing before recovery runs, or serving traffic before the database state is consistent.

### 4. Event Bus Adapter Struct Coupling

`internal/adapters/eventbus/adapters.go` maps domain structs directly to Pub/Sub message payloads. If you add a new field to `incident.Incident` and also need it in downstream consumers (e.g., `beacon-dashboard`), you must update the publisher adapter in this package too. The compiler will *not* catch a missing field in the payload mapping. Test the event payload schema explicitly.

### 5. Redis Cache Is Fail-Open — But Stale Data Is Real

Incidents are cached in Redis (TTL: 1 hour, from `config.yaml`). Cache writes and invalidations fail silently — they log a warning and continue. This is the right design for availability, but it means:

- After an `UpdateIncident` or `CloseIncident`, the cache is invalidated.
- If the invalidation silently fails (Redis down), the stale version will be served for up to 1 hour.
- Never add business-logic side effects that depend on the cache being fresh. Always treat the cache as a best-effort read optimization.

### 6. Approval Is the Master Gate — Touch It Carefully

The `approved` boolean on an incident is checked in at least four places:
- `CreateDispatch` — rejects if not approved
- `CreateEvacuation` — rejects if not approved
- `PublishIncidentCreated` — only publishes to event bus if approved
- `PublishIncidentClosed` — only publishes to event bus if approved

If you ever change the approval semantics (e.g., auto-approve certain incident types), audit every one of these call sites. Missing one will silently suppress push notifications or allow dispatches on unverified reports.

---

## Quick Reference

```
beacon-core/
├── cmd/server/main.go          # START HERE — wires all dependencies
├── config.yaml                 # Service URLs, DB/Redis/ClickHouse creds
├── internal/
│   ├── domain/                 # Business logic — the law of the land
│   │   ├── incident/           # Incident lifecycle + audit events
│   │   ├── dispatch/           # Dispatch assignment + SSE publishing
│   │   ├── evacuation/         # Evacuation zones + advisory broadcast
│   │   ├── location/           # GPS track ingestion/query
│   │   └── vault/              # GCS file management model
│   ├── http/
│   │   ├── router.go           # All routes + RBAC middleware
│   │   └── handlers/           # HTTP ↔ CQRS translation
│   ├── registry/
│   │   ├── commands/           # Mutating operations (write path)
│   │   └── queries/            # Read operations (slave DB path)
│   ├── adapters/
│   │   ├── database/           # Postgres master/slave repos
│   │   ├── microservices/      # HTTP clients for IAM, Observability, TomTom
│   │   ├── eventbus/           # GCP Pub/Sub publisher adapters
│   │   └── storage/gcs/        # Google Cloud Storage adapter
│   └── sse/broker.go           # In-memory SSE dispatch event broker
├── migrations/                 # postgres-migrate SQL files (numbered)
└── pkg/                        # Shared utilities (auth, config, logger, redis, etc.)
```

---

---

## Graph Analysis (code-review-graph v1.27.0)

> Generated from the live knowledge graph — 615 nodes, 2851 edges, 109 files.

### Summary

| Metric | Value |
|---|---|
| Total nodes | 615 (294 types, 193 functions, 109 files, 19 tests) |
| Total edges | 2851 (1558 CALLS, 572 CONTAINS, 472 IMPORTS_FROM, 249 TESTED_BY) |
| Risk score | **medium (0.55)** |
| Test gaps | **51 untested symbols** |
| Languages | Go only |

### Communities (Detected Clusters)

The graph algorithm found **15 logical communities**. Cohesion score = how tightly the cluster's internal edges are relative to external ones (1.0 = perfectly isolated, 0.0 = fully porous).

| Community | Size | Cohesion | Health |
|---|---|---|---|
| `incident-event` | 25 | **0.93** | Excellent — domain types are tightly scoped |
| `evacuation-advisory` | 11 | 0.83 | Good |
| `vault-file` | 12 | 0.79 | Good |
| `errors-error` | 17 | 0.73 | Good |
| `dispatch-dispatch` | 15 | 0.67 | Acceptable |
| `handlers-request` | 16 | 0.58 | Acceptable |
| `config-config` | 17 | 0.52 | Acceptable |
| `unit-incident` | 22 | 0.50 | Watch — tests are tightly coupled to domain types |
| `microservices-response` | 11 | 0.40 | Concerning |
| `handlers-incident` | 10 | 0.36 | Concerning |
| `response-response` | 11 | 0.29 | Concerning |
| `eventbus-adapter` | 10 | **0.23** | High coupling risk — see below |
| `dispatch-event` | 14 | **0.22** | High coupling risk — see below |
| `auth-user` | 20 | **0.19** | Highest coupling risk in the codebase |
| `incident-checker` | 11 | **0.19** | High coupling risk — cross-domain interfaces |

### Critical Execution Flows (by criticality score)

The top flows are all in the **auth/middleware layer**, meaning every single HTTP request passes through them. Breaking anything here takes down the whole API.

| Score | Flow | What it does |
|---|---|---|
| 0.62 | `Middleware` (auth) | JWT validation on every request — most critical path |
| 0.61 | `RequireRole` | RBAC enforcement for all protected routes |
| 0.61 | `NewConnectionPools` | DB connection setup at startup |
| 0.53 | `GetUserPrimaryRole` | Extracts role claim from JWT — used in 3 domains |
| 0.50 | `IsAdmin` | Admin check used across handlers |
| 0.40 | `TestIncidentCRUD` | Integration test — highest-coverage test flow |

### Coupling Risks Flagged by Graph

Three communities stand out with cohesion below 0.25 — meaning their internal code has strong dependencies going **outward** to other clusters:

1. **`auth-user` (cohesion 0.19, 20 nodes)** — The auth package is imported by almost every other package. It has the most cross-community CALLS edges in the graph. This is expected but means any breaking change to `pkg/auth` (JWT claims shape, role constants, middleware signature) will ripple through all 5 domains simultaneously.

2. **`eventbus-adapter` (cohesion 0.23, 10 nodes)** — The event bus adapters in `internal/adapters/eventbus/` each import from a different domain package (`incident`, `dispatch`, `evacuation`) to map types to Pub/Sub payloads. Changing any domain struct without updating its adapter produces silent payload truncation — no compile error.

3. **`dispatch-event` (cohesion 0.22, 14 nodes)** — The dispatch event/SSE types are consumed by both the dispatch service and the SSE broker. The low cohesion indicates these types are reaching across layers (domain → adapter → handler) in ways the graph considers risky.

4. **`incident-checker` (cohesion 0.19, 11 nodes)** — These are the cross-domain checker interfaces (`DispatchChecker`, `EvacuationChecker`) wired in `main.go`. They form a hidden dependency bridge between all three domains.

### Test Gap Warning

The graph identified **51 functions with no test coverage**. The most critical untested area is in `internal/adapters/microservices/` — the IAM, Observability, and TomTom clients have no unit tests, meaning failures in those integrations will only be caught in production or integration tests.

---

*Last updated by code-review-graph v1.27.0 analysis, 2026-04-09.*
