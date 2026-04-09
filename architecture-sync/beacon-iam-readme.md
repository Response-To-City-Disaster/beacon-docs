# beacon-iam

Identity and Access Management microservice for the Beacon emergency-response platform. beacon-iam is the authentication and authorization gateway for all platform traffic — every request to every other service is validated through this service. It acts as a facade over Keycloak, extending it with Beacon-specific concepts: ground-staff duty management, emergency responder (ERT) approval workflows, device and FCM token tracking, and brute-force login protection.

---

## Overview

beacon-iam serves a dual purpose. For end users it is the service they interact with to sign in, manage their profile, and register their devices for push notifications. For other Beacon microservices it is a shared infrastructure dependency — `beacon-core` calls beacon-iam to validate every JWT on every request, check ground-staff availability when creating a dispatch, and retrieve FCM tokens to send push notifications.

The service is opinionated about identity: it does not implement its own OAuth server. All token issuance, validation, and refresh is delegated to Keycloak. beacon-iam adds the platform-level concepts — user profiles with emergency-specific fields, device registrations, ERT membership approval — that Keycloak does not know about.

### User Roles

| Role | Description |
|---|---|
| `Citizen` | Members of the public. Can report incidents, view safety information, use the AI support bot, and receive push notifications. Sign in via Google OAuth. |
| `GroundStaff` | First-responder field staff. Can accept/reject dispatches, update incident status from the field, and participate in team communication channels. Authenticate via username and password or LDAP. |
| `ERTMember` | Emergency Response Team members with elevated privileges. Can create dispatches, create evacuation zones, and view operational analytics. |
| `Admin` | Platform administrators. Full access to all operations including user management and system configuration. |

---

## Architecture

beacon-iam follows the same Hexagonal Architecture + CQRS pattern used across the Beacon platform:

```
HTTP Layer (Gorilla Mux + JWT middleware)
      │
      ▼
Registry Layer (Commands + Queries)
      │
      ├── Commands → Master PostgreSQL (writes: create user, update profile, register device)
      └── Queries  → Slave PostgreSQL  (reads: get user, list staff, check availability)
      │
Domain Layer
      ├── User        — Keycloak-extended profile, contact info, preferences
      ├── ERTMember   — Approval workflow for ERT role promotion
      ├── GroundStaff — Duty status (on-duty / off-duty), availability tracking
      ├── Session     — Active JWT references, multi-device session management
      ├── Device      — FCM tokens, device metadata, location last-seen
      └── LoginAttempt — Failed attempt tracking, account lockout enforcement
      │
Adapters Layer
      ├── database/     — Postgres master/slave repositories
      ├── keycloak/     — Keycloak Admin REST API client
      └── fcm/          — Firebase Admin SDK for token validation
```

---

## Key Responsibilities

### Authentication

- **Google OAuth (Citizens)**: Handles the full OAuth2 PKCE flow for citizen sign-in with Google. Exchanges the authorization code for tokens, creates or updates the citizen profile in beacon-iam's database, and issues a Keycloak token for subsequent API calls.
- **Apple OAuth (Citizens)**: Sign in with Apple flow for iOS users of the mobile app.
- **Password / LDAP (ERT + Ground Staff)**: Username and password authentication for operational staff. Passwords are validated through Keycloak, which can back against an LDAP directory for enterprise deployments.
- **Token Refresh**: Accepts Keycloak refresh tokens and issues new access tokens. Called automatically by the mobile app and dashboard to maintain sessions.

### JWT Validation (Service-to-Service)

Every other Beacon service calls beacon-iam to validate incoming JWTs. The validation endpoint checks the token signature against Keycloak's public keys, confirms the token has not expired, and returns the decoded claims including the user ID, role, and email. This is the highest-traffic operation in beacon-iam and is called on every authenticated HTTP request across the platform.

### Ground Staff Management

beacon-core calls beacon-iam at dispatch creation time to find available ground-staff members near an incident. The availability query returns staff members who are currently on-duty and not already assigned to an active dispatch. beacon-iam owns the duty status (`on_duty` / `off_duty`) which ground staff toggle from the mobile app at the start and end of their shift.

### FCM Token Management

The mobile app registers its Firebase Cloud Messaging token with beacon-iam on startup and whenever the FCM library rotates the token. beacon-core and beacon-notification call beacon-iam to retrieve active FCM tokens for a user before sending push notifications. The Device domain tracks which tokens are current and which have been invalidated by FCM.

### ERT Member Approval Workflow

When a user requests ERT membership, a pending approval record is created. An existing admin reviews and either approves or rejects the request. On approval, beacon-iam calls the Keycloak Admin API to assign the `ERTMember` role to the user's Keycloak account and updates the local record accordingly.

### Login Lockout

Failed authentication attempts are tracked per-user. After a configurable threshold of consecutive failures, the account is temporarily locked to prevent brute-force attacks. The lockout record stores the failure count, first-failure timestamp, and lockout-until time. Successful authentication resets the counter.

---

## API Reference

All endpoints are prefixed with `/api/iam/v1`.

### Authentication

#### `POST /auth/login`
Authenticates a ground-staff or ERT member with username and password. Passes credentials through to Keycloak and returns the full token pair (access token + refresh token). Increments the failed-attempt counter on failure. Locks the account after threshold failures.

**Request body:** `{ "username": "...", "password": "..." }`
**Response:** `{ "access_token": "...", "refresh_token": "...", "expires_in": 3600, "user": { ... } }`

#### `POST /auth/refresh`
Exchanges a valid Keycloak refresh token for a new access token without requiring the user to re-enter their password. Called automatically by the mobile app and dashboard before each request when the access token is within 60 seconds of expiry.

**Request body:** `{ "refresh_token": "..." }`
**Response:** `{ "access_token": "...", "expires_in": 3600 }`

#### `POST /auth/oauth/google`
Completes the Google OAuth2 flow for citizen sign-in. Accepts the authorization code returned by Google after the consent screen, exchanges it for an ID token and access token, creates or retrieves the Beacon citizen profile, and issues a Keycloak token. This is the final callback step of the PKCE flow initiated by the mobile app.

**Request body:** `{ "code": "...", "redirect_uri": "..." }`
**Response:** Same token structure as `/auth/login`

#### `POST /auth/oauth/apple`
Completes the Sign in with Apple flow. Equivalent to the Google endpoint but for Apple's identity service. Used by iOS users of the mobile app.

#### `POST /auth/logout`
Invalidates the provided refresh token in Keycloak. The access token will expire naturally (JWTs are stateless). The device's FCM token association is maintained — the user continues to receive push notifications until they explicitly unregister their device.

#### `POST /auth/validate`
Validates a JWT and returns the decoded claims. Called by every other Beacon service on every authenticated request. This endpoint is on the critical path for the entire platform — any latency here adds latency to every API call across all services.

**Request body:** `{ "token": "..." }`
**Response:** `{ "valid": true, "user_id": "...", "role": "Admin", "email": "..." }`

#### `POST /auth/register`
Self-registration for ground staff. Creates a new user account in both Keycloak and the beacon-iam database, assigns the `GroundStaff` role, and sends an email verification link. The account is active immediately but the `GroundStaff` role is only fully effective once email is verified.

### User Profile

#### `GET /users/me`
Returns the full profile of the authenticated user, including Keycloak attributes (email, name, verified status) merged with beacon-iam-specific fields (role, duty status, device count, FCM token count).

#### `PUT /users/me`
Updates the authenticated user's profile. Accepts fields such as display name, phone number, emergency contact, and notification preferences. Field validation is enforced — phone numbers must match E.164 format if provided.

#### `GET /users/:id` *(Admin only)*
Returns the full profile for any user by ID. Used by the admin dashboard for user management.

#### `GET /users` *(Admin only)*
Lists all users with pagination and optional filters (role, status, search query). Used by the admin dashboard to display the user management table.

### Ground Staff

#### `GET /ground-staff`
Returns a list of all ground staff members. beacon-core calls this endpoint during dispatch creation to find available staff near an incident. The response includes each staff member's duty status, current location (if shared), and active dispatch count.

#### `GET /ground-staff/available`
Returns ground staff members who are currently on-duty and not assigned to an active dispatch. This is a filtered version of `GET /ground-staff` used specifically by beacon-core's dispatch creation logic.

#### `PUT /ground-staff/duty`
Updates the duty status of the authenticated ground-staff member. Ground staff call this from the mobile app to go on-duty at the start of a shift and off-duty at the end. Only the staff member themselves can update their own duty status; admins cannot override it remotely.

**Request body:** `{ "on_duty": true }`

### ERT Members

#### `POST /ert-members/apply`
Submits an application for ERT membership. Creates a pending approval record. The applicant must already have a verified citizen account. Multiple pending applications are not allowed — the endpoint returns an error if an open application already exists for the user.

#### `GET /ert-members/applications` *(Admin only)*
Lists all pending ERT membership applications. Admins review these in the dashboard to approve or reject new ERT members.

#### `POST /ert-members/applications/:id/approve` *(Admin only)*
Approves an ERT membership application. Calls the Keycloak Admin API to promote the user's role to `ERTMember`, updates the local application record to `approved`, and triggers a notification to the applicant.

#### `POST /ert-members/applications/:id/reject` *(Admin only)*
Rejects an ERT membership application with an optional reason. Updates the application record and notifies the applicant.

### Devices and FCM Tokens

#### `POST /devices`
Registers or updates a device and its FCM token for the authenticated user. Called by the mobile app on startup and whenever Firebase rotates the FCM token. Multiple devices per user are supported (the user may be signed in on a tablet and phone simultaneously).

**Request body:** `{ "device_id": "...", "fcm_token": "...", "platform": "android|ios", "model": "..." }`

#### `DELETE /devices/:device_id`
Removes a device registration. Called on explicit sign-out from a device. After removal, the device no longer receives push notifications for the user.

#### `GET /users/:id/fcm-tokens` *(Internal — called by beacon-core and beacon-notification)*
Returns all active FCM tokens for a user. beacon-core calls this when publishing an `incident.approved` or `evacuation.advisory` event to include the push recipients in the event payload. If the user has no registered devices, an empty list is returned — this is not an error.

### Sessions

#### `GET /sessions/me`
Returns all active sessions for the authenticated user — one per device or browser where the user is currently signed in. Each session includes the device type, last-seen timestamp, and partial IP address.

#### `DELETE /sessions/:session_id`
Revokes a specific session (equivalent to signing out from a specific device remotely). Invalides the associated Keycloak refresh token.

---

## Local Development

### Prerequisites

| Dependency | Version | Purpose |
|---|---|---|
| Go | 1.24+ | Service runtime |
| Docker + Docker Compose | Latest | Keycloak + PostgreSQL |
| `golang-migrate` | Latest | Database migrations |

### Setup

```bash
# 1. Clone and install dependencies
git clone https://github.com/Response-To-City-Disaster/beacon-iam.git
cd beacon-iam
go mod download

# 2. Start Keycloak and PostgreSQL
docker-compose up -d

# 3. Import the Keycloak realm (creates the Beacon realm with roles and clients)
# Wait ~30 seconds for Keycloak to start, then:
make keycloak-import

# 4. Run database migrations
make migrate-up

# 5. Copy and edit config
cp config.example.yaml config.yaml
# Set keycloak.admin_password and database credentials

# 6. Run the service
make run
```

The service starts on port `8081` by default. Keycloak Admin Console is available at `http://localhost:8080`.

### Keycloak Realm Configuration

The `realm-export.json` file in the repository root contains a complete Keycloak realm export that defines:
- The `beacon` realm
- Client configuration for the mobile app and dashboard
- The four platform roles: `Citizen`, `GroundStaff`, `ERTMember`, `Admin`
- Social identity providers (Google and Apple)

Import it during initial setup with `make keycloak-import`. If you modify the Keycloak configuration, export the updated realm and commit the new `realm-export.json`.

---

## Database Migrations

Migrations are in `migrations/` and managed by `golang-migrate`. All migrations are numbered sequentially and are forward-only (no down migrations in production).

| Migration | Description |
|---|---|
| `0001` | Initial schema: users table |
| `0002` | Ground staff: duty status and location |
| `0003` | ERT members: application workflow |
| `0004` | Sessions: JWT reference tracking |
| `0005` | Devices: FCM tokens and device metadata |
| `0006` | Login attempts: brute-force lockout tracking |
| `0007` | Indexes for availability queries |
| `0008` | Profile extensions: emergency contact, notification preferences |

```bash
make migrate-up            # Apply all pending migrations (local)
make migrate-down          # Roll back the last migration (local, with caution)
make migrate-up-cloud      # Apply migrations against the production Cloud SQL instance
```

---

## Configuration Reference

Key configuration values (set via `config.yaml` or environment variables):

```yaml
server:
  port: 8081

database:
  master:
    host: ${BEACON_DATABASE_MASTER_HOST}
    port: 5432
    dbname: iam
  slave:
    host: ${BEACON_DATABASE_SLAVE_HOST}

keycloak:
  base_url: http://keycloak:8080
  realm: beacon
  admin_username: admin
  admin_password: ${KEYCLOAK_ADMIN_PASSWORD}
  client_id: beacon-core
  client_secret: ${KEYCLOAK_CLIENT_SECRET}

oauth:
  google:
    client_id: ${GOOGLE_OAUTH_CLIENT_ID}
    client_secret: ${GOOGLE_OAUTH_CLIENT_SECRET}
  apple:
    team_id: ${APPLE_TEAM_ID}
    key_id: ${APPLE_KEY_ID}
    private_key_path: /var/secrets/apple.p8

login:
  max_attempts: 5          # failed logins before lockout
  lockout_minutes: 30      # lockout duration

firebase:
  credentials_file: /var/secrets/firebase.json
```

---

## Deployment (GKE)

```bash
# Build and push the Docker image
make gke-build-push

# Deploy to Kubernetes
make gke-deploy

# Check pod status
make gke-pods

# Tail live logs
make gke-logs
```

The Kubernetes deployment reads all secrets from the `beacon-iam-secrets` Secret in the cluster, which must be populated before deployment.

---

## Project Structure

```
beacon-iam/
├── cmd/server/main.go           # Composition root — wires all dependencies
├── config.yaml                  # Service configuration
├── realm-export.json            # Keycloak realm definition
├── docker-compose.yml           # Local Keycloak + PostgreSQL setup
├── internal/
│   ├── domain/
│   │   ├── user/                # User profile lifecycle
│   │   ├── ertmember/           # ERT approval workflow
│   │   ├── groundstaff/         # Duty status and availability
│   │   ├── session/             # JWT session references
│   │   ├── device/              # FCM token and device management
│   │   └── loginattempt/        # Brute-force lockout
│   ├── http/
│   │   ├── router.go            # All routes + RBAC middleware
│   │   └── handlers/            # auth_handler.go, profile_handler.go, etc.
│   ├── registry/
│   │   ├── commands/            # Mutating operations (write path → master)
│   │   └── queries/             # Read operations (slave path)
│   └── adapters/
│       ├── database/            # Postgres master/slave repositories
│       ├── keycloak/            # Keycloak Admin REST API client
│       └── fcm/                 # Firebase Admin SDK adapter
├── migrations/                  # golang-migrate SQL files
└── pkg/                         # Shared utilities (auth, config, logger)
```
