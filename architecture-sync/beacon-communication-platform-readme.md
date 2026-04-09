# beacon-communication-platform

Real-time communication microservice for the Beacon emergency-response platform. beacon-communication-platform provides the messaging infrastructure for the platform — WebSocket-based team chat for ground staff and ERT coordinators, structured community and social posts for citizens, and Twilio-powered voice channels for critical voice coordination during emergencies.

---

## Overview

Emergency response coordination demands fast, reliable communication between field responders, dispatch operators, and incident commanders. beacon-communication-platform provides that infrastructure. It is the backend for the communication features in both the ERT dashboard and the mobile app: team channels where ground staff share field updates, social community boards where citizens share incident reports and safety information, and voice channels for situations where text is not sufficient.

The service is built around the same Hexagonal Architecture + CQRS pattern used across the Beacon platform. WebSocket connections are managed by the gorilla/websocket library and persist for the lifetime of a user's session. Messages are stored in PostgreSQL and replayed to reconnecting clients so no messages are lost across reconnections.

---

## Architecture

```
Mobile app / Dashboard
      │
      │  WebSocket (ws://beacon-communication-platform/ws)
      │  REST  (http://beacon-communication-platform/api/communication/v1)
      ▼
┌─────────────────────────────────────────────────────────────┐
│  beacon-communication-platform                              │
│                                                             │
│  WebSocket Hub ──────────────────────────────────────────┐  │
│  (gorilla/websocket, one conn per authenticated user)    │  │
│                                                          │  │
│  ┌──────────────────┐  ┌───────────────────┐            │  │
│  │  Message domain  │  │  Social domain    │            │  │
│  │  (channel msgs,  │  │  (posts, comms,   │            │  │
│  │   flagging,      │  │   comments,       │            │  │
│  │   delivery ACK)  │  │   reactions)      │            │  │
│  └──────────────────┘  └───────────────────┘            │  │
│                                                          │  │
│  ┌────────────────────────────────────────────────────┐  │  │
│  │  VoiceChannel domain (Twilio Programmable Voice)   │  │  │
│  └────────────────────────────────────────────────────┘  │  │
└──────────────────────────────────────────────────────────┘  │
      │                                                        │
      ├── PostgreSQL (master/slave — messages, channels,       │
      │                              social posts, voice logs) │
      ├── beacon-event-bus (publish communication events)      │
      ├── beacon-iam (user profile, roles)                     │
      └── Twilio (voice channel management)
```

---

## Domains

### Message

The core real-time messaging domain. A Message is a piece of content (text, image, or location share) sent by a user to a Channel. Channels are persistent conversation rooms scoped to an incident, a team, or a topic.

The message domain handles:
- Delivery acknowledgement tracking (sent / delivered / read receipts)
- Message flagging for moderation (admins can review and remove flagged messages)
- Editing and soft deletion of messages within a time window
- Typing indicators broadcast via WebSocket to other channel participants
- Message history replay for reconnecting clients (the last N messages are sent immediately on WebSocket connection)

### Conversation (Channel)

Channels are the rooms that messages flow through. Each channel has a set of participant users, a type (incident-scoped, team, broadcast), and access control rules. Incident-scoped channels are automatically created when a new incident is approved and archived when it is closed. Team channels are persistent and shared across incidents.

Broadcast channels are one-way: only admins and ERT commanders can post to them, but all platform users can read them. They are used for platform-wide emergency alerts that supplement push notifications.

### Social

Social features support citizen community engagement during and after emergencies. Citizens can create posts with text and images, organize them into communities (neighborhood groups, emergency-type groups), comment on posts, and react with emoji. Social posts are moderated — ERT members and admins can remove posts that contain misinformation or sensitive content.

The news API adapter integrates Dublin Live RSS feed into the social stream so that verified news articles about ongoing city events are surfaced alongside citizen posts.

### Voice Channel

Voice channels are Twilio Programmable Voice conference rooms. Ground staff and ERT commanders can join a voice channel from the mobile app when text coordination is insufficient — for example, during a rapidly evolving incident or when operating in conditions where typing is impractical. The service manages voice channel creation, participant invite and removal, and stores a log of who joined and left each channel.

---

## API Reference

All endpoints are prefixed with `/api/communication/v1`.

### WebSocket

#### `GET /ws`
Upgrades an HTTP connection to WebSocket. Requires a valid JWT in the `Authorization` header (or `token` query parameter for clients that cannot set WebSocket headers). Once connected, the service:

1. Sends a `welcome` frame with the user's current channel memberships
2. Replays the last 50 messages from each active channel the user is a member of
3. Begins broadcasting real-time events: new messages, typing indicators, read receipts, user presence changes

**WebSocket message format:**
```json
{
  "type": "message.new | message.updated | message.deleted | typing | presence | read_receipt",
  "channel_id": "chan-uuid",
  "payload": { ... }
}
```

### Channels

#### `POST /channels`
Creates a new channel. When creating an incident-scoped channel, the `incident_id` field links the channel to the incident lifecycle — the channel will be automatically archived when the incident is closed.

**Request body:**
```json
{
  "name": "Incident-001 Coordination",
  "type": "incident | team | broadcast",
  "incident_id": "inc-uuid",
  "participant_ids": ["user-1", "user-2"],
  "description": "Operations channel for the Dame Street fire"
}
```

#### `GET /channels`
Returns all channels the authenticated user is a participant of. Includes the last message preview and unread count for each channel.

#### `GET /channels/:id`
Returns full channel details including participant list, settings, and the creation context.

#### `POST /channels/:id/join`
Adds the authenticated user to a channel. For invite-only channels, requires an existing participant to have issued an invitation.

#### `DELETE /channels/:id/leave`
Removes the authenticated user from a channel. They will no longer receive messages from this channel.

### Messages

#### `POST /channels/:channel_id/messages`
Sends a message to a channel. Supported message types: `text` (plain text content), `image` (URL to an uploaded image), `location` (lat/lng coordinate share for field responders sharing their position with the team). The message is immediately broadcast via WebSocket to all connected participants.

**Request body:**
```json
{
  "type": "text | image | location",
  "content": "Gate secured, moving to secondary access point",
  "metadata": { "lat": 53.3410, "lng": -6.2675 }
}
```

#### `GET /channels/:channel_id/messages`
Returns paginated message history for a channel. Used to load older messages when scrolling up in the chat view.

**Query parameters:** `before` (message ID cursor for pagination), `limit` (default: 50)

#### `PUT /messages/:id`
Edits the content of a previously sent message. Only the original sender can edit their message. The message record retains the original content and edit timestamp for audit purposes.

#### `DELETE /messages/:id`
Soft-deletes a message. The message record is marked as deleted and its content is replaced with a "message deleted" placeholder, but the record is not removed from the database. Admins can review and permanently purge deleted messages through the admin API.

#### `POST /messages/:id/flag`
Flags a message for moderation review. Any participant can flag a message they believe violates community guidelines. Flagged messages appear in the admin moderation queue. Multiple flags on the same message do not create duplicate entries.

### Social

#### `POST /social/posts`
Creates a social post. Posts support text content, image attachments, and community tags. All posts are immediately visible to other users of the community they are posted to.

#### `GET /social/posts`
Returns a paginated feed of social posts. The default feed combines posts from all communities the user is a member of, ordered by recency. Query parameters allow filtering by community, post type, or keyword search.

#### `POST /social/posts/:id/comments`
Adds a comment to a social post. Comments support the same content types as posts (text, image) but are scoped to the post they belong to.

#### `POST /social/posts/:id/react`
Adds or removes a reaction to a social post. The reaction type must be one of the supported emoji set (`👍`, `❤️`, `🚨`, `🙏`). Calling this endpoint with an emoji the user has already reacted with removes the reaction (toggle behaviour).

### Voice Channels

#### `POST /voice/channels`
Creates a Twilio conference room and returns a join token. Ground staff use this token in the mobile app to connect their device's audio to the voice channel.

#### `POST /voice/channels/:id/invite`
Generates a join token for an additional participant. Used to bring in additional responders mid-coordination.

#### `DELETE /voice/channels/:id`
Terminates an active voice channel and disconnects all participants.

---

## Local Development

```bash
# Install dependencies
go mod download

# Start PostgreSQL
docker-compose up -d postgres

# Run database migrations
make migrate-up

# Configure Twilio (optional — voice features are disabled if not set)
export TWILIO_ACCOUNT_SID=...
export TWILIO_AUTH_TOKEN=...

# Start the service
make run
```

The WebSocket endpoint is available at `ws://localhost:8084/api/communication/v1/ws`. Use a WebSocket client like `websocat` or the browser console to test connections.

---

## Project Structure

```
beacon-communication-platform/
├── cmd/server/main.go           # Composition root
├── internal/
│   ├── domain/
│   │   ├── message/             # Message, channel, delivery receipts, flagging
│   │   ├── social/              # Posts, communities, comments, reactions
│   │   └── voice/               # Twilio conference room management
│   ├── http/
│   │   ├── router.go
│   │   └── handlers/
│   │       ├── websocket_handler.go
│   │       └── socials_handler.go
│   ├── registry/
│   │   ├── commands/
│   │   └── queries/
│   └── adapters/
│       ├── database/            # Postgres master/slave repos
│       ├── microservices/
│       │   └── iam_client.go
│       ├── eventbus/            # Publish communication events
│       ├── twilio/              # Voice channel adapter
│       └── newsapi/             # Dublin Live RSS integration
├── migrations/
└── pkg/
```
