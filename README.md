# Beacon — Platform Overview

Beacon is a city-scale emergency response platform built for Irish local authorities and emergency services. It coordinates the full lifecycle of a city emergency — from the moment a citizen taps "Report Incident" to the moment an incident commander types "Incident Closed" — across a suite of twelve purpose-built repositories.

This document is the single entry point for understanding the entire platform: what each service does, how they connect, how data flows through the system, and where to find deeper documentation.

---

## Table of Contents

1. [What Beacon Does](#what-beacon-does)
2. [Repository Map](#repository-map)
3. [System Architecture](#system-architecture)
4. [Service Dependency Graph](#service-dependency-graph)
5. [Data Flow: A Citizen Reports a Fire](#data-flow-a-citizen-reports-a-fire)
6. [Data Flow: A Dispatch Is Created](#data-flow-a-dispatch-is-created)
7. [Data Flow: An Evacuation Is Broadcast](#data-flow-an-evacuation-is-broadcast)
8. [Infrastructure](#infrastructure)
9. [Shared Libraries](#shared-libraries)
10. [Authentication and Roles](#authentication-and-roles)
11. [Event Bus: Topics and Consumers](#event-bus-topics-and-consumers)
12. [Data Stores](#data-stores)
13. [External API Integrations](#external-api-integrations)
14. [Local Development](#local-development)
15. [Deployment](#deployment)
16. [Code Graph Explorer](#code-graph-explorer)
17. [Per-Repository Documentation](#per-repository-documentation)

---

## What Beacon Does

A citizen in Dublin notices a building fire. They open the Beacon mobile app and tap "Report Incident." That single tap sets off a coordinated sequence across twelve services:

- The incident is created and a push notification reaches every on-duty Emergency Response Team (ERT) member in the city.
- An ERT commander reviews the report on the web dashboard, approves it, and assigns the nearest available ground-staff team.
- The assigned ground-staff member receives a push notification and accepts the dispatch from their phone. Their device begins streaming GPS breadcrumbs.
- Citizens in the surrounding blocks receive an evacuation advisory with turn-by-turn routing that navigates them away from the building.
- Real-time city data — traffic closures, hospital ED capacity, power grid status — updates the ERT dashboard automatically.
- A citizen too panicked to act gets calm guidance from the AI support bot, which escalates to a human support worker when distress indicators are detected.
- Every action — approval, dispatch, evacuation order, closure — is written to an immutable audit trail in ClickHouse.

When the fire is under control, the incident commander closes the incident. Notifications go out. The audit trail is complete. The data flows into ClickHouse for post-incident analytics.

---

## Repository Map

The platform is split into twelve repositories across three categories:

### Go Microservices (the operational core)

| Repository | Port | Role |
|---|---|---|
| [beacon-core](architecture-sync/beacon-core-readme.md) | 8080 | Incident, dispatch, evacuation, vault, and location lifecycle management. The operational brain. |
| [beacon-iam](architecture-sync/beacon-iam-readme.md) | 8081 | Identity and access management — authentication, JWT validation, FCM token registry, ground-staff roster. |
| [beacon-notification](architecture-sync/beacon-notification-readme.md) | 8083 | Multi-channel notification delivery — Firebase push, Twilio SMS, SendGrid email. |
| [beacon-observability](architecture-sync/beacon-observability-readme.md) | 8082 | Real-time city data API — traffic, transport, weather, hospitals, power grid, water levels, ambulances. |
| [beacon-communication-platform](architecture-sync/beacon-communication-platform-readme.md) | 8084 | WebSocket team chat, social posts, Twilio voice channels. |
| [beacon-support-bot](architecture-sync/beacon-support-bot-readme.md) | 8085 | Vertex AI conversational support, distress detection, automatic escalation to human agents. |

### Go Libraries (shared infrastructure)

| Repository | Role |
|---|---|
| [beacon-auditing](architecture-sync/beacon-auditing-readme.md) | Async ClickHouse audit log client. Imported by all microservices. |
| [beacon-event-bus](architecture-sync/beacon-event-bus-readme.md) | GCP Pub/Sub wrapper with CloudEvents model, circuit breaker, and mock backend. Imported by beacon-core and beacon-notification. |
| [beacon-scheduler](architecture-sync/beacon-scheduler-readme.md) | GCP Cloud Scheduler adapter. Imported by beacon-notification for deferred delivery. |

### Data, Frontend, and Mobile

| Repository | Role |
|---|---|
| [beacon-etl-flows](architecture-sync/beacon-etl-flows-readme.md) | Apache Airflow DAGs — 13 ETL pipelines ingesting Irish public APIs into ClickHouse, 5 alert DAGs that fire incidents to beacon-core when thresholds are crossed. |
| [beacon-dashboard](architecture-sync/beacon-dashboard-readme.md) | React 18 + TypeScript ERT web dashboard — incident management, dispatch coordination, evacuation planning, analytics. |
| [beacon-mobile-app](architecture-sync/beacon-mobile-app-readme.md) | Flutter mobile app for citizens and ground staff — incident reporting, live map, safety info, AI chat, dispatch management. |

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                        │
│                                                                                  │
│   beacon-mobile-app (Flutter)           beacon-dashboard (React + TypeScript)    │
│   Citizens + Ground Staff               ERT Members + Admins                     │
└────────────────┬──────────────────────────────────────┬─────────────────────────┘
                 │ HTTPS + WebSocket                    │ HTTPS + SSE
                 ▼                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           SERVICE LAYER                                          │
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  ┌───────────────┐  │
│  │ beacon-core  │  │ beacon-iam   │  │ beacon-observa-    │  │ beacon-       │  │
│  │ :8080        │  │ :8081        │  │ bility :8082        │  │ notification  │  │
│  │              │  │              │  │                    │  │ :8083         │  │
│  │ Incidents    │  │ Auth + JWT   │  │ Traffic / Weather  │  │ FCM / SMS /   │  │
│  │ Dispatches   │  │ FCM tokens   │  │ Hospitals / Power  │  │ Email         │  │
│  │ Evacuations  │  │ Staff roster │  │ Transport / Floods │  │               │  │
│  │ Vault / GPS  │  │              │  │                    │  │               │  │
│  └──────┬───────┘  └──────────────┘  └────────────────────┘  └───────┬───────┘  │
│         │                                                             │          │
│  ┌──────▼──────────────────────────────────────────────────────────┐ │          │
│  │  beacon-communication-platform :8084                             │ │          │
│  │  WebSocket channels · Social posts · Voice (Twilio)             │ │          │
│  └──────────────────────────────────────────────────────────────────┘ │          │
│                                                                        │          │
│  ┌──────────────────────────────────────────────────────────────────┐ │          │
│  │  beacon-support-bot :8085                                        │ │          │
│  │  Vertex AI · Risk analysis · Escalation · Wellbeing follow-ups  │ │          │
│  └──────────────────────────────────────────────────────────────────┘ │          │
└──────────────────────────────────────────────────────────────────────┼───────────┘
                                                                        │
┌──────────────────────────────────────────────────────────────────────▼───────────┐
│                         ASYNC / MESSAGING LAYER                                  │
│                                                                                  │
│          GCP Pub/Sub (beacon-event-bus library)                                  │
│          Topics: beacon-incidents · beacon-dispatches · beacon-evacuations       │
└──────────────────────────────────────────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────────────────────────┐
│                            DATA LAYER                                            │
│                                                                                  │
│  PostgreSQL (master + slave)    Redis (cache + session)    ClickHouse            │
│  ─────────────────────────      ───────────────────────    ──────────────────    │
│  beacon-core    (core DB)       Incident cache (1h TTL)    audit_events (1y TTL) │
│  beacon-iam     (iam DB)        Auth sessions               external_data (ETL)  │
│  beacon-notif.  (notif DB)                                  GPS breadcrumbs      │
│  beacon-obs.    (obs DB)                                    analytics             │
│  beacon-comms.  (comms DB)                                                       │
│  beacon-bot     (bot DB)                                                         │
└──────────────────────────────────────────────────────────────────────────────────┘
                 ▲
┌────────────────┴─────────────────────────────────────────────────────────────────┐
│                         DATA INGESTION LAYER                                     │
│                                                                                  │
│  beacon-etl-flows (Apache Airflow on Kubernetes)                                 │
│  13 ETL DAGs: NTA, Met Éireann, ESB, TomTom, Irish Rail, Luas, waterlevel.ie    │
│  5 Alert DAGs: fires incidents to beacon-core when thresholds exceeded           │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Service Dependency Graph

The arrows show runtime HTTP call dependencies (not event-bus relationships):

```
beacon-mobile-app ──────────────────────────────────────────────────────────────┐
beacon-dashboard  ──────────────────────────────────────────────────────────────┤
                                                                                │
                    calls all services on every authenticated request           │
                    ┌───────────────────────────────────────────────────────────┘
                    │
                    ▼
             beacon-iam  ◄──── JWT validation (every request, all services)
                    │
          ┌─────────┼────────────────────────────────────────────────┐
          │         │                                                 │
          ▼         ▼                                                 ▼
    beacon-core   beacon-observability                       beacon-notification
          │         ▲                                                 ▲
          │         │ station lookup (dispatch creation)              │
          └─────────┘                                                 │
          │                                                           │
          └── publishes events via beacon-event-bus ─────────────────┘
                    (incident.approved, dispatch.accepted,
                     evacuation.advisory, etc.)

    beacon-support-bot ──► beacon-core (incident context)
                       ──► beacon-notification (escalation alerts)

    beacon-communication-platform ──► beacon-iam (user profiles)
                                  ──► beacon-event-bus (publish chat events)

    beacon-etl-flows ──► ClickHouse (ETL writes)
                     ──► beacon-core (alert DAGs create incidents)
    beacon-observability ──► ClickHouse (reads external_data)
```

**Critical dependency:** beacon-iam is on the hot path of every authenticated request across all services. JWT validation calls beacon-iam on every request. If beacon-iam degrades, all services degrade.

---

## Data Flow: A Citizen Reports a Fire

```
1. Citizen taps "Report Incident" in beacon-mobile-app
   └─► POST /api/core/v1/incidents  →  beacon-core
         ├─ Validates JWT via beacon-iam
         ├─ Creates incident record (status: "reported")
         ├─ Writes audit event via beacon-auditing → ClickHouse
         └─ Returns incident ID to mobile app

2. beacon-core publishes "incident.created" to GCP Pub/Sub
   └─► beacon-notification subscribes
         ├─ Retrieves FCM tokens from beacon-iam for nearby ERT members
         ├─ Sends push notifications via Firebase
         └─ Records notification delivery in its own DB

3. ERT member sees the notification on beacon-dashboard (polls /incidents every 30s)
   └─► Reviews incident, clicks "Approve"
         └─► PUT /api/core/v1/incidents/:id/approve  →  beacon-core
               ├─ Updates status to "approved"
               ├─ Invalidates Redis cache for this incident
               ├─ Writes audit event
               └─ Publishes "incident.approved" to Pub/Sub
                     └─► beacon-notification sends push to all area citizens
```

---

## Data Flow: A Dispatch Is Created

```
1. ERT member creates a dispatch on beacon-dashboard
   └─► POST /api/core/v1/dispatches  →  beacon-core
         ├─ Checks incident is approved (gate)
         ├─ Calls beacon-iam: GET /ground-staff/available  (nearest on-duty staff)
         ├─ Calls beacon-observability: GET /emergency-stations  (nearest station)
         ├─ Creates dispatch record (status: "pending_acceptance")
         ├─ Writes audit event
         └─ Publishes "dispatch.assigned" to Pub/Sub

2. beacon-notification consumes "dispatch.assigned"
   └─► Sends push + SMS to assigned ground-staff member (requires acknowledgement)

3. Ground staff member taps "Accept" in beacon-mobile-app
   └─► PUT /api/core/v1/dispatches/:id/accept  →  beacon-core
         ├─ Updates status to "accepted"
         ├─ Publishes SSE event via in-memory broker → beacon-dashboard updates live
         ├─ Publishes "dispatch.accepted" to Pub/Sub
         └─► beacon-notification sends push to incident commander

4. Ground staff GPS breadcrumbs stream to beacon-core
   └─► POST /api/core/v1/location/update  →  beacon-core
         └─ Writes to ClickHouse (location_updates table)
```

---

## Data Flow: An Evacuation Is Broadcast

```
1. ERT commander creates evacuation zone on beacon-dashboard
   └─► POST /api/core/v1/evacuations  →  beacon-core
         ├─ Checks incident is approved (gate)
         ├─ Creates evacuation zone with geographic boundary
         └─ Writes audit event

2. Commander clicks "Broadcast Advisory"
   └─► POST /api/core/v1/evacuations/:id/advisory  →  beacon-core
         ├─ Calls beacon-iam: GET /users/fcm-tokens  (all citizens in zone)
         ├─ Publishes "evacuation.advisory.broadcast" to Pub/Sub
         └─ Writes audit event

3. beacon-notification consumes the event
   └─► Sends push + SMS + email to every citizen in the evacuation zone
         └─ Citizens receive notification with evacuation routing in beacon-mobile-app
               └─ TomTom Routing API calculates route AWAY from the incident
```

---

## Infrastructure

All services run on **Google Kubernetes Engine (GKE)** in the `europe-west2` (London) region. The backend URL is `https://beacon-tcd.tech`.

| Component | Technology | Notes |
|---|---|---|
| Container orchestration | GKE (Autopilot) | All services deployed as Kubernetes Deployments |
| Container registry | Google Artifact Registry | Images pushed via `make gke-build-push` |
| Secret management | Kubernetes Secrets | Injected as env vars via `secretKeyRef` |
| Ingress | GKE Ingress + Cloud Load Balancer | TLS termination at the load balancer |
| Database | Cloud SQL (PostgreSQL 15) | Separate database per service; master + read replica |
| Cache | Cloud Memorystore (Redis 7) | Shared Redis instance; per-service key namespacing |
| Analytics store | Self-hosted ClickHouse | `audit_events` DB + `external_data` DB for ETL |
| Event bus | GCP Pub/Sub | 3 topics: `beacon-incidents`, `beacon-dispatches`, `beacon-evacuations` |
| Scheduled jobs | GCP Cloud Scheduler | Managed via beacon-scheduler library |
| ETL pipelines | Airflow on GKE (Helm) | CeleryExecutor, git-sync sidecar, Cloud SQL metadata DB |
| File storage | Google Cloud Storage | `beacon-vault` bucket for incident evidence files |
| Maps | TomTom Maps SDK | Tile rendering + routing for mobile app and dashboard |
| Push notifications | Firebase Cloud Messaging | Token management via beacon-iam |
| SMS | Twilio | Critical alerts and voice channels |
| Email | SendGrid | Non-urgent notifications and digests |
| AI | Vertex AI (Gemini Pro + Flash) | beacon-support-bot conversations and risk analysis |
| Identity provider | Keycloak (self-hosted on GKE) | Managed by beacon-iam |

### Docker Images

All Go microservices use a two-stage Docker build: `golang:1.24-alpine` builder → `gcr.io/distroless/static:nonroot` runtime. The distroless image has no shell, no package manager, and no OS utilities — the attack surface is the application binary and its dependencies only. The resulting images are typically under 25 MB.

---

## Shared Libraries

Two Go libraries are shared across multiple services. Both are versioned and published as Go modules within the `github.com/Response-To-City-Disaster/` organisation.

### beacon-auditing

Every Go microservice imports `beacon-auditing` and injects the client into its domain services via `SetAuditClient()`. The library accepts events onto a buffered in-process channel (non-blocking) and flushes them to ClickHouse in background goroutines. If ClickHouse is down, events are dropped with a warning — audit logging never fails the primary request path.

### beacon-event-bus

`beacon-core` uses `beacon-event-bus` to publish lifecycle events. `beacon-notification` uses it to subscribe. The library wraps the GCP Pub/Sub Go SDK with a CloudEvents-compatible event model, circuit breaker, exponential backoff retry, and a mock backend for hermetic testing.

---

## Authentication and Roles

All authentication is delegated to **Keycloak**, managed by beacon-iam. Keycloak issues JWTs; every other service validates them by calling `POST /api/iam/v1/auth/validate` on beacon-iam.

Four roles are in play across the platform:

| Role | Sign-in method | Capabilities |
|---|---|---|
| `Citizen` | Google OAuth (mobile app) or Apple OAuth | Report incidents, view safety info, receive push notifications, use AI support bot |
| `GroundStaff` | Username + password (mobile app) | Accept/reject dispatches, update incident from field, team chat, GPS tracking |
| `ERTMember` | Username + password (dashboard) | All citizen capabilities + create dispatches, create evacuations, view analytics |
| `Admin` | Username + password (dashboard) | Full platform access including user management, template management, all data |

RBAC is enforced at the HTTP layer in each service via Gorilla Mux sub-router middleware (`auth.RequireRole(...)`). There is no annotation magic — role checks are explicit in the router.

---

## Event Bus: Topics and Consumers

All async communication flows through GCP Pub/Sub via the `beacon-event-bus` library.

| Topic | Published by | Event Types | Consumed by |
|---|---|---|---|
| `beacon-incidents` | beacon-core | `incident.created`, `incident.approved`, `incident.closed` | beacon-notification, beacon-dashboard |
| `beacon-dispatches` | beacon-core | `dispatch.assigned`, `dispatch.accepted`, `dispatch.rejected`, `dispatch.completed`, `dispatch.timed_out` | beacon-notification |
| `beacon-evacuations` | beacon-core | `evacuation.created`, `evacuation.advisory.broadcast`, `evacuation.closed` | beacon-notification |

All events follow the CloudEvents 1.0 envelope format with a `type`, `source`, `subject` (entity ID), `time`, and `data` payload. Event payloads are JSON-serialized and published as Pub/Sub message bodies, with routing metadata stored as Pub/Sub message attributes.

**Downstream consumers** of these events (beacon-dashboard, mobile app) receive data through polling and SSE, not directly from Pub/Sub. The event bus is service-to-service only.

---

## Data Stores

### PostgreSQL

Each microservice owns its own PostgreSQL database on the shared Cloud SQL instance. The CQRS pattern is enforced platform-wide: **commands go to the master**, **queries go to the read replica**. Mixing these is a common source of replication-lag bugs — always check which connection pool a new query is using.

| Service | Database | Key tables |
|---|---|---|
| beacon-core | `core` | incidents, dispatches, evacuations, vault_files, incident_events |
| beacon-iam | `iam` | users, ert_members, ground_staff, sessions, devices, login_attempts |
| beacon-notification | `notification` | notifications, templates, acknowledgements, batch_jobs |
| beacon-observability | `observability` | hospitals, emergency_stations, camera_feeds, poi_locations |
| beacon-communication-platform | `communication` | messages, channels, social_posts, communities, voice_channels |
| beacon-support-bot | `support_bot` | conversations, escalations, wellbeing_checkins, resources |

### Redis

A shared Cloud Memorystore Redis instance. beacon-core uses it as a read-through cache for incident objects (1-hour TTL). Cache writes and invalidations are fail-open — if Redis is down, the service continues without caching. Never put business logic that depends on cache freshness; always treat Redis as a best-effort read optimisation.

### ClickHouse

Two logical databases on the shared ClickHouse cluster:

**`audit_events`** — Written by every microservice via `beacon-auditing`. One row per state mutation. TTL: 1 year. Partitioned by month. Used for compliance queries and the audit dashboard.

**`external_data`** — Written exclusively by `beacon-etl-flows`. Contains all time-series city data ingested from Irish public APIs: vehicle positions, weather observations, water levels, power outages, traffic incidents, etc. Read by `beacon-observability` to serve the real-time city state API. Read by `beacon-core` for GPS breadcrumb storage and post-incident location analytics.

---

## External API Integrations

| Provider | Used by | Purpose |
|---|---|---|
| **TomTom** | beacon-core, beacon-mobile-app, beacon-etl-flows | Geocoding, map tiles, turn-by-turn routing, traffic incidents |
| **Firebase (FCM)** | beacon-iam, beacon-notification | Push notification token registry and message delivery |
| **Twilio** | beacon-notification, beacon-communication-platform | SMS delivery, programmable voice channels |
| **SendGrid** | beacon-notification | Transactional email delivery |
| **Google Vertex AI** | beacon-support-bot | Gemini Pro (conversation), Gemini Flash (risk analysis), Embeddings (resource matching) |
| **Google OAuth** | beacon-iam, beacon-mobile-app | Citizen sign-in |
| **Apple Sign In** | beacon-iam, beacon-mobile-app | iOS citizen sign-in |
| **Keycloak** | beacon-iam | JWT issuance and validation, role management |
| **NTA GTFS-R** | beacon-etl-flows | Real-time bus and rail vehicle positions |
| **Met Éireann** | beacon-etl-flows | Weather observations and warnings |
| **ESB Networks** | beacon-etl-flows | Power outage data |
| **waterlevel.ie** | beacon-etl-flows | River and coastal flood sensor readings |
| **Irish Rail** | beacon-etl-flows | Train movement and punctuality |
| **Luas** | beacon-etl-flows | Tram forecast and arrival times |
| **Dublin Live** | beacon-etl-flows | News RSS for social feed integration |

---

## Local Development

Each service has its own `docker-compose.yml` (or `make run` target) for standalone local development. For full end-to-end local testing, bring up the shared infrastructure first:

```bash
# 1. Start shared infrastructure (PostgreSQL, Redis, ClickHouse, Keycloak, Pub/Sub emulator)
#    Each service's docker-compose.yml brings up its own dependencies.
#    For cross-service testing, start services individually:

cd beacon-iam    && docker-compose up -d && make migrate-up && make run &
cd beacon-core   && make migrate-up && make run &
# ... repeat for other services as needed

# 2. Point the mobile app and dashboard at localhost
#    Mobile app: flutter run --dart-define=BASE_URL=http://localhost:8080
#    Dashboard:  VITE_API_BASE_URL=http://localhost:8080 npm run dev

# 3. Use the Pub/Sub emulator for event bus testing
export EVENTBUS_EMULATOR_HOST=localhost:8085
gcloud beta emulators pubsub start --project=local-test
```

**Minimum viable local stack** (for beacon-core development):
```bash
cd beacon-core
docker-compose up -d    # starts Postgres, Redis, ClickHouse
make migrate-up
make run
```

### Prerequisites

| Tool | Version | Used by |
|---|---|---|
| Go | 1.24+ | All Go services and libraries |
| Flutter | 3.9+ | beacon-mobile-app |
| Node.js | 18+ | beacon-dashboard |
| Python | 3.10+ | beacon-etl-flows |
| Docker + Compose | Latest | All services |
| `golang-migrate` | Latest | All Go services (DB migrations) |
| `gcloud` CLI | Latest | GKE deployment, Pub/Sub emulator |

---

## Deployment

All Go microservices and the dashboard follow the same deployment pattern:

```bash
# 1. Authenticate with GCP
make gcp-auth

# 2. Build and push the Docker image to Artifact Registry
make gke-build-push

# 3. Apply the Kubernetes deployment manifest
make gke-deploy

# 4. Check rollout status
make gke-status
make gke-pods
make gke-logs
```

Kubernetes manifests are in `k8s/deployment.yml` in each repository. All sensitive configuration is injected from a Kubernetes Secret named `<service>-secrets` that must be populated before the first deployment.

The ETL platform is deployed via the Apache Airflow Helm chart:

```bash
cd beacon-etl-flows
helm upgrade --install beacon-airflow apache-airflow/airflow \
  -f deploy/values.yaml --namespace airflow
```

DAG updates are deployed automatically: a `git-sync` sidecar on each Airflow pod pulls the latest DAGs from the `beacon-etl-flows` main branch every 60 seconds.

---

## Code Graph Explorer

Interactive dependency graphs for all twelve repositories are available in [`graphs/`](graphs/index.html). Open `graphs/index.html` in a browser, or serve the directory locally:

```bash
python3 -m http.server 8080 --directory graphs/
# Open http://localhost:8080
```

Each graph shows the repository's node and edge count, detected community clusters with cohesion scores, critical execution flows, coupling risks, and test coverage gaps. Generated by **code-review-graph v1.27.0**.

---

## Per-Repository Documentation

Full production-grade documentation for each repository is in [`architecture-sync/`](architecture-sync/):

### Go Microservices
- [beacon-core](architecture-sync/beacon-core-readme.md) — Incident, dispatch, evacuation, vault, GPS
- [beacon-iam](architecture-sync/beacon-iam-readme.md) — Auth, JWT validation, FCM tokens, ERT workflows
- [beacon-notification](architecture-sync/beacon-notification-readme.md) — FCM, Twilio SMS, SendGrid email
- [beacon-observability](architecture-sync/beacon-observability-readme.md) — 11 city data domains, SSE streaming
- [beacon-communication-platform](architecture-sync/beacon-communication-platform-readme.md) — WebSocket chat, social, Twilio voice
- [beacon-support-bot](architecture-sync/beacon-support-bot-readme.md) — Vertex AI, risk analysis, escalation

### Go Libraries
- [beacon-auditing](architecture-sync/beacon-auditing-readme.md) — Async ClickHouse audit client
- [beacon-event-bus](architecture-sync/beacon-event-bus-readme.md) — GCP Pub/Sub wrapper
- [beacon-scheduler](architecture-sync/beacon-scheduler-readme.md) — GCP Cloud Scheduler adapter

### Data, Frontend, Mobile
- [beacon-etl-flows](architecture-sync/beacon-etl-flows-readme.md) — Apache Airflow DAGs, Irish public APIs
- [beacon-dashboard](architecture-sync/beacon-dashboard-readme.md) — React ERT dashboard
- [beacon-mobile-app](architecture-sync/beacon-mobile-app-readme.md) — Flutter citizen + ground staff app

### Architecture Deep-Dive
- [beacon-core architecture guide](architecture-sync/beacon-core-arch.md) — Senior engineer onboarding doc with code graph analysis.
