# beacon-notification

Multi-channel notification microservice for the Beacon emergency-response platform. beacon-notification is the central broadcast and alerting hub — it receives lifecycle events from beacon-core via the event bus and translates them into push notifications, SMS messages, and emails, delivered through Firebase Cloud Messaging, Twilio, and SendGrid respectively.

---

## Overview

When a major incident is approved, when a ground-staff member is dispatched to a scene, or when an evacuation advisory is broadcast to a district, citizens and responders need to be informed immediately. beacon-notification is the service responsible for that final delivery step. It consumes events published by beacon-core to the GCP Pub/Sub event bus and fans them out to the appropriate users across all available channels.

The service is designed around three principles:

1. **Channel-agnostic business logic.** The notification domain does not know or care whether a message will be sent via FCM, SMS, or email. Delivery adapters are plugged in at startup and the domain logic remains unchanged regardless of which external providers are active.

2. **Template-driven content.** Notification content is managed as templates stored in the database, not hardcoded in the service. Admins can update the wording of a push notification without redeploying the service.

3. **Acknowledgement tracking.** Critical notifications (dispatch assignments, mandatory evacuation orders) require explicit acknowledgement from the recipient. The service tracks whether each notification has been seen and acknowledged, enabling escalation if a responder does not acknowledge a dispatch within the timeout window.

---

## Architecture

```
GCP Pub/Sub (event bus)
      │
      │  incident.approved, dispatch.accepted, evacuation.advisory.broadcast, ...
      ▼
┌─────────────────────┐
│  Event Subscriber   │  Consumes events from beacon-event-bus
│  (beacon-event-bus) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  Notification Domain                                │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │ Notification    │  │ Template engine           │  │
│  │ service         │  │ (resolves content by type │  │
│  │                 │  │  and locale)              │  │
│  └────────┬────────┘  └──────────────────────────┘  │
│           │                                          │
│  ┌────────▼──────────────────────────────────────┐  │
│  │  Channel router                               │  │
│  │  (FCM for push, Twilio for SMS,               │  │
│  │   SendGrid for email — based on user prefs    │  │
│  │   and notification criticality)               │  │
│  └────────┬──────────────────────────────────────┘  │
└───────────┼─────────────────────────────────────────┘
            │
            ├── Firebase Cloud Messaging (push)
            ├── Twilio                  (SMS)
            └── SendGrid                (email)
```

---

## Domains

### Notification

The core entity. A Notification record is created for each delivery attempt and tracks the recipient, channel, content, delivery status (pending / sent / failed / acknowledged), and timestamps. Multiple Notification records may be created for a single triggering event — one per channel per recipient.

### Template

Templates define the content for each notification type, keyed by `(type, locale)`. A template contains a subject line, short body (for push/SMS), and long body (for email). Variables in the template body (`{{incident_id}}`, `{{severity}}`, `{{location}}`) are substituted at render time using the event payload.

Templates are managed through the admin API. When a new notification type is added (e.g., for a new event type emitted by beacon-core), a corresponding template must be created in the database before the notification can be rendered.

### Acknowledgement

Some notifications require the recipient to explicitly confirm they have received and understood the message. Acknowledgements are stored with the recipient's user ID, timestamp, and optional comment. beacon-core can query acknowledgement status to determine whether a ground-staff member has accepted or ignored a dispatch notification.

### Batch Job

For broadcast scenarios — notifying all citizens in an evacuation zone, or all on-duty ground staff about a region-wide incident — the batch domain manages large-scale fan-out. A batch job specifies the recipient criteria, channel, template, and delivery schedule. The beacon-scheduler library is used to stagger delivery and avoid overwhelming FCM or Twilio rate limits.

---

## API Reference

All endpoints are prefixed with `/api/notification/v1`.

### Notifications

#### `POST /notifications/send`
Sends an immediate notification to a specific user. Used for operational alerts where the event-bus-driven flow is not appropriate — for example, sending a manual message from an admin to a specific ERT member.

**Request body:**
```json
{
  "recipient_id": "user-uuid",
  "type": "manual.alert",
  "channels": ["push", "sms"],
  "data": { "message": "Please call dispatch immediately." }
}
```

The `channels` field specifies the delivery channels for this notification. If omitted, the service uses the recipient's notification preferences (stored in beacon-iam) to determine which channels to use.

#### `POST /notifications/broadcast`
Sends a notification to all users matching a given criteria set — for example, all citizens currently located within a geographic zone, or all on-duty ground staff in a region. This creates a batch job internally and returns a job ID for status tracking.

**Request body:**
```json
{
  "criteria": {
    "role": "Citizen",
    "zone_id": "evac-zone-7b"
  },
  "type": "evacuation.mandatory",
  "channels": ["push", "sms", "email"],
  "data": { "zone": "Dublin 2", "assembly_point": "Merrion Square" }
}
```

#### `POST /notifications/:id/mark-read`
Marks a notification as read by the authenticated user. Called by the mobile app when the notification is displayed. Does not constitute an acknowledgement — only updates the read timestamp.

#### `GET /notifications/history`
Returns the notification history for the authenticated user, paginated and ordered by sent time descending. Used by the mobile app's notification list screen.

**Query parameters:** `page` (default: 1), `page_size` (default: 20), `unread_only` (boolean), `type` (filter by notification type)

### Acknowledgements

#### `POST /notifications/:id/acknowledge`
Records an explicit acknowledgement for a notification. Used for critical alerts — dispatch assignments, mandatory evacuation orders — where passive delivery is insufficient. The service records the acknowledgement timestamp and notifies beacon-core that the recipient has confirmed receipt.

**Request body:** `{ "comment": "En route to scene" }` *(comment is optional)*

#### `GET /notifications/:id/acknowledgements` *(ERT and Admin only)*
Returns all acknowledgements for a given notification — who acknowledged it, when, and with what comment. Used by ERT commanders to track which ground staff have confirmed dispatch acceptance.

### Templates

#### `POST /templates` *(Admin only)*
Creates a new notification template. Templates are keyed by type and locale — creating `("dispatch.assigned", "en")` defines the English-language content for dispatch assignment notifications.

**Request body:**
```json
{
  "type": "dispatch.assigned",
  "locale": "en",
  "subject": "New Dispatch Assignment",
  "short_body": "You have been dispatched to incident {{incident_id}} at {{location}}.",
  "long_body": "A dispatch has been created for you...",
  "variables": ["incident_id", "location", "severity"]
}
```

#### `GET /templates` *(Admin only)*
Lists all templates. Supports filtering by type and locale.

#### `PUT /templates/:id` *(Admin only)*
Updates an existing template. Changes take effect immediately for all subsequent notifications of that type — no redeployment required.

#### `DELETE /templates/:id` *(Admin only)*
Deletes a template. Notifications of the associated type will fail to render until a replacement template is created.

### Batch Jobs

#### `POST /batch` *(Admin and ERT only)*
Creates and schedules a batch notification job. Returns a job ID immediately; delivery happens asynchronously.

#### `GET /batch/:id`
Returns the current status of a batch job: how many recipients were identified, how many notifications have been sent, how many have been acknowledged, and how many failed.

#### `DELETE /batch/:id`
Cancels a pending or in-progress batch job. Notifications already sent are not retracted. The associated beacon-scheduler job is also cancelled.

---

## Event-Driven Delivery

The majority of notifications are triggered automatically by events published to GCP Pub/Sub by beacon-core. The service subscribes to the following event types:

| Event Type | Notification Triggered |
|---|---|
| `incident.created` | Push notification to nearby ERT members and admins |
| `incident.approved` | Push notification to all active citizens in the affected area |
| `incident.closed` | Summary push notification to involved responders |
| `dispatch.assigned` | Push + SMS to the assigned ground-staff member (requires acknowledgement) |
| `dispatch.accepted` | Push notification to the incident commander |
| `dispatch.completed` | Push notification to the incident commander |
| `evacuation.advisory.broadcast` | Push + SMS + email to all citizens in the affected evacuation zone |

For each event, the service:
1. Retrieves the FCM tokens for all targeted recipients from beacon-iam
2. Renders the notification content using the appropriate template
3. Delivers via each configured channel concurrently
4. Records a Notification record for each delivery attempt
5. For critical notifications, creates an Acknowledgement requirement

---

## Delivery Channels

### Firebase Cloud Messaging (Push)

Used for all real-time push notifications to the mobile app. beacon-notification calls the Firebase Admin SDK with the recipient's FCM token (retrieved from beacon-iam). If a token has been invalidated by Firebase (device unregistered), the delivery adapter calls beacon-iam to remove the stale token.

### Twilio (SMS)

Used for critical notifications where push is not sufficient — mandatory evacuation orders, SOS acknowledgements. SMS delivery is best-effort: Twilio attempts delivery once and reports status asynchronously via webhook. The service stores the Twilio message SID for status tracking.

### SendGrid (Email)

Used for non-urgent, high-detail notifications — incident summaries, ERT report digests, membership approval decisions. SendGrid delivery uses transactional email templates. The service passes template variables to SendGrid's dynamic template system rather than rendering HTML locally.

---

## Configuration

```yaml
server:
  port: 8083

database:
  master:
    host: ${BEACON_DATABASE_MASTER_HOST}
  slave:
    host: ${BEACON_DATABASE_SLAVE_HOST}

eventbus:
  project_id: ${GCP_PROJECT_ID}
  subscription_id: beacon-notification-events

firebase:
  credentials_file: /var/secrets/firebase.json

twilio:
  account_sid: ${TWILIO_ACCOUNT_SID}
  auth_token:  ${TWILIO_AUTH_TOKEN}
  from_number: +353XXXXXXXXX

sendgrid:
  api_key: ${SENDGRID_API_KEY}
  from_email: noreply@beacon-tcd.tech
  from_name: Beacon Emergency Response

iam:
  base_url: http://beacon-iam:8081      # For FCM token lookups
```

---

## Local Development

```bash
# Install dependencies
go mod download

# Run database migrations
make migrate-up

# Start the service
make run

# Run tests
make test
```

For local event-driven testing, set `EVENTBUS_EMULATOR_HOST=localhost:8085` and use the Pub/Sub emulator to publish test events without real GCP credentials.

---

## Project Structure

```
beacon-notification/
├── cmd/server/main.go           # Composition root
├── internal/
│   ├── domain/
│   │   ├── notification/        # Notification lifecycle and delivery state
│   │   ├── template/            # Template management and rendering
│   │   ├── acknowledgement/     # Acknowledgement tracking
│   │   └── batch/               # Bulk broadcast job management
│   ├── http/
│   │   ├── router.go            # Routes + RBAC middleware
│   │   └── handlers/
│   ├── registry/
│   │   ├── commands/            # send_notification, broadcast, mark_read, acknowledge, etc.
│   │   └── queries/             # get_history, get_acknowledgements, get_batch_status
│   └── adapters/
│       ├── database/            # Postgres repositories
│       ├── microservices/
│       │   └── iam_client.go    # FCM token retrieval from beacon-iam
│       ├── fcm_client.go        # Firebase Cloud Messaging
│       ├── twilio_client.go     # Twilio SMS
│       └── sendgrid_client.go   # SendGrid email
├── migrations/
└── pkg/
```
