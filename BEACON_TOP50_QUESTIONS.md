# Beacon Platform — Top 50 Questions
### A Complete Study Guide for Onboarding, Architecture Review, and Advanced Engineering Thinking

> **Who is this for?**
> Whether you just joined the team as a junior intern or you are preparing for a senior engineering review, this document walks you through every major aspect of the Beacon platform — what each service does, how the services talk to each other, and then escalates into the deeper engineering and philosophical questions about Test-Driven Development, AI as a coding partner, and the future of software craftsmanship. Every answer is written to be self-contained. You should be able to read any question in isolation and come away with a clear mental model.

---

## Table of Contents

**Part 1 — Service Deep Dives (Q1–Q12)**
What each of the 12 Beacon services does, why it exists, and what problem it solves.

**Part 2 — Integration & Data Flow (Q13–Q25)**
How the services are wired together, how events travel through the system, and why each architectural decision was made.

**Part 3 — Infrastructure, DevOps & Data (Q26–Q33)**
Databases, Kubernetes, CI/CD, and the ETL layer.

**Part 4 — Advanced Software Engineering (Q34–Q50)**
Test-Driven Development, AI as a Copilot, Agentic AI, Vibe Coding, and the future of how we write software.

---

# Part 1 — Service Deep Dives

---

## Q1. What is the Beacon platform and why does it exist?

**The short answer:** Beacon is a city-scale emergency response platform built for Irish local authorities and emergency services. Its job is to coordinate every person and every piece of information involved in a city emergency — from the moment a citizen taps "Report Incident" on their phone to the moment a commander types "Incident Closed" hours later.

**The longer answer, told as a story:**

Imagine a fire breaks out on Dame Street in Dublin at 11 PM on a Friday. Without Beacon, here is what chaos looks like:
- A citizen calls 999. An operator writes it down.
- A commander radios the nearest available unit. Nobody knows if they are already on another call.
- Another citizen posts on Twitter. A different department sees it ten minutes later.
- An ambulance gets routed through a road that has been blocked for the last two hours due to a water main burst. Nobody told them.
- The whole district loses power. Nobody coordinated with the hospital that ED capacity just dropped.

**With Beacon, here is the same event:**
1. A citizen taps "Report Incident" in the mobile app. In 3 seconds, a structured incident record exists in `beacon-core`. Every on-duty ERT member gets a push notification from `beacon-notification`.
2. A commander reviews it on `beacon-dashboard` and approves. The nearest available ground-staff member gets a push + SMS from `beacon-notification` via `beacon-iam`'s FCM token registry.
3. The ground staff member accepts from `beacon-mobile-app`. The dashboard updates live via Server-Sent Events.
4. `beacon-observability` is already showing the traffic closure, the ESB outage, and the hospital ED wait time on the commander's screen because `beacon-etl-flows` has been pulling that data from public APIs every 5–10 minutes all day.
5. Citizens in the affected blocks get an evacuation advisory with routing that navigates them *away* from the fire.
6. A panicked citizen who cannot move gets calm guidance from `beacon-support-bot`, which automatically escalates to a human agent when it detects critical distress.
7. Every click, approval, dispatch, and escalation is recorded in an immutable audit trail via `beacon-auditing` in ClickHouse.

That is what Beacon does. It is twelve services working as one organism to coordinate a city in crisis.

---

## Q2. What does `beacon-core` do, and why is it called the "operational brain"?

**Simple answer for a junior intern:** `beacon-core` is the source of truth for everything happening during an emergency. If you need to know "is this incident approved?", "who is dispatched where?", or "has an evacuation been ordered for this area?" — the answer is always in `beacon-core`.

**Technical detail:**

`beacon-core` manages five domains:

| Domain | What it is | Think of it as |
|---|---|---|
| **Incidents** | The emergency record itself (fire, flood, SOS, etc.) with a strict state machine: `reported → approved → active → closed` | The master ticket for an emergency |
| **Dispatches** | The assignment of a ground-staff team to an approved incident, tracked through `pending_acceptance → accepted → in_progress → completed` | The work order sent to the field |
| **Evacuations** | Geographic zones attached to an incident. Can broadcast advisory messages to all citizens in the zone | The safe-evacuation instruction |
| **Vault** | A Google Cloud Storage-backed document store where photos and PDFs are attached to incidents | An evidence locker |
| **Location Tracking** | GPS breadcrumbs from ground-staff devices during an active dispatch, stored in ClickHouse | A map of where responders actually went |

**The approval gate — the single most important rule in the whole codebase:**
> You cannot dispatch responders, order an evacuation, or publish events to the event bus unless the incident is first approved.

This sounds obvious, but it means this check is enforced in at least **four places** in the codebase: `CreateDispatch`, `CreateEvacuation`, `PublishIncidentCreated`, and `PublishIncidentClosed`. If you ever change what "approved" means, you need to audit all four.

**Architecture pattern:** `beacon-core` uses Hexagonal Architecture + CQRS. This means:
- Commands (write operations) go to the PostgreSQL *master* connection.
- Queries (read operations) go to the PostgreSQL *slave/read-replica*.
- All HTTP handling is in `internal/http/`, all business logic is in `internal/domain/`, and all external integrations are in `internal/adapters/`.

---

## Q3. What does `beacon-iam` do, and why is it the most critical service in the platform?

**Simple answer:** `beacon-iam` is the gatekeeper. Every person using Beacon — citizen, ground-staff, ERT member, admin — must prove their identity through beacon-iam before doing anything else. And every other service in the platform validates that proof by calling beacon-iam on every single request.

**Why it is the most critical service:**

The README says it bluntly: "beacon-iam is on the hot path of every authenticated request across all services. JWT validation calls beacon-iam on every request. **If beacon-iam degrades, all services degrade.**"

This is not hyperbole. When a citizen reports an incident, `beacon-core` receives the request and immediately calls `POST /api/iam/v1/auth/validate` on `beacon-iam` before doing anything else. If that call fails or takes 3 seconds, every incident report takes at least 3 seconds. Multiply that by every API call across all services at peak usage.

**What beacon-iam actually does:**

1. **Authentication** — It wraps Keycloak (an open-source identity provider). Citizens sign in with Google OAuth or Apple Sign In. Ground staff and ERT members use username/password. beacon-iam handles the OAuth dance, token exchange, and stores user profiles.

2. **JWT Validation (service-to-service)** — Every other Beacon service calls `/auth/validate` on every incoming HTTP request. beacon-iam checks the token signature against Keycloak's public keys and returns decoded claims (user ID, role, email). This is the highest-traffic operation in the entire platform.

3. **Ground-Staff Roster** — `beacon-core` calls `/ground-staff/available` during dispatch creation to find on-duty staff who are not already assigned to another dispatch.

4. **FCM Token Registry** — The mobile app registers its Firebase Cloud Messaging token with beacon-iam. When `beacon-core` needs to send push notifications, it calls beacon-iam to get the recipient's current FCM tokens.

5. **ERT Approval Workflow** — Citizens can apply for ERT membership. Admins review and approve/reject applications. On approval, beacon-iam calls the Keycloak Admin API to promote the user's role.

6. **Login Lockout** — Failed logins are counted per user. After 5 failures (configurable), the account locks for 30 minutes to prevent brute-force attacks.

**Roles in the system:**

| Role | How they sign in | What they can do |
|---|---|---|
| `Citizen` | Google/Apple OAuth | Report incidents, view safety info, use AI bot |
| `GroundStaff` | Username + password | Accept/reject dispatches, GPS tracking, team chat |
| `ERTMember` | Username + password | Create dispatches and evacuations, view analytics |
| `Admin` | Username + password | Full access including user management |

---

## Q4. What does `beacon-notification` do, and how does it know when to send a message?

**Simple answer:** `beacon-notification` is the broadcast hub. It listens to events published by `beacon-core` and translates them into push notifications (Firebase), SMS (Twilio), or emails (SendGrid) sent to the right people at the right time.

**The key design principle — "channel-agnostic business logic":**

The notification domain does not care whether a message ends up on your phone screen, in your SMS inbox, or your email. The delivery adapters (FCM, Twilio, SendGrid) are plugged in at startup. The business logic just says "send this notification of this type to this user" — it does not say how.

**How it knows when to send:**

`beacon-notification` runs a subscriber that listens to GCP Pub/Sub topics published by `beacon-core`:

| Event arriving from Pub/Sub | What notification gets sent |
|---|---|
| `incident.created` | Push to nearby ERT members and admins |
| `incident.approved` | Push to all active citizens in the affected area |
| `dispatch.assigned` | Push **and** SMS to the assigned ground-staff member (must be acknowledged) |
| `dispatch.accepted` | Push to the incident commander |
| `evacuation.advisory.broadcast` | Push + SMS + email to every citizen in the evacuation zone |

**The acknowledgement system:**
For critical notifications (dispatch assignments, mandatory evacuations), just sending the message is not enough. The service creates an Acknowledgement requirement — the recipient must explicitly confirm they received and understood the message. This is tracked in the database. ERT commanders can query who has and has not acknowledged a dispatch.

**Templates:**
Notification content is not hardcoded. It lives in the database as templates keyed by `(type, locale)`. An admin can change the wording of "You have been dispatched to incident {{incident_id}} at {{location}}" without touching code. Variables are substituted at render time from the event payload.

**Batch jobs for large-scale broadcasts:**
When an evacuation zone covers thousands of citizens, sending 10,000 push notifications simultaneously would hit Firebase rate limits. The batch domain staggers delivery using `beacon-scheduler`, which defers messages across time windows to stay within provider limits.

---

## Q5. What does `beacon-observability` do, and what is the difference between it and `beacon-etl-flows`?

**Simple answer:** `beacon-observability` is the window into the live city. It answers questions like "Which roads are closed?", "Is the nearest hospital on diversion?", "Is there flooding at the Liffey sensors?", "Where are all the ambulances right now?" — and it serves those answers fast, via REST and real-time Server-Sent Events.

**`beacon-etl-flows` vs. `beacon-observability` — what is the difference?**

Think of it like a library:
- `beacon-etl-flows` is the **librarian** who goes out and collects books (data from Irish public APIs like NTA, Met Éireann, ESB Networks), processes them, and puts them on the shelves (ClickHouse).
- `beacon-observability` is the **library desk** — when you need a book, you ask the desk, and it retrieves it from the shelves quickly and hands it to you.

They are deliberately separated. A failure in the ETL pipelines (maybe a public API is down) does not take down the observability service — it just means the data gets stale. And the observability service never needs to know about GTFS Protobuf parsing or Met Éireann XML formats.

**The 11 data domains:**

1. **Traffic** — Road closures, accidents, congestion levels from TomTom
2. **Public Transport** — Live bus (NTA), Luas tram, and Irish Rail positions
3. **Weather** — Current conditions and warnings from Met Éireann (6 Dublin stations)
4. **Water Levels** — River gauges and coastal sensors from waterlevel.ie
5. **Power Grid** — Active outages from ESB Networks
6. **Hospitals** — ED wait times, bed availability, diversion status
7. **Ambulances** — Live positions, availability, and closest-unit queries
8. **Emergency Services** — Police/fire station locations and status
9. **Camera Feeds** — CCTV metadata and thumbnails
10. **Geographic** — Country/city reference data
11. **Social Media** — Aggregated emergency-related social posts

**How `beacon-core` uses it:**
When a dispatcher creates a new dispatch assignment, `beacon-core` calls `GET /api/observability/v1/emergency-stations` to find the nearest emergency station to the incident location and pre-populate the dispatch with those coordinates. Without this call, dispatchers would have to look up station coordinates manually.

**SSE streaming:**
For time-critical categories (traffic, water levels, ambulances), the service provides Server-Sent Events endpoints. The ERT dashboard opens a persistent HTTP connection to these endpoints and receives updates pushed from the server as they arrive — no polling needed.

---

## Q6. What does `beacon-support-bot` do, and why is it more than just a chatbot?

**Simple answer:** `beacon-support-bot` is an AI-powered crisis counsellor. When a citizen is trapped in a flood or panicking in a fire evacuation, the bot provides calm, accurate guidance generated by Google's Gemini model. But it also silently monitors every message for signs of psychological distress, and when it detects someone who needs a real human, it automatically escalates to a human support agent.

**Why it is more than a chatbot:**

Most chatbots just respond to what you type. `beacon-support-bot` does four things simultaneously:

1. **Conversation generation** — Gemini Pro generates empathetic, contextually-aware responses. The model is given a system prompt and, importantly, the current status of the incident the user is affected by (fetched live from `beacon-core`). This prevents the bot from giving outdated or generic information.

2. **Risk analysis (happening in parallel, invisibly)** — Gemini Flash (faster and cheaper than Pro) analyses every single user message for distress indicators: panic, expressions of physical danger, mentions of self-harm or suicidal ideation. It produces a risk severity score: `low / moderate / high / critical`. The user never sees this analysis.

3. **Automatic escalation** — If the risk score hits `high` or `critical`, the system immediately (without any human pressing a button):
   - Notifies an available human support agent via `beacon-notification` (push + SMS)
   - Sends the agent the full transcript and risk assessment
   - Informs the user that a human is being connected
   - Pauses AI responses so the agent can take over

4. **Wellbeing follow-ups** — After a major incident closes, the service schedules check-in conversations with involved citizens at 24 hours, 3 days, and 7 days. These follow-ups run the same risk analysis — delayed stress responses are just as real as acute ones, and the system is designed to catch them.

**Why Vertex AI was chosen:**
The service uses three different models for different tasks because they have different cost/speed/quality tradeoffs:
- **Gemini Pro**: Highest quality, used for conversation (1–3 second response time acceptable)
- **Gemini Flash**: Faster and cheaper, used for risk analysis on every message (needs to be near-instant)
- **Embeddings model**: Semantic search to match user queries to the most relevant self-help resources

---

## Q7. What does `beacon-communication-platform` do, and who uses it?

**Simple answer:** `beacon-communication-platform` is the messaging infrastructure for the whole platform. It powers WebSocket team chat for ground staff and ERT coordinators, community social boards for citizens, and Twilio voice channels for situations where typing is not fast enough.

**Who uses what:**

| User type | Communication features they use |
|---|---|
| **ERT members** | Incident-scoped team channels (one channel auto-created per approved incident), broadcast channels for platform-wide alerts |
| **Ground staff** | Team channels with other responders and dispatch operators, voice channels when typing is impractical |
| **Citizens** | Community social posts (share information, ask questions), news feed (Dublin Live articles surfaced alongside citizen posts) |

**The four domains:**

1. **Message domain** — Real-time chat with delivery receipts (sent/delivered/read), typing indicators, message editing and soft deletion, and moderation flagging. Messages are stored in PostgreSQL and replayed to reconnecting clients so no history is lost across disconnects.

2. **Channel domain** — Persistent conversation rooms. Incident-scoped channels are automatically created when a new incident is approved and automatically archived when it is closed. Team channels are permanent. Broadcast channels are one-way (admin posts only, all users can read).

3. **Social domain** — Citizen community boards with posts, comments, emoji reactions, and news article integration. ERT members can moderate and remove misinformation.

4. **Voice Channel domain** — Twilio Programmable Voice conference rooms. Managed through the service: create room → invite participants → terminate room. Logs who joined and left each session.

**WebSocket details:**
The service upgrades HTTP connections to WebSocket using gorilla/websocket. On connection, it immediately sends the last 50 messages from each active channel the user belongs to ("message replay"), then streams new messages, typing indicators, read receipts, and presence events in real time.

---

## Q8. What does `beacon-etl-flows` do, and what are the two types of DAGs?

**Simple answer:** `beacon-etl-flows` is a collection of automated data pipelines built on Apache Airflow. Think of it as a robot that wakes up every few minutes, fetches live city data from Irish government and third-party APIs, cleans and transforms it, and loads it into ClickHouse. A second group of robots then watches that data and triggers emergency incidents when danger thresholds are crossed.

**The two types of DAGs:**

### Type 1: ETL DAGs (13 pipelines)
These pull data from external sources and load it into ClickHouse. They follow a strict `extract → transform → load` pattern.

| DAG | Data source | Frequency |
|---|---|---|
| `gtfs_realtime_vehicle_positions_dag` | NTA GTFS-R (all Dublin buses) | Every 5 min |
| `gtfs_realtime_trip_updates_dag` | NTA GTFS-R (arrival predictions) | Every 5 min |
| `luas_forecast_dag` | Luas REST API (tram arrivals) | Every 5 min |
| `irish_rail_dag` | Irish Rail (train movements) | Every 5 min |
| `met_eireann_dag` | Met Éireann (weather observations) | Hourly |
| `weather_warnings_dag` | Met Éireann (active warnings) | Every 30 min |
| `water_levels_dag` | waterlevel.ie (river/coastal gauges) | Every 15 min |
| `esb_power_outages_dag` | ESB Networks (outage map) | Every 10 min |
| `tomtom_traffic_incidents_dag` | TomTom (road incidents) | Hourly |
| `dublin_live_news_dag` | Dublin Live RSS | Every 15 min |
| `gtfs_static_dag` | NTA (full schedule reference data) | Daily at 03:00 |
| `poi_locations_dag` | TomTom (emergency POI reference) | Monthly |

### Type 2: Alert DAGs (5 monitors)
These watch the data already loaded by ETL DAGs and create incidents in `beacon-core` when thresholds are exceeded.

**The alert DAG pattern (same for all 5):**
1. Query ClickHouse for records exceeding the alert threshold
2. For each match, check a PostgreSQL `incident_alerts` table to see if an incident was already raised (deduplication)
3. If no existing incident → call `beacon-core`'s incident creation API
4. Record the mapping in `incident_alerts` so future DAG runs do not create duplicates

| Alert DAG | What it watches | Threshold |
|---|---|---|
| `alert_water_levels_dag` | Water sensor readings | >80% = watch, >100% = active flood |
| `alert_power_outages_dag` | Power outages | Outages above configurable customer-count |
| `alert_severe_weather_dag` | Weather warnings | Status Yellow, Orange, Red events |
| `alert_traffic_incidents_dag` | Traffic incidents | Closures on primary/emergency routes |
| `alert_weather_warnings_dag` | Specific weather events | Thunderstorms, freezing fog, black ice |

---

## Q9. What does `beacon-dashboard` do, and what makes it technically interesting?

**Simple answer:** `beacon-dashboard` is the web interface for ERT members and administrators. It is their command centre during an emergency — live incident feed, dispatch coordination, evacuation management, team chat, analytics, and user management, all in one single-page application.

**Tech stack:**
React 18 + TypeScript 5, built with Vite, styled with Tailwind CSS, UI primitives from Radix UI (shadcn/ui), charts from Recharts, maps from TomTom Maps SDK.

**What makes it technically interesting:**

1. **Atomic Design hierarchy** — Every component lives at exactly one level of abstraction: Atoms (badges, status dots), Molecules (incident cards, stat cards), Organisms (incident tables, dispatch panels), and Pages. Pages fetch data. Nothing below them does. This prevents the common React antipattern of data being fetched in nested components, which causes waterfall loading.

2. **No global state library** — There is no Redux, no Zustand, no Jotai. State is managed with React's built-in `useState`, `useReducer`, and custom hooks per feature. The auth state singleton (`authService`) is a module, not React context. This keeps things simple and predictable.

3. **URL-first navigation** — Pages must be independently loadable from a direct URL. No page depends on data passed through navigation state from a previous page. This means refreshing any page always works, and deep links always work.

4. **Dual real-time strategies** — Polling (every 30 seconds) for the incident feed, SSE subscription for dispatch status. This is a deliberate tradeoff: polling is simple and reliable for the feed (stale by 30s is acceptable), but dispatch status needs sub-second updates when a ground-staff member accepts from the field.

5. **Request utility with silent token refresh** — All API calls go through `src/utils/request.ts`. If a call returns 401 (token expired), the utility automatically calls `/auth/refresh`, gets a new access token, and retries the original request — transparently to the calling component.

---

## Q10. What does `beacon-mobile-app` do, and how does it serve two completely different user types on one codebase?

**Simple answer:** `beacon-mobile-app` is the Flutter app for both citizens and ground staff. Citizens use it to report emergencies, see what is happening on a live map, receive evacuation routing, and talk to the AI bot. Ground staff use it to accept dispatches, navigate to incidents, and coordinate with their team.

**How it serves two user types on one codebase:**

The app uses role-based navigation. After login, `beacon-iam` returns the user's role in the JWT claims. The app's router (`go_router`) reads that role and renders different navigation structures:

- **Citizens** see: Incident feed, Map, Safety dashboard (hospitals/weather/evacuations), AI Support Bot chat, Notifications, Profile
- **Ground Staff** see everything citizens see, plus an additional "Ground Staff" tab with: Duty toggle, Pending dispatch assignments, Team chat, Navigation to incident site, Dispatch history

The architecture is Clean Architecture per feature — each feature module has its own `data/`, `domain/`, and `presentation/` layers. Riverpod providers manage all async state. This means the ground-staff dispatch feature is completely self-contained and does not affect the citizen incident-reporting feature.

**Notable features:**

- **SOS button** — A single tap creates a highest-priority incident from the user's current location instantly, skipping any form. This is critical for someone who cannot stop and type.
- **Evacuation routing** — TomTom Routing API calculates a route *away* from the emergency, not toward it. This is `calculateEvacuationRoute`, distinct from the `calculateRoute` used by ground staff navigating *to* the scene.
- **FCM token rotation** — Firebase periodically rotates push notification tokens. The app detects this and automatically re-registers the new token with `beacon-iam`.
- **Offline detection** — `connectivity_plus` monitors network status. The app degrades gracefully when offline rather than showing cryptic errors.

---

## Q11. What does `beacon-auditing` do, and why is it designed to never block the caller?

**Simple answer:** `beacon-auditing` is a Go library imported by every Beacon microservice. Its job is to record every meaningful action taken by every user — "who did what to which record, and when" — into an immutable ClickHouse database. It is the compliance and accountability trail for the entire platform.

**Why it must never block the caller:**

Imagine an ERT member approves an incident. That approval triggers: a PostgreSQL write, a Redis cache invalidation, an event bus publish, a Keycloak JWT validation, and potentially an audit log write to ClickHouse. If the audit write were synchronous, and ClickHouse happened to be slow at that moment, the ERT member's browser would hang waiting for an analytics write to complete before showing them that the approval succeeded. That is unacceptable in an emergency.

**The design:**

```
Caller goroutine → Validation (synchronous, no I/O) →
Non-blocking channel send (capacity: 1000 events) →
Background goroutine drains the queue →
Batch assembler (accumulates up to 100 events OR waits 5 seconds) →
ClickHouse batch INSERT
```

The `LogAuditEvent()` call validates required fields synchronously (a few microseconds of CPU), then places the event on an in-process channel and returns. It does not wait for ClickHouse. The background goroutine handles all the I/O.

If ClickHouse goes down, the in-process channel fills up. When it is full, `LogAuditEvent()` returns `ErrBufferFull`. The caller is expected to log this and continue serving the primary request — the audit trail has a small gap, but the user-facing operation succeeds. That is the deliberate design choice: availability over completeness.

**The audit schema:**
Every event captures: `ServiceName`, `ActorID`, `ActorRole`, `Action` (in `domain.verb` format like `incident.approved`), `ResourceType`, `ResourceID`, `Metadata` (arbitrary JSON), `IPAddress`, and `OccurredAt`. The table has a 1-year TTL enforced by ClickHouse itself — no maintenance job needed.

---

## Q12. What do `beacon-event-bus` and `beacon-scheduler` do, and why are they Go libraries rather than services?

**`beacon-event-bus`** is a Go library that wraps the GCP Pub/Sub SDK. Instead of every service writing its own Pub/Sub boilerplate (error handling, retries, CloudEvents formatting, mock testing), this library provides a single consistent interface used by all services.

Key features:
- **CloudEvents 1.0 envelope** — All events have a standard `type`, `source`, `subject`, `time`, `data`, `priority`, and `traceID`. This makes events self-describing and consistent across all topics.
- **Circuit breaker** — After N consecutive publish failures, the breaker opens and calls fail immediately without a network attempt. This prevents goroutines from piling up waiting for a degraded Pub/Sub.
- **Exponential backoff retry** — Transient failures are automatically retried with increasing delays.
- **Mock backend** — A `mock` sub-package provides an in-memory event bus for unit tests. No real GCP credentials needed. Tests can assert "was event X published to topic Y with payload Z?"

**`beacon-scheduler`** is a Go library that wraps the GCP Cloud Scheduler API. It is used by `beacon-notification` to defer time-sensitive operations: scheduling wellbeing follow-up check-ins at 24 hours/3 days/7 days after incident close, staggering batch notification delivery to avoid rate limits, and triggering recurring ERT summary reports.

**Why are these libraries and not services?**

A service has its own process, its own HTTP server, its own database, its own deployment. That overhead makes sense for complex, stateful, multi-consumer domains. An event-bus wrapper and a scheduler adapter are thin, stateless utilities with no domain logic of their own. Making them libraries means:
- Zero network hops — the calling service calls the library directly, not over HTTP
- Zero additional deployment surface to manage
- Simple dependency injection (the interface can be swapped for a mock in tests)
- One place to fix a bug in retry logic or event envelope format, rather than fixing it in 6 services

---

# Part 2 — Integration & Data Flow

---

## Q13. How do services talk to each other? What are the two communication patterns?

Beacon uses **two communication patterns**, chosen based on whether the communication needs to be synchronous (immediate response required) or asynchronous (fire and forget).

### Pattern 1: Synchronous HTTP REST calls (service-to-service)

Used when the caller needs an answer before it can continue:
- `beacon-core` calls `beacon-iam` to validate every JWT (needs "is this user authenticated and what is their role?" before doing anything)
- `beacon-core` calls `beacon-iam` to get ground-staff availability (needs "who is on duty?" before creating a dispatch)
- `beacon-core` calls `beacon-observability` to find the nearest station (needs "where is the nearest fire station?" before creating a dispatch)
- `beacon-support-bot` calls `beacon-core` to fetch incident context (needs "what is actually happening right now?" before crafting a response)

### Pattern 2: Asynchronous events via GCP Pub/Sub (beacon-event-bus)

Used when the publisher does not need to wait for the consumer:
- `beacon-core` approves an incident → publishes `incident.approved` to Pub/Sub → `beacon-notification` subscribes and sends push notifications. `beacon-core` does not wait. Whether the push notification was delivered in 100ms or 2 seconds does not affect the approval operation.

**Why this separation matters:**
If `beacon-notification` went down, the async pattern means `beacon-core` keeps working — incidents can still be created and approved; the push notifications just queue up in Pub/Sub until `beacon-notification` comes back. If the sync calls went through Pub/Sub, a dispatch creation would become fire-and-forget, and `beacon-core` would not know if staff availability was actually checked.

---

## Q14. What are the three Pub/Sub topics, and who publishes/consumes each one?

Beacon uses exactly **three GCP Pub/Sub topics**, one per major operational domain:

| Topic | Published by | Key event types | Consumed by |
|---|---|---|---|
| `beacon-incidents` | beacon-core | `incident.created`, `incident.approved`, `incident.escalated`, `incident.closed` | beacon-notification, beacon-dashboard |
| `beacon-dispatches` | beacon-core | `dispatch.assigned`, `dispatch.accepted`, `dispatch.rejected`, `dispatch.completed`, `dispatch.timed_out` | beacon-notification |
| `beacon-evacuations` | beacon-core | `evacuation.created`, `evacuation.advisory.broadcast`, `evacuation.closed` | beacon-notification |

**Important nuance:** `beacon-dashboard` and `beacon-mobile-app` do not consume Pub/Sub directly. They receive updates through polling (every 30 seconds for the incident feed) and SSE (for dispatch status). The event bus is service-to-service only.

**CloudEvents format:** Every event has a standard envelope — `type` (domain.verb), `source` (the publishing service), `subject` (the entity UUID), `time`, `priority`, `traceID`, and a JSON `data` payload. This makes events self-describing: a consumer can filter on `type: incident.approved` without parsing the payload.

---

## Q15. Walk me through the complete journey of a citizen reporting a fire — from tap to dispatch.

This is the most important data flow to understand. It crosses six services.

```
1. Citizen taps "Report Incident" in beacon-mobile-app (Flutter)
   └─► POST /api/core/v1/incidents  →  beacon-core

2. beacon-core receives the HTTP request:
   ├─ Calls POST /api/iam/v1/auth/validate on beacon-iam
   │   └─ beacon-iam returns: { user_id, role: "Citizen", email }
   ├─ Validates request payload (type, description, location)
   ├─ Creates incident record in PostgreSQL (status: "reported")
   ├─ Writes audit event via beacon-auditing → ClickHouse (async, non-blocking)
   └─ Returns HTTP 201 with incident ID to mobile app

3. beacon-core (in the incident.Service.CreateIncident method):
   ├─ Calls beacon-iam: GET /users/{id}/fcm-tokens  (get citizen's own tokens)
   └─ Publishes "incident.created" to GCP Pub/Sub (beacon-incidents topic)

4. beacon-notification (running a Pub/Sub subscriber) receives "incident.created":
   ├─ Calls beacon-iam: GET /users/fcm-tokens?role=ERTMember&area={zone}
   ├─ Renders notification content from template ("New incident reported: {{type}} at {{location}}")
   ├─ Sends FCM push notifications to all nearby on-duty ERT members
   └─ Records Notification records in its own PostgreSQL

5. ERT member sees notification on beacon-dashboard (also polling /incidents every 30s):
   └─► Clicks "Approve" → PUT /api/core/v1/incidents/{id}/approve → beacon-core
       ├─ Updates incident status from "reported" → "approved" in PostgreSQL
       ├─ Invalidates Redis cache for this incident ID
       ├─ Writes audit event to ClickHouse
       └─ Publishes "incident.approved" to Pub/Sub

6. ERT member creates a dispatch:
   └─► POST /api/core/v1/dispatches → beacon-core
       ├─ Checks incident is approved (approval gate enforced)
       ├─ Calls beacon-iam: GET /ground-staff/available  (who is on duty and free?)
       ├─ Calls beacon-observability: GET /emergency-stations (nearest station?)
       ├─ Creates dispatch record (status: "pending_acceptance")
       ├─ Writes audit event
       └─ Publishes "dispatch.assigned" to Pub/Sub

7. beacon-notification consumes "dispatch.assigned":
   └─► Sends push + SMS to assigned ground-staff member
       (this notification requires explicit acknowledgement)

8. Ground staff taps "Accept" in beacon-mobile-app:
   └─► PUT /api/core/v1/dispatches/{id}/accept → beacon-core
       ├─ Updates dispatch status to "accepted"
       ├─ Publishes SSE event via in-memory broker → beacon-dashboard updates live
       ├─ Publishes "dispatch.accepted" to Pub/Sub
       └─ beacon-notification sends push to incident commander
```

---

## Q16. What is CQRS and why does Beacon use it everywhere?

**Simple answer for a junior intern:** CQRS stands for Command Query Responsibility Segregation. It means "split your write operations and your read operations into separate paths." In practice, all writes go to the PostgreSQL master database and all reads go to the PostgreSQL slave (read replica).

**Why this matters:**

In most web applications, writes (INSERT, UPDATE, DELETE) and reads (SELECT) compete for the same database connection. During peak load, a heavy analytics query can slow down an incident approval because both share the same server.

With CQRS:
- Commands (ApproveIncident, CreateDispatch) → master database (strong consistency, can handle conflict resolution)
- Queries (ListIncidents, GetIncidentById) → slave database (eventually consistent, optimised for reads, horizontally scalable)

**Where to watch out:**
The most common bug in a CQRS system is accidentally routing a read to the master or a write to the slave. In `beacon-core`, this would happen if someone added a new query handler and accidentally used the master connection pool instead of the slave. This is why the codebase has explicit `MasterRepository` and `SlaveRepository` interfaces — the type system makes the separation obvious, and code review should catch it.

---

## Q17. What is Hexagonal Architecture and why does it make testing easier?

**Simple answer:** Hexagonal Architecture (also called Ports and Adapters) is a way of organising code so that your business logic has no idea how it talks to the outside world. The domain (business rules) sits in the middle. Everything that connects to the outside (database, HTTP, event bus, external APIs) is an "adapter" that plugs into the domain through an "interface" (called a "port").

**In Beacon's code:**

```
Domain layer (beacon-core/internal/domain/incident/service.go):
  - Knows about: incidents, dispatches, state machines, business rules
  - Does NOT know about: SQL, HTTP, Redis, GCP, Pub/Sub

Adapters layer (beacon-core/internal/adapters/):
  - database/   → SQL queries for PostgreSQL
  - eventbus/   → GCP Pub/Sub publish calls
  - microservices/ → HTTP calls to beacon-iam, beacon-observability
  - storage/gcs/ → Google Cloud Storage calls

HTTP layer (beacon-core/internal/http/):
  - Translates HTTP requests into domain method calls
  - Returns domain results as HTTP responses
```

**Why testing is easier:**

When the incident service wants to test "does approving an incident invalidate the Redis cache and publish an event?", it does not need a real Redis server or a real GCP Pub/Sub connection. It creates a mock Redis adapter and a mock event bus (beacon-event-bus ships a mock sub-package), injects them into the service, runs the approval, and asserts:
- `mockRedis.DeleteCalls()` = 1 with the right key
- `mockEventBus.GetPublishedEvents("beacon-incidents")` contains `incident.approved`

The business logic is tested in isolation. The adapters are tested separately with integration tests against real external systems.

---

## Q18. How does the Redis cache work, and why is it designed to fail silently?

Redis sits between `beacon-core`'s query handlers and PostgreSQL. When the dashboard polls for incident details, `beacon-core` first checks Redis. If the incident is cached (TTL: 1 hour), it returns from Redis without a PostgreSQL query. On a miss, it queries PostgreSQL and writes the result to Redis for the next caller.

**Fail-open design:**

Cache writes and invalidations fail silently — if Redis is down, `beacon-core` logs a warning and continues. It does not return an error to the caller. This means:

- A cache miss on a down Redis just means "serve directly from PostgreSQL" — slower but correct.
- A failed cache invalidation after an `UpdateIncident` means the stale version will be served for up to 1 hour.

**The critical rule this implies:** Never add business logic that depends on the cache being fresh. If code anywhere in `beacon-core` ever does "check Redis to decide whether to allow an operation," that is a bug waiting to happen. Redis is a read optimisation only, not a consistency mechanism.

---

## Q19. How does authentication work end-to-end across the platform?

**The flow for a citizen signing in with Google:**

1. Mobile app launches a browser-based Google OAuth consent screen using `flutter_web_auth_2` with the `beaconflutterapp://` URL scheme as callback.
2. Google redirects to `beacon-iam`'s `/auth/oauth/google` endpoint with an authorization code.
3. `beacon-iam` exchanges the code for a Google ID token, creates or retrieves the citizen's Beacon profile, and calls Keycloak to issue a Beacon JWT.
4. The JWT is returned to the mobile app, which stores it in `flutter_secure_storage` (encrypted on-device storage).
5. Every subsequent API call injects the JWT in the `Authorization: Bearer ...` header via `AuthInterceptor` in Dio.
6. When the JWT is within 60 seconds of expiry, `AuthInterceptor` calls `/auth/refresh` to get a new token before making the API call — the user never sees a 401.

**How other services validate JWTs:**

Every service (beacon-core, beacon-observability, beacon-notification, etc.) calls `POST /api/iam/v1/auth/validate` on beacon-iam with the token from the incoming request. beacon-iam verifies the signature against Keycloak's public keys and returns the decoded claims. The calling service then checks the `role` claim against its RBAC rules.

**RBAC enforcement:**

In `beacon-core` (and all other services), RBAC is explicit in the router:
```go
// Only ERT members and admins can create dispatches:
ertRouter := router.NewRoute().Subrouter()
ertRouter.Use(auth.RequireRole("ERTMember", "Admin"))
ertRouter.HandleFunc("/dispatches", handlers.CreateDispatch).Methods("POST")
```
There is no annotation magic, no framework scanning. The role check is literally in the router function.

---

## Q20. What is the in-memory SSE broker in `beacon-core`, and why is it a scaling risk?

`beacon-core` uses Server-Sent Events to push real-time dispatch status updates to the ERT dashboard. When a ground-staff member accepts a dispatch from the mobile app, the dashboard must update immediately — not wait for the next 30-second poll.

The SSE broker (`internal/sse/broker.go`) is an in-memory Go `map` protected by a `sync.RWMutex`. It stores all live SSE client connections, keyed by dispatch ID. When a dispatch status changes, the broker iterates the map and sends the event to every connected client.

**The scaling risk:**

If two instances of `beacon-core` run (for load balancing), a dashboard client connected to Instance A will miss events published by Instance B. The in-memory broker has no cross-instance visibility.

**The fix (not yet implemented):** Replace the in-memory broker with Redis Pub/Sub. Both instances subscribe to the same Redis channel. When either instance updates a dispatch, it publishes to Redis. Both instances receive it and can push it to their connected SSE clients.

**Junior intern translation:** Imagine a whiteboard in an office. When someone writes on it, everyone in that room can see it. But if you have two offices (two `beacon-core` instances) each with their own whiteboard, writing on Office A's whiteboard means people in Office B never see it. Redis Pub/Sub is like having one shared whiteboard that both offices can see.

---

## Q21. How does `beacon-etl-flows` prevent duplicate incidents from being created?

When a water level sensor exceeds its alert threshold, the alert DAG queries ClickHouse and finds it has breached the flood threshold. It should create one incident in `beacon-core` — not one every 15 minutes as long as the sensor stays above threshold.

**The deduplication mechanism:**

Each alert DAG checks a PostgreSQL table called `incident_alerts` before calling `beacon-core`. This table maps `(alert_type, external_id)` → `incident_id`. Before creating an incident:

1. Query: `SELECT incident_id FROM incident_alerts WHERE alert_type = 'water_level' AND external_id = 'sensor-liffey-001'`
2. If a row exists → skip. The incident was already created for this event.
3. If no row exists → call `beacon-core` to create the incident, then INSERT the new mapping into `incident_alerts`.

This means the alert DAG can run every 15 minutes indefinitely, and as long as the same sensor ID keeps triggering, it creates exactly one incident. When the water level drops below the threshold, a separate operation would resolve/close the incident and remove the `incident_alerts` row.

---

## Q22. What is the vault, and how does file storage work?

The **vault** is beacon-core's incident evidence management system. When a ground-staff member takes a photo at the scene, or an ERT member attaches a PDF report to an incident, the file goes into the vault.

**Storage backend:** Google Cloud Storage (GCS), a bucket named `beacon-vault`. Metadata (file name, size, uploader, incident ID, soft-delete status, starred status) is stored in PostgreSQL. The file content is stored in GCS.

**Operations:**
- **Upload** — Client uploads directly to beacon-core, which streams to GCS and records metadata in PostgreSQL.
- **Star** — A file can be "starred" (flagged as important) without moving it.
- **Soft delete** — Files are never permanently deleted immediately. A `deleted_at` timestamp is set. The actual GCS object can be purged later in a scheduled cleanup job.
- **Move** — Files can be reassigned from one incident to another (rare but needed when incidents are merged or split).

---

## Q23. How does `beacon-auditing` know which service is writing each event?

Every `AuditEvent` struct requires a `ServiceName` field. Each microservice provides its own name at startup when it creates the audit client and wires it into domain services. For example, in `beacon-core/cmd/server/main.go`:

```go
incidentService.SetAuditClient(auditClient)
```

When the incident service calls `LogAuditEvent()`, it passes `ServiceName: "beacon-core"` explicitly. The library validates this field is non-empty. If code ever accidentally passes an empty `ServiceName`, `LogAuditEvent()` returns `ErrValidation` — a programming error that tests should catch.

This means the `audit_events` ClickHouse table records which of the 12 services wrote each event. A compliance query can filter `WHERE service_name = 'beacon-core' AND action = 'incident.approved'` to see every incident approval ever made, with who approved it, when, and from what IP.

---

## Q24. What happens if `beacon-iam` goes down?

This is the single most important failure scenario in the platform, because beacon-iam is on the critical path of every authenticated request.

**What fails immediately:**
- All authentication — no new logins possible
- All JWT validation — every API request to every service fails with a 401 or 503
- All dispatch creations — beacon-core cannot check ground-staff availability
- Any new incident push notifications that require fetching FCM tokens

**What partially degrades (fail-open):**
- FCM token resolution for event publishing — beacon-core logs a warning and publishes the event without recipient tokens. Push notifications will not be sent for that event, but the event is still recorded and the state transition still completes.

**What keeps working:**
- Already-accepted dispatches that are in progress (the SSE broker does not depend on beacon-iam)
- ETL pipelines (beacon-etl-flows does not authenticate against beacon-iam)
- beacon-observability data serving (its existing authenticated sessions may still have valid cached tokens briefly, but new requests will fail)

**The practical implication for operations:** beacon-iam must be treated as a high-availability service. It should run multiple replicas, have health checks, and be the first service scaled up during incident response. Its Kubernetes deployment should use PodDisruptionBudgets to prevent all replicas from being taken down simultaneously during a cluster update.

---

## Q25. How do the two Go libraries (`beacon-auditing`, `beacon-event-bus`) use interfaces to support testing?

Both libraries follow the same pattern: they define a Go interface that represents the library's capabilities, provide a real implementation backed by GCP/ClickHouse, and ship a mock implementation in a `mock/` sub-package.

**`beacon-auditing`:**
```go
// The interface any domain service depends on:
type AuditClient interface {
    LogAuditEvent(ctx context.Context, event AuditEvent) error
    QueryAuditLog(ctx context.Context, filter AuditFilter) ([]AuditEvent, error)
}

// In unit tests:
mockAudit := mock.NewClient()
svc := incident.NewService(mockRepo, mockAudit)
svc.ApproveIncident(ctx, "inc-001", "admin-007")
logged := mockAudit.GetLoggedEvents()
assert.Equal(t, "incident.approved", logged[0].Action)
```

**`beacon-event-bus`:**
```go
// In unit tests:
mockBus := mock.New()
svc := dispatch.NewService(mockRepo, mockBus)
svc.AcceptDispatch(ctx, "disp-001", "staff-007")
published := mockBus.GetPublishedEvents("beacon-dispatches")
assert.Equal(t, "dispatch.accepted", published[0].Type)
```

**Why this matters architecturally:**
Domain services depend on interfaces, not concrete implementations. At startup (in `main.go`), the real implementations are injected. In tests, mock implementations are injected. The domain service code is identical in both cases — it calls the interface method. This is dependency injection applied correctly.

---

# Part 3 — Infrastructure, DevOps & Data

---

## Q26. What databases does Beacon use, and why are three different databases needed?

Beacon uses three different database technologies, each chosen for its specific access pattern:

| Database | Technology | Who writes to it | Who reads from it | Why this technology |
|---|---|---|---|---|
| **Operational DB** | PostgreSQL (Cloud SQL) | All microservices (each has its own DB) | Same services (via CQRS slave replica) | ACID transactions, relational integrity, strong consistency for operational state |
| **Cache** | Redis (Cloud Memorystore) | beacon-core (incident cache writes) | beacon-core (incident cache reads) | Sub-millisecond reads for hot data; fail-open means no crash if Redis is down |
| **Analytics/Audit** | ClickHouse | beacon-auditing (audit events), beacon-etl-flows (ETL data) | beacon-observability (ETL data), beacon-core (GPS breadcrumbs) | Column-oriented, high-throughput appends, fast time-range scans, built-in TTL |

**Why not just use PostgreSQL for everything?**

PostgreSQL is excellent for transactional, relational data. But if you store 1 million audit events in PostgreSQL and run `SELECT * WHERE action = 'incident.approved' AND occurred_at > '2026-01-01'`, it becomes slow — PostgreSQL's row-based storage reads entire rows even for queries that only touch two or three columns.

ClickHouse stores data column by column. That same query reads only the `action` and `occurred_at` columns from disk, ignoring all others. For time-series and analytical workloads, this is 10–100x faster. And ClickHouse's built-in `TTL occurred_at + INTERVAL 1 YEAR DELETE` means old audit records are automatically pruned — no maintenance job needed.

---

## Q27. What is Kubernetes and how does Beacon use it?

**Junior intern explanation:** Kubernetes (K8s) is a system for running containerised applications in production. Think of it as an operating system for a cluster of servers. You tell Kubernetes "I want 3 copies of this container running, and if any of them crash, replace them automatically." Kubernetes figures out which server to put them on, restarts crashed containers, and updates to new versions without taking the application down.

**Beacon's setup:**
- Platform: Google Kubernetes Engine (GKE) Autopilot in `europe-west2` (London)
- All Go microservices are deployed as Kubernetes `Deployments`
- Container images are stored in Google Artifact Registry
- Secrets (database passwords, API keys) are stored as Kubernetes Secrets and injected as environment variables
- Traffic enters through a GKE Ingress with a Cloud Load Balancer (TLS termination at the load balancer)

**The distroless image:**
All Go services use a two-stage Docker build: `golang:1.24-alpine` builder → `gcr.io/distroless/static:nonroot` runtime. The distroless image has no shell, no package manager, and no OS utilities. The final image is only ~15–25 MB and has a minimal attack surface — if an attacker gets code execution inside the container, they cannot run `bash`, `curl`, or `wget`. The entire attack surface is the application binary and its linked libraries.

**Deployment command:**
```bash
make gke-build-push  # Build Docker image, push to Artifact Registry
make gke-deploy      # Apply k8s/deployment.yml with envsubst for image tag
make gke-status      # Check rollout progress
```

---

## Q28. What is the CI/CD pipeline and how does a change get from a developer's laptop to production?

Beacon uses GitHub Actions with three workflow files:

| Workflow | Trigger | What it does |
|---|---|---|
| `lint_test.yml` | Push or PR to `main`/`develop` | Runs `golangci-lint` and `go test`. Must pass before merging. |
| `ci.yml` | Push of a version tag (`v*`) | Builds the Docker image, pushes to GCP Artifact Registry |
| `cd.yml` | Manual trigger | Deploys the specified image version to Kubernetes (staging or production) |

**The journey:**

```
Developer writes code on a feature branch
  → Opens a PR → lint_test.yml runs automatically → must pass to merge
  → Code review + approval → merges to main
  → Developer creates version tag: git tag -a v1.2.3 → ci.yml runs
  → Docker image built and pushed to registry
  → Developer goes to GitHub Actions → CD → Run workflow → enters v1.2.3 → deploys
```

**Why CD is manually triggered:**
Automatic deployments to production are considered risky — a passing CI build does not guarantee the feature is ready for production users. The manual trigger gives the team control over "when" something goes live, not just "whether" the tests pass.

---

## Q29. What is semantic versioning and how does Beacon use it?

Semantic Versioning (SemVer) is the convention `vMAJOR.MINOR.PATCH`:

| Change type | Which number increments | Example |
|---|---|---|
| Bug fix (backwards compatible) | PATCH | v1.0.0 → v1.0.1 |
| New feature (backwards compatible) | MINOR | v1.0.1 → v1.1.0 |
| Breaking change (incompatible with previous) | MAJOR | v1.1.0 → v2.0.0 |

In Beacon, the CI workflow (`ci.yml`) triggers on tags matching `v*`. This means every Docker image in the registry is tagged with an exact version. Kubernetes manifests reference this specific version. If a deployment goes wrong, rolling back means deploying the previous version tag.

**Practical example:** If `beacon-core` v1.4.2 introduces a bug, you deploy v1.4.1 in about 2 minutes by running the CD workflow with the old version tag.

---

## Q30. How does Apache Airflow work and why was it chosen for the ETL pipelines?

**Junior intern explanation:** Apache Airflow is a workflow orchestration system for data pipelines. You write DAGs (Directed Acyclic Graphs) in Python — a DAG is just a sequence of tasks with dependencies between them (Task A must complete before Task B starts). Airflow runs these DAGs on a schedule, retries failed tasks, logs every run, and gives you a web UI to monitor everything.

**Why Airflow for Beacon's ETL:**

The ETL pipelines need to:
1. Run on different schedules (every 5 minutes, hourly, daily, monthly)
2. Retry on failure (public APIs are unreliable)
3. Not interfere with each other (a slow weather API response should not delay the bus position fetch)
4. Be auditable (someone needs to see "the water level DAG ran at 14:35 and loaded 47 records")
5. Be easy to add new data sources (write a new DAG file, push to main, it auto-deploys via git-sync)

All of these are built-in Airflow features. The alternative — writing 13 cron jobs with custom retry logic and monitoring — would be much more code for the same result.

**CeleryExecutor:** The Airflow deployment uses CeleryExecutor, which means multiple worker pods run tasks concurrently. If the weather DAG and the bus position DAG both trigger at the same time, they run in parallel on separate workers.

**git-sync:** A sidecar container on each Airflow pod polls the `beacon-etl-flows` GitHub repository every 60 seconds and syncs any new or changed DAG files. This means updating an ETL pipeline is just a `git push` — no Helm upgrade, no pod restart.

---

## Q31. What is ClickHouse and what makes it different from PostgreSQL?

**Junior intern analogy:**
PostgreSQL stores data like a spreadsheet where every row is stored together. ClickHouse stores data like a transposed spreadsheet where every column is stored together.

When you ask "what is the average incident approval time for October?", PostgreSQL reads every row in the incidents table (all columns for every row) and then calculates the answer. ClickHouse reads only the `status`, `created_at`, and `approved_at` columns for October (skipping every other column entirely) and computes the answer much faster because it read far less data from disk.

**Why Beacon uses ClickHouse specifically for:**

1. **Audit events** — The `audit_events` table receives thousands of writes per hour (every state mutation from every service). ClickHouse is designed for high-throughput append workloads. PostgreSQL would struggle under the same write rate and would slow down operational queries.

2. **ETL city data** — Vehicle positions update every 5 minutes for hundreds of buses. Water levels update every 15 minutes. Over a day, these are millions of rows. Time-range queries ("show me all vehicle positions between 14:00 and 15:00") are exactly what ClickHouse is optimised for.

3. **GPS breadcrumbs** — Every active ground-staff dispatch streams GPS coordinates to beacon-core, which writes them to ClickHouse. Post-incident analysis ("where exactly did responder X go during the fire?") is a range query on the `location_updates` table.

**The built-in TTL** is a killer feature: `TTL occurred_at + INTERVAL 1 YEAR DELETE` means ClickHouse automatically deletes rows older than a year. No maintenance script needed, no cron job, no manual VACUUM.

---

## Q32. How is the platform secured, and what threat model does it defend against?

**Authentication and authorisation:**
- All JWTs are signed by Keycloak's private key. Forging a JWT requires stealing that private key.
- Every API request is validated by beacon-iam. There is no path to bypass validation.
- RBAC is enforced with explicit `RequireRole()` middleware — no annotation magic that could be misapplied.
- Google OAuth and Apple Sign In for citizens means Beacon never stores citizen passwords.

**Infrastructure:**
- All containers run as non-root users (`nonroot:nonroot` in the Kubernetes pod spec).
- Distroless images have no shell — even if code execution is achieved inside a container, the attacker cannot run OS tools.
- All secrets are stored in Kubernetes Secrets and injected as environment variables — never hardcoded or in source control.
- TLS is terminated at the Cloud Load Balancer — all traffic between the internet and the platform is encrypted.

**Brute-force protection:**
- beacon-iam tracks failed login attempts per user. After 5 failures, the account locks for 30 minutes.
- This prevents automated credential-stuffing attacks against the ground-staff and ERT login endpoints.

**Audit trail:**
- Every state mutation is recorded in ClickHouse with the actor's ID, role, and IP address.
- Records have a 1-year retention period.
- The audit log is append-only — records cannot be modified.

---

## Q33. What is the local development setup, and how does a new developer get running?

The minimum viable local stack for beacon-core development:
```bash
cd beacon-core
docker-compose up -d    # starts PostgreSQL, Redis, ClickHouse in Docker
make migrate-up          # applies all database migrations
make run                 # starts the Go server on localhost:8080
```

For full cross-service testing:
```bash
# Start beacon-iam first (because everything else needs JWT validation)
cd beacon-iam && docker-compose up -d && make keycloak-import && make migrate-up && make run &

# Then beacon-core
cd beacon-core && make migrate-up && make run &

# Start the Pub/Sub emulator for event bus testing
export EVENTBUS_EMULATOR_HOST=localhost:8085
gcloud beta emulators pubsub start --project=local-test

# Point dashboard at localhost
VITE_API_BASE_URL=http://localhost:8080 npm run dev

# Point mobile app at localhost
flutter run --dart-define=BASE_URL=http://localhost:8080
```

**Prerequisites:**
- Go 1.24+, Flutter 3.9+, Node.js 18+, Python 3.10+, Docker + Compose
- `golang-migrate` for database migrations
- `gcloud` CLI for the Pub/Sub emulator
- A Google service account for Vertex AI access (for beacon-support-bot)

---

# Part 4 — Advanced Software Engineering

---

## Q34. What is Test-Driven Development (TDD) and how does it actually work in practice?

**The ultra-simple explanation (for a day-1 intern):**
TDD is writing the test first, then writing the code to make it pass, then cleaning up the code. The cycle is: **Red → Green → Refactor**.

1. **Red** — Write a test that fails (because the feature does not exist yet)
2. **Green** — Write the minimum code to make the test pass (even if the code is ugly)
3. **Refactor** — Clean up the code without breaking the test

**Example from the Beacon codebase:**

Imagine you need to add "an incident cannot be closed if it has active evacuations." Without TDD, you would write the code first and then think "I should probably test this." With TDD:

```go
// Step 1 — Write the failing test (RED)
func TestCloseIncident_WithActiveEvacuation_ReturnsError(t *testing.T) {
    mockEvacuationChecker := &MockEvacuationChecker{HasActive: true}
    svc := incident.NewService(mockRepo, mockEventBus, mockEvacuationChecker)
    
    err := svc.CloseIncident(ctx, "inc-001", "admin-007")
    
    require.Error(t, err)
    assert.Contains(t, err.Error(), "active evacuation")
}
// This test fails because CloseIncident doesn't check evacuations yet.

// Step 2 — Write minimum code to make it pass (GREEN)
func (s *Service) CloseIncident(ctx context.Context, id, actorID string) error {
    if s.evacuationChecker.HasActiveEvacuations(ctx, id) {
        return errors.New("cannot close incident: active evacuation in progress")
    }
    // ... rest of close logic
}
// Test passes.

// Step 3 — Refactor
// Maybe extract a named error type, improve the error message, add the audit log write.
// Tests still pass throughout.
```

**What TDD actually gives you:**

1. **Executable specification** — The test describes exactly what the code should do. It is documentation that cannot go stale (unlike a comment, which can lie).

2. **Design pressure** — If a piece of code is hard to test, that is a signal it is too tightly coupled. TDD forces you to write code that is loosely coupled by nature — because if you cannot inject a mock, you cannot test in isolation.

3. **Regression protection** — Three months from now, someone refactors `CloseIncident` and accidentally removes the evacuation check. The test fails immediately. Without the test, this breaks production.

4. **Confidence to refactor** — A codebase with good test coverage can be safely changed. Without tests, "if it ain't broke, don't touch it" becomes the only viable strategy, which leads to unmaintainable code over time.

---

## Q35. What are the common criticisms of TDD, and are they valid?

TDD is not universally loved. Here are the honest criticisms:

**Criticism 1: "TDD is slow — I write tests after features, it is faster"**

This is true in the very short term and false in the medium term. Writing the test first takes more upfront time. But debugging a production bug, tracing it through logs, reproducing it locally, and fixing it without breaking something else takes much more time than a test would have caught in seconds. Studies in software engineering (including one at Microsoft on the Windows team) show TDD reduces defect rates by 40–90% at a time cost of 15–35% — an excellent trade.

**Criticism 2: "TDD produces lots of tests that don't test real behaviour"**

This is valid when TDD is applied badly. "Unit test theatre" — testing implementation details rather than behaviour — is a real problem. If you test that `ApproveIncident` calls `repo.Save()` exactly once, you are testing the implementation, not the outcome. The test breaks every time the implementation changes even if the behaviour is correct. Good TDD tests behaviour: "after calling ApproveIncident, the incident status is approved" — not "which methods were called internally."

**Criticism 3: "TDD does not work for exploratory or UI code"**

This is partially valid. TDD is most effective for code with clear inputs and outputs — business logic, API handlers, domain services. It is harder to apply to UI rendering (what does "the button is red" look like in a unit test?) or to genuinely exploratory R&D code where you do not know what you are building yet. Most teams apply TDD strictly to business logic and integration tests to UI.

**Criticism 4: "Tests become a maintenance burden"**

True if tests are brittle (testing implementation, not behaviour). False if tests are written well. A test that fails when you change an internal detail is a bad test. A test that only fails when actual observable behaviour changes is a good test and is never a burden — it is protection.

---

## Q36. What does "AI as a Copilot" mean, and how should a developer actually think about it?

**The copilot metaphor explained:**
In aviation, the copilot handles routine procedures — flap settings, radio communications, instrument checks — so the captain can focus on navigation and decision-making. GitHub Copilot, Cursor, and Claude Code operate similarly: they handle the routine, repetitive parts of writing code so the developer can focus on architecture, design, and problem-solving.

**What AI is genuinely good at:**

1. **Boilerplate generation** — Writing the fifth PostgreSQL repository struct that follows the same pattern as the previous four. AI does this faster than copy-paste-and-modify.

2. **Completing known patterns** — "I need an HTTP handler that validates the JWT, reads the incident ID from the URL, calls the domain service, and returns a 200 or 404" — the AI has seen this pattern thousands of times and produces correct code in seconds.

3. **Looking up syntax and APIs** — "What is the signature for the GCP Pub/Sub Go SDK's Publish method?" The AI answers instantly without a browser tab context switch.

4. **First draft test generation** — Given a function signature, the AI generates test cases (including edge cases the developer might not have thought of). The developer then reviews, edits, and ensures the tests test behaviour rather than implementation.

5. **Documentation** — Explaining what a complex piece of code does, in plain English, for a junior colleague.

**What AI is bad at (and what developers must own):**

1. **Architecture decisions** — Should this be a new service or extend beacon-core? Should this event be synchronous or async? AI does not understand your team's constraints, your load profile, your deployment complexity, or your organisation's risk tolerance. Humans must own this.

2. **Security** — AI will generate SQL queries that are vulnerable to injection if you do not specify parameterised queries. It will generate code that logs passwords if you do not specify otherwise. Security thinking requires deliberate attention that AI does not automatically apply.

3. **Domain correctness** — In Beacon, the approval gate is a critical business rule. AI does not know that "an incident must be approved before a dispatch can be created" is a rule enforced in four places. It might generate dispatch creation code that skips the check. The developer must know the domain rules and verify the AI's output against them.

4. **Code review and judgment** — AI-generated code is a first draft, not a finished product. It needs the same review as a junior developer's PR: does this handle errors correctly? Does this match our conventions? Does this introduce a security vulnerability? Does this violate the CQRS pattern?

---

## Q37. What is "Agentic AI" and how is it different from using AI as a Copilot?

**Copilot mode:** A developer types a function signature, the AI suggests a completion. The developer accepts, modifies, or rejects it. The human is in the loop on every decision. The AI is a fast autocomplete.

**Agentic AI mode:** The developer says "add a GET /incidents/:id/vault endpoint that returns all vault files for an incident." The AI reads the existing codebase, understands the patterns, writes the handler, writes the domain service method, writes the repository query, adds the route to the router, and writes the tests — without the developer approving each step. The human describes the outcome; the agent determines and executes the steps.

**Where agentic AI is powerful:**

- **Well-defined, bounded tasks** — "Add a new endpoint following the existing pattern" is perfect for an agent. The pattern is clear, the outcome is concrete, and the blast radius of mistakes is limited to one feature.
- **Repetitive scaffolding** — "Generate the migration file, model, repository, service, handler, and test for a new domain entity" is exactly the kind of work that takes an experienced developer two hours and an agent fifteen minutes.
- **Refactoring within a clear boundary** — "Rename this field across all uses in this service" is safe for an agent.

**Where agentic AI is dangerous:**

- **Cross-service changes** — An agent that starts modifying multiple services simultaneously (beacon-core + beacon-iam + beacon-notification) can introduce inconsistencies that are hard to catch because no single pull request contains the full picture.
- **Security-critical paths** — The JWT validation logic, the RBAC role checks, the approval gate. These must be human-reviewed. An agent making changes here is a high-risk operation.
- **Database migrations** — A wrong migration in production cannot be easily reversed. An agent that generates and applies a migration without human review of the SQL is dangerous.

**The right mental model:** Agentic AI is not a replacement for engineering judgment — it is an amplifier of engineering productivity. The developer who can clearly specify what they want, review the agent's output critically, and catch mistakes before they reach production will deliver much faster than one who writes every line by hand. But the developer who blindly accepts every agent output will eventually create a serious incident.

---

## Q38. What is "Vibe Coding" and why is it a controversial topic among software engineers?

**What it is:** "Vibe coding" is a term (popularised by Andrej Karpathy) for a development style where the programmer describes what they want at a high level, the AI generates the code, and the programmer mostly accepts the output without deeply understanding each line. The focus is on shipping features quickly, with the programmer as a product manager of the AI rather than a line-by-line author.

**The case for vibe coding:**
- Speed of initial prototyping is genuinely much higher
- Non-programmers can produce working software for personal or low-stakes projects
- For throwaway scripts, one-off data processing, or internal tools, deep code understanding may not be worth the time investment
- Many enterprise features are genuinely "boilerplate + pattern matching" — why not let AI do that?

**The case against vibe coding (especially for production safety-critical systems):**

In a system like Beacon, where software errors can mean a delayed ambulance dispatch or a missed evacuation advisory, vibe coding is dangerous for several concrete reasons:

1. **You cannot debug what you do not understand.** When the production system behaves unexpectedly at 2 AM, the developer needs to understand the code well enough to diagnose the problem, form a hypothesis, and fix it under pressure. Code you accepted without understanding is code you cannot debug.

2. **Security vulnerabilities are invisible to pattern-matching.** SQL injection, IDOR (insecure direct object reference), missing authorisation checks — these are absent not because code is broken, but because a specific constraint was not met. Vibe coding generates code that *works* but may not be *safe*.

3. **Architectural debt accumulates invisibly.** An AI generating code does not know that the CQRS pattern requires commands to go to the master and queries to the slave. It will route queries to the master if that is what it infers from context. That works — until it causes replication lag and then mysterious "stale data" bugs in production.

4. **Test coverage becomes meaningless.** If tests are generated by the same AI that generated the code, they tend to test the implementation, not independent behaviour. They pass because they were written to pass the code that was just generated, not because they verify the behaviour is correct.

**The synthesis:** Vibe coding is appropriate for prototyping, learning, personal projects, and low-stakes automation. It is inappropriate as the sole development practice for production systems where correctness, security, and reliability matter. The best engineers use AI to accelerate work they understand well, not to replace the understanding itself.

---

## Q39. Can TDD be replaced by AI-generated tests?

**The short answer:** AI can generate a first draft of tests quickly, but it cannot replace the thinking process of TDD.

**What AI does well in testing:**
- Given a function, generate test cases including common edge cases
- Generate table-driven tests for functions with many input combinations
- Convert a developer's English description into test code

**What TDD provides that AI-generated tests cannot:**

1. **Design feedback.** The act of writing a test first forces you to think about the interface before the implementation. If writing the test is awkward, that is a signal the interface is awkward. AI writing tests after the fact does not provide this feedback.

2. **Specification first.** In TDD, the test is the specification. The test says "when I call ApproveIncident with an incident that has active dispatches, I expect an error." This forces you to define behaviour before implementation. AI generating tests from existing code validates existing behaviour — it cannot tell you whether that behaviour is correct.

3. **Coverage of your mental model.** An AI testing a function generates tests based on what it infers from the code. If the code has a bug, the AI might generate tests that pass the buggy behaviour — because it infers the intention from the (incorrect) code. A human writing tests against a specification catches bugs. An AI writing tests against code validates the code as-is.

4. **Mutation testing mindset.** Good TDD practitioners ask "if I deliberately break this code, will the test catch it?" (This is the formal practice of mutation testing.) AI-generated tests often have the right assertions but with weak specificity — they assert `err != nil` instead of asserting the specific error type, which means they would not catch a wrong error being returned.

**The practical hybrid approach:**
Many teams use AI to generate a complete first draft of a test suite, then review and strengthen those tests — checking that assertions are specific, that edge cases are covered, that the tests test behaviour not implementation. This is faster than writing tests from scratch and produces better coverage than AI alone.

---

## Q40. What is the difference between unit tests, integration tests, and end-to-end tests, and how should you think about the balance?

**Definitions in the context of Beacon:**

| Test type | What it tests | Dependencies | Speed | Example |
|---|---|---|---|---|
| **Unit test** | A single function or domain service in isolation | All external dependencies are mocked | Milliseconds | Test that `ApproveIncident` publishes the right event to the mock event bus |
| **Integration test** | A service interacting with its real external dependencies | Real PostgreSQL, real Redis (usually in Docker) | Seconds | Test that `CreateDispatch` actually creates a row in the database |
| **End-to-end test** | A full user journey across multiple services | Real running services (usually a test environment) | Minutes | Test that a citizen reporting an incident via the mobile app results in an ERT member receiving a push notification |

**The Testing Pyramid:**

The classic guidance is "write many unit tests, some integration tests, few end-to-end tests." This is because unit tests are fast (100s run in seconds), deterministic (no network flakiness), and precise (they pinpoint exactly what broke). End-to-end tests are slow, flaky, and when they fail they often tell you "something is broken somewhere" without pointing to where.

**Beacon's specific tradeoffs:**

The code-review-graph analysis of `beacon-core` identified **51 untested symbols**, primarily in `internal/adapters/microservices/`. The IAM, Observability, and TomTom HTTP clients have no unit tests. This means:
- If the IAM client's JSON deserialization changes in a way that silently drops the role claim, no unit test catches it
- Only integration or end-to-end tests would catch this — and those tests are slower and less reliable

This is a known gap. The fix is adding unit tests that mock the HTTP transport layer and assert that specific request/response cycles produce the correct parsed structs.

---

## Q41. What does "code cohesion" mean, and what does the graph analysis tell us about beacon-core?

**Cohesion** is a measure of how closely related the responsibilities of a single module (function, class, package) are to each other. High cohesion = "this package does one thing and does it well." Low cohesion = "this package does many different things and reaches into many other packages."

The code-review-graph analysis of `beacon-core` (615 nodes, 2851 edges) found communities with cohesion scores:

- `incident-event`: cohesion **0.93** — Excellent. Domain types are tightly scoped.
- `evacuation-advisory`: cohesion 0.83 — Good.
- `auth-user`: cohesion **0.19** — High risk. The auth package is imported by almost every other package. Any breaking change (JWT claim shape, role constant name) ripples through all 5 domains simultaneously.
- `eventbus-adapter`: cohesion **0.23** — High risk. Adapters map domain structs to Pub/Sub payloads. If a domain struct field is added but the adapter is not updated, the downstream consumers get truncated data — and the **compiler will not catch it**.
- `dispatch-event`: cohesion 0.22 — SSE types are consumed across layers (domain → adapter → handler).

**What this tells us:**
The auth and event bus adapter layers are the most fragile parts of the codebase. Changes to JWT claim shapes or domain event structs are "silent breaking changes" — the code compiles but behaves incorrectly. These areas need the most test coverage and the most careful code review.

---

## Q42. What is "coupling risk" and how does the cross-domain adapter pattern in `main.go` create hidden coupling?

**Coupling** is when changing one piece of code requires changing another piece of code. High coupling = fragile. Low coupling = resilient.

In `beacon-core`, three anonymous structs in `main.go` stitch the domains together:
- `incidentApprovalAdapter` — lets dispatch and evacuation services check if an incident is approved
- `dispatchActivityAdapter` — lets the incident service block closure when active dispatches exist
- `evacuationActivityAdapter` — lets the incident service block closure when active evacuations exist

These implement interfaces defined in the domain packages. Here is the coupling risk:

If you change `incident.SlaveRepository` (add or remove a method), you silently break `dispatchActivityAdapter` and `evacuationActivityAdapter`. The compiler will catch it — **but only if you run `go build`**. A syntax error in a test file can prevent the build from running. And in a CI pipeline that only runs `go test` on specific packages, you might miss the build failure.

**The lesson:** When interfaces cross domain boundaries (incident domain ↔ dispatch domain), changes require auditing every implementation of that interface. The graph's `incident-checker` community having cohesion 0.19 is a signal that this is happening too broadly.

---

## Q43. How does the principle of "failing loudly" apply to `beacon-core`'s startup sequence?

"Fail loudly" means: if your service cannot function correctly, crash immediately with a clear error message rather than starting up and silently behaving incorrectly.

In `beacon-core`'s `main.go`:

```go
// Failing loudly: if the database connection fails, crash immediately
pools, err := postgres.NewConnectionPools(cfg)
if err != nil {
    logger.Fatal().Err(err).Msg("failed to connect to PostgreSQL")
}

// Failing loudly: if audit ClickHouse fails, crash immediately
auditClient, err := auditing.NewClientFromEnv(ctx)
if err != nil {
    logger.Fatal().Err(err).Msg("failed to connect to audit ClickHouse")
}

// Crash recovery: find dispatches that timed out while offline
// This runs BEFORE the HTTP server starts — intentionally
dispatchService.RecoverExpiredDispatches(context.Background())
// Then start the HTTP server
```

The dispatch recovery runs **synchronously before serving traffic**. This is "fail loudly" applied to startup state consistency: the service will not accept new dispatch requests until it has resolved all the timed-out dispatches from before the last crash. If this recovery fails, the service should not start — it would be serving traffic on top of inconsistent state.

**Contrast with "failing silently":** Redis cache invalidation and audit log writes are designed to fail silently (fail-open). This is correct because Redis is an optimisation (not correctness-critical) and audit logging must not block operations. But databases, migrations, and startup state recovery are correctness-critical and must fail loudly.

---

## Q44. What does "Hexagonal Architecture" mean for testability, and how does it apply to the Beacon services?

**The core idea:** Domain code (business logic) should have zero dependencies on infrastructure (databases, HTTP, message queues). The domain defines interfaces ("I need something that can save an incident and retrieve it by ID"). Adapters implement those interfaces. At startup, the real adapters are injected. In tests, mock adapters are injected.

**Applied to `beacon-core`:**

```
domain/incident/service.go:
  // Service depends on INTERFACE, not on PostgreSQL
  type Repository interface {
      Save(ctx, incident Incident) error
      FindByID(ctx, id string) (Incident, error)
  }
  
  type EventPublisher interface {
      Publish(ctx, topic string, event interface{}) error
  }
  
  func NewService(repo Repository, bus EventPublisher, ...) *Service {
      return &Service{repo: repo, bus: bus, ...}
  }

adapters/database/incident_postgres.go:
  // PostgreSQL implementation of the interface
  type IncidentPostgresRepo struct { pool *pgxpool.Pool }
  func (r *IncidentPostgresRepo) Save(ctx, incident) error { /* SQL */ }

cmd/server/main.go:
  // At startup: inject real adapter
  repo := &postgres.IncidentPostgresRepo{pool: masterPool}
  svc := incident.NewService(repo, realEventBus, ...)

test files:
  // In tests: inject mock
  mockRepo := &MockIncidentRepo{}
  svc := incident.NewService(mockRepo, mockEventBus, ...)
```

The domain service code is identical in production and in tests. The only thing that changes is what gets injected. This makes it possible to run thousands of fast unit tests without a database server.

---

## Q45. What is the difference between "integration testing" in theory and how it is actually done in Go/Beacon?

**In theory:** Integration tests verify that multiple real components work together correctly.

**In practice in Go with Docker:**

The Beacon Go services use `docker-compose up -d` in CI to start real PostgreSQL, Redis, and ClickHouse containers. The integration tests then run against these real systems. The `beacon-event-bus` integration tests start the GCP Pub/Sub emulator.

This means "integration test" in Beacon's context usually means: a test that exercises a handler, a service, and a real database, all in the same process, with the database running in a local Docker container.

```go
// Integration test in beacon-core
func TestCreateDispatch_Integration(t *testing.T) {
    // Use the real PostgreSQL connection (from test helper that starts docker-compose)
    repo := postgres.NewIncidentRepo(testPool)
    svc := incident.NewService(repo, mockEventBus, ...)
    
    // Create a real incident in the database
    inc := createTestIncident(t, svc)
    
    // Approve it (updates real DB)
    svc.ApproveIncident(ctx, inc.ID, "admin-001")
    
    // Create a dispatch (reads from real DB, writes to real DB)
    dispatch, err := svc.CreateDispatch(ctx, inc.ID, "staff-001", "station-1")
    
    require.NoError(t, err)
    assert.Equal(t, "pending_acceptance", dispatch.Status)
    
    // Verify the real DB has the dispatch record
    retrieved, _ := dispatchRepo.FindByID(ctx, dispatch.ID)
    assert.Equal(t, dispatch.ID, retrieved.ID)
}
```

This test proves the SQL is correct, the column types are right, and the Go struct serialisation matches the database schema — things unit tests with mocked repositories cannot catch.

---

## Q46. What is the "shift left" principle in testing, and how does it connect to TDD and CI/CD?

**"Shift left"** means moving quality checks earlier in the development lifecycle. "Left" refers to the left side of the timeline: write code → test → build → deploy. Shift left = test and validate closer to "write code," not closer to "deploy."

**Without shift left:**
- Developer writes code
- Developer manually tests in the browser
- Code goes to a shared staging environment
- QA manually tests
- Bug found and reported
- Developer must context-switch back to three-week-old code to fix it

**With shift left (Beacon's CI/CD pipeline):**
- Developer writes a failing test (TDD) — bug caught before code is even written
- Developer runs `make test` locally — caught in 2 seconds on their laptop
- PR opens → `lint_test.yml` runs — caught in GitHub Actions before code is merged
- Docker image is built — static analysis and build-time checks run
- Staging deployment — integration tests run against the deployed service

Each stage catches different classes of bugs. The earlier a bug is caught, the cheaper it is to fix. A bug caught by a unit test before `git push` takes 30 seconds to fix. A bug caught in production might take days of investigation, a hotfix, and a post-mortem.

**TDD is the ultimate shift-left technique** — you catch bugs at the moment you write the code, not even after. The test is the first execution of the code.

---

## Q47. Should TDD always be used? When is it overkill?

**TDD is clearly valuable for:**
- Core business logic with clear rules (the incident approval gate, the dispatch timeout logic)
- Public library APIs (beacon-auditing, beacon-event-bus) — these will be used by many callers; bugs are multiplied
- Security-critical code (JWT validation, RBAC checks, login lockout)
- Code that will be maintained by a team over years

**TDD is overkill for:**
- A one-time data migration script that will be deleted after it runs
- A config file parser that has been stable for three years
- A simple HTML template render
- Exploratory code during early R&D when you are not sure what you are building

**TDD has diminishing returns for:**
- Pure adapters that are thin wrappers around external SDKs (the TomTom geocoding client). The adapter itself has almost no logic — the logic is in the SDK. Testing the adapter mostly tests that you called the SDK correctly, which an integration test does better.
- Generated code (Freezed models in Flutter, `json_serializable` Go structs) — testing that a struct deserialises from JSON is testing the code generator, not your logic.

**The pragmatic answer:** Apply TDD to the 20% of your code that contains 80% of the business rules and is most likely to be changed. Apply integration tests to adapters. Apply end-to-end tests to critical user journeys. Do not apply TDD everywhere as a religious principle — apply it where it provides the highest return on the time invested.

---

## Q48. What are the long-term risks of over-relying on AI for code generation without TDD or tests?

**Scenario: A team builds a new feature in beacon-core using AI-generated code. The AI writes the handler, service method, repository, and tests. No TDD was done. Tests were generated from the implementation.**

**Risk 1: The tests validate the implementation, not the requirement.**
If the AI incorrectly understood "a dispatch can only be created for an approved incident" and generated code that creates dispatches for *any* incident, the AI's generated tests would also test for dispatches on any incident (because they were generated from the same code). All tests pass. The bug ships.

**Risk 2: The next AI is trained on your buggy code.**
Enterprise AI coding assistants increasingly use your organisation's codebase as training data for suggestions. If your codebase has a pattern where dispatches bypass the approval gate, future AI suggestions in your codebase will also bypass it.

**Risk 3: Refactoring becomes impossible.**
Code generated at speed without tests accumulates technical debt invisibly. When the team wants to change the dispatch logic six months later, they have no tests to tell them what existing behaviour must be preserved. Every change is a gamble. The team stops refactoring. The code calcifies.

**Risk 4: Knowledge stays in the AI context, not the team.**
If a developer uses an AI agent to build a feature end-to-end over three hours, and the AI agent's context window expires, and the developer never deeply read the code — nobody on the team understands that feature. The developer who built it cannot explain it. Code review is superficial. Documentation is missing.

**The protection:** TDD (or at minimum, comprehensive tests written independently of the AI's code generation) provides a specification that outlives any AI session, documents intended behaviour, and catches regressions regardless of who (or what) made the change.

---

## Q49. How does "AI as a Copilot" change the skill set a developer needs to have?

This is a genuinely important question for developers early in their careers.

**Skills that matter MORE with AI copilots:**

1. **Architectural thinking** — AI generates code at the function level. Deciding where a new feature belongs in the system, how it should integrate with existing services, what the data model should look like — that requires architectural judgment that AI cannot substitute.

2. **Critical code reading** — AI output must be reviewed. Developers need to be able to read unfamiliar code quickly and spot correctness issues, security vulnerabilities, and style violations. This is a skill developed through deliberate practice.

3. **Problem specification** — AI is only as useful as the prompt it receives. "Add an endpoint" produces mediocre results. "Add a GET /dispatches/:id endpoint that returns 404 if not found, 403 if the requesting user is not the assigned ground-staff or an ERT member, and returns the dispatch entity including the incident summary" produces a much more complete result. Knowing how to specify a problem precisely is now a core engineering skill.

4. **Testing and verification** — Developers need to be more rigorous about testing AI output because AI code is less predictable than code from a colleague you know well. TDD becomes more important, not less.

5. **Security awareness** — AI generates what looks plausible, including plausibly-insecure code. Developers need to review AI output specifically for SQL injection, missing authorisation checks, sensitive data in logs, and other OWASP Top 10 vulnerabilities.

**Skills that matter LESS with AI copilots:**
- Memorising library APIs and syntax
- Typing speed / raw code writing throughput
- Knowing every flag of every CLI tool by heart

**The net effect for a junior developer:** AI copilots raise the productivity ceiling for junior developers significantly. A junior who knows how to write good specs, review code critically, and test thoroughly will produce more correct, complete features than a junior who writes every line by hand. But a junior who uses AI as a substitute for understanding will have a fragile foundation that collapses when things go wrong.

---

## Q50. Looking at the Beacon platform as a whole — what are the five most important engineering lessons it demonstrates?

**Lesson 1: Design for failure, not for success.**

Every design decision in Beacon assumes something will fail. ClickHouse goes down → audit logging degrades silently and primary operations continue. Redis goes down → cache reads miss and PostgreSQL absorbs the load. beacon-iam degrades → FCM token resolution fails open and events are published without recipients. The system is designed around which failures are acceptable (degraded features) and which are not (incident creation, dispatch creation require beacon-iam to be up).

**Lesson 2: Explicit beats implicit.**

RBAC in Beacon is explicit: `ertRouter.Use(auth.RequireRole("ERTMember", "Admin"))` in the router. There is no annotation magic, no implicit role inheritance, no convention-over-configuration. The role check is right there in the code, readable by anyone, auditable in a PR. Implicit security mechanisms (annotations, decorators that can be misapplied) are a common source of vulnerabilities.

**Lesson 3: Shared libraries have outsized leverage — and outsized risk.**

`beacon-auditing` and `beacon-event-bus` are imported by all 6 microservices. A bug in the audit client's batch flush logic affects every service simultaneously. A breaking change to the CloudEvents envelope format (a field rename) breaks every publisher and consumer at once. Shared libraries must be treated with more care than any individual service — reviewed more thoroughly, versioned carefully, and tested more comprehensively.

**Lesson 4: Separation of concerns makes change safe.**

The ETL pipelines (beacon-etl-flows) write to ClickHouse. The observability service (beacon-observability) reads from ClickHouse. They never call each other. This means adding a new ETL data source (write a new DAG) never requires changes to beacon-observability. Adding a new observability endpoint (add a new domain) never requires changes to any ETL pipeline. The separation is the feature.

**Lesson 5: Tests are the most durable documentation.**

Architecture diagrams go stale. READMEs go stale. Comments go stale. But a test that says `TestCloseIncident_WithActiveDispatch_ReturnsError` will always be up to date, because if the behaviour changes, the test fails. The 51 untested functions identified in beacon-core's code analysis are not just a test coverage metric — they are 51 pieces of undocumented behaviour that the next developer to touch that code will have to reverse-engineer from the implementation. Tests are the only documentation that cannot lie.

---

## Appendix: Quick Reference Map

```
Service                 Port    Language   Primary Role
────────────────────────────────────────────────────────────────────────
beacon-core             8080    Go         Incidents, dispatches, evacuations, vault
beacon-iam              8081    Go         Authentication, JWT, FCM tokens, staff roster
beacon-observability    8082    Go         Live city data (traffic, weather, hospitals, etc.)
beacon-notification     8083    Go         Push (FCM), SMS (Twilio), email (SendGrid)
beacon-communication-  8084    Go         WebSocket chat, social posts, Twilio voice
  platform
beacon-support-bot      8085    Go         Vertex AI conversation, risk analysis, escalation
beacon-auditing         —       Go lib     Async ClickHouse audit log client
beacon-event-bus        —       Go lib     GCP Pub/Sub wrapper (CloudEvents + circuit breaker)
beacon-scheduler        —       Go lib     GCP Cloud Scheduler adapter
beacon-etl-flows        —       Python     13 ETL DAGs + 5 alert DAGs on Apache Airflow
beacon-dashboard        —       React/TS   ERT web dashboard (Vite + Tailwind + TomTom)
beacon-mobile-app       —       Flutter    Citizen + ground-staff mobile app (iOS + Android)
```

```
Data Stores
────────────────────────────────────────────────────────────────────────
PostgreSQL (Cloud SQL)   Per-service relational data, CQRS master/slave
Redis (Cloud Memorystore) Incident cache (1h TTL), session data
ClickHouse               Audit events (1y TTL), ETL city data, GPS breadcrumbs
Google Cloud Storage     Vault files (beacon-vault bucket)
```

```
External APIs
────────────────────────────────────────────────────────────────────────
TomTom                   Maps, geocoding, routing (mobile app + ETL)
Firebase (FCM)           Push notification delivery
Twilio                   SMS + voice channels
SendGrid                 Transactional email
Vertex AI (Gemini)       AI conversation + risk analysis (support-bot)
Google / Apple OAuth     Citizen authentication
Keycloak                 JWT issuance and RBAC management (beacon-iam)
NTA GTFS-R               Bus positions (ETL)
Met Éireann              Weather data (ETL)
ESB Networks             Power outage data (ETL)
waterlevel.ie            River/coastal gauges (ETL)
Irish Rail               Train movements (ETL)
Luas                     Tram forecasts (ETL)
Dublin Live              News RSS (ETL)
```

---

*Document generated: 2026-04-14. Based on beacon-docs architecture-sync, development guides, and code graph analysis. Covers all 12 repositories in the Beacon platform.*
