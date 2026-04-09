# beacon-support-bot

AI-powered psychological support and crisis guidance microservice for the Beacon emergency-response platform. beacon-support-bot provides real-time conversational assistance to citizens during emergencies, monitors conversations for signs of psychological distress, and automatically escalates to human support agents when the situation warrants it.

---

## Overview

Emergencies are traumatic. Citizens caught in a flood, fire, or mass-casualty event need more than a list of shelters — they need someone (or something) to talk to while they wait for help. beacon-support-bot fills that gap with a Vertex AI-powered conversational agent that can provide calm, accurate guidance, answer questions about ongoing incidents, and connect distressed users to human support workers.

The bot is not a replacement for emergency services. It does not dispatch responders or make operational decisions. Its role is psychological: to reduce panic, provide verified information, and ensure that citizens in severe distress are identified and connected with real human support before the situation escalates.

---

## Architecture

```
Mobile app / Dashboard (citizen or ERT)
      │
      │  POST /api/support-bot/v1/conversations/:id/messages
      ▼
┌──────────────────────────────────────────────────────┐
│  beacon-support-bot                                  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Conversation domain                           │  │
│  │  (message handling, context window management, │  │
│  │   session continuity)                          │  │
│  └────────────────────┬───────────────────────────┘  │
│                       │                              │
│  ┌────────────────────▼───────────────────────────┐  │
│  │  Risk domain                                   │  │
│  │  (sentiment analysis, distress indicator       │  │
│  │   extraction, severity scoring)                │  │
│  └────────────────────┬───────────────────────────┘  │
│                       │                              │
│  ┌────────────────────▼───────────────────────────┐  │
│  │  Escalation domain                             │  │
│  │  (automatic + manual escalation to human       │  │
│  │   agents, ERT notification)                    │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌──────────────┐  ┌────────────────┐               │
│  │  Wellbeing   │  │  Resource      │               │
│  │  (scheduled  │  │  (self-help    │               │
│  │   follow-ups)│  │   library,     │               │
│  └──────────────┘  │   i18n)        │               │
│                    └────────────────┘               │
└──────────────────────────────────────────────────────┘
      │
      ├── Vertex AI (Google GenAI) — conversation generation, risk analysis
      ├── beacon-core           — incident context retrieval
      ├── beacon-notification   — alert human agents
      └── PostgreSQL            — conversation history, escalations, wellbeing records
```

---

## Domains

### Conversation

Each user session with the support bot is a Conversation — a persistent record of the exchange between the citizen and the AI. Conversations store all messages (user turns and bot responses), the current context state, and metadata about the triggering event (which incident the user is affected by, if known).

The Vertex AI model is given a system prompt that instructs it to act as a calm, empathetic emergency support assistant. It is provided with the current conversation history (up to the model's context window) and, if the user's incident is known, a summary of that incident's current status from beacon-core. This context grounding prevents the bot from providing outdated or generic information when the user asks about their specific emergency.

### Risk

The risk domain runs alongside every conversation. After each user message, the risk analyser classifies the user's emotional state along several dimensions:
- **Sentiment score** — how distressed or calm the message content appears (negative to positive scale)
- **Distress indicators** — specific phrases or patterns associated with panic, physical danger, self-harm, or suicidal ideation
- **Risk severity** — the aggregate classification: low / moderate / high / critical

Risk assessments are not shown to the user. They inform the escalation domain, which decides whether to intervene with additional support resources or alert a human agent.

### Escalation

Escalation handles the critical transition from AI-assisted conversation to human-assisted support. There are two escalation paths:

**Automatic escalation** is triggered when the risk domain reports a high or critical severity classification. The system immediately:
1. Notifies an available human support agent via beacon-notification (push + SMS)
2. Sends the agent the conversation transcript and the risk assessment
3. Informs the user that a human agent is being connected
4. Pauses AI responses on that conversation so the agent can take over

**Manual escalation** is triggered when the user explicitly asks to speak to a human, or when an ERT member reviewing the bot's conversation queue decides to intervene.

Both escalation paths create an Escalation record with timestamps, agent assignment, and resolution notes. ERT commanders can review open escalations in the dashboard.

### Wellbeing

After a major incident is resolved, the wellbeing domain schedules follow-up check-ins with citizens who were significantly involved (e.g., they reported the incident, or were in the evacuated zone). Check-ins are short bot-initiated conversations that ask how the person is coping and offer links to longer-term support resources. They are scheduled through the beacon-scheduler library to fire 24 hours, 3 days, and 7 days after the incident closes.

Wellbeing conversations follow the same risk analysis and escalation pipeline as emergency conversations — a delayed stress response is just as real as an acute one.

### Resource

The resource domain stores a library of self-help content: breathing exercises, grounding techniques, mental health hotline numbers, local shelter addresses, and emergency procedure guides. Resources are tagged by category, incident type, and locale.

The AI model is instructed to include relevant resource links in its responses when appropriate. Resources are also served directly through the REST API so the mobile app can display a help library screen independent of the bot conversation.

---

## API Reference

All endpoints are prefixed with `/api/support-bot/v1`.

### Conversations

#### `POST /conversations`
Creates a new support bot conversation session. Optionally associates the conversation with a specific incident if the user is affected by a known emergency.

**Request body:**
```json
{
  "incident_id": "inc-uuid",
  "locale": "en",
  "context": "citizen_evacuation | ert_support | general"
}
```

**Response:** `{ "conversation_id": "...", "greeting": "Hello, I'm here to help..." }`

The `context` field tailors the AI system prompt. `citizen_evacuation` prompts the model to focus on safety instructions and emotional support. `ert_support` shifts the model toward operational guidance for trained responders. `general` is for non-emergency wellbeing queries.

#### `POST /conversations/:id/messages`
Sends a message to the bot and receives a response. This is the primary endpoint used by the mobile app's chat screen. The call is synchronous — it waits for the Vertex AI model to generate a response (typically 1–3 seconds) before returning.

**Request body:** `{ "message": "I'm scared, what do I do?" }`

**Response:**
```json
{
  "message_id": "msg-uuid",
  "response": "I understand you're frightened. Let's focus on what you can do right now...",
  "resources": [
    { "type": "breathing_exercise", "title": "4-7-8 Breathing", "url": "..." }
  ],
  "risk_level": "moderate",
  "escalated": false
}
```

The `risk_level` field in the response allows the mobile app to adjust its UI — for example, displaying emergency contact buttons more prominently if the risk level is high.

#### `GET /conversations/:id`
Returns the full conversation history with all messages, timestamps, and risk assessments. Used by the ERT dashboard to review a citizen's conversation when an escalation alert is received.

#### `GET /conversations`
Returns a paginated list of conversations for the authenticated user (citizen) or all open escalated conversations (ERT/admin). ERT members see only escalated conversations pending human agent assignment.

### Escalations

#### `POST /conversations/:id/escalate`
Manually escalates a conversation to a human agent. Used when the user explicitly requests human support or when an ERT member reviewing the conversation decides to intervene. Returns the escalation record including estimated wait time.

#### `GET /escalations` *(ERT and Admin only)*
Returns all open escalations sorted by urgency (critical risk escalations first, then high, then moderate). Used by the ERT dashboard's support queue view.

#### `PUT /escalations/:id/assign`
Assigns an escalation to the authenticated ERT member or support agent. Once assigned, the agent receives the full conversation transcript and takes over the human response.

#### `PUT /escalations/:id/resolve`
Marks an escalation as resolved with an optional resolution note. After resolution, the AI conversation can optionally resume for lower-priority queries.

### Risk Analysis

#### `POST /risk/analyse`
Analyses a text message for distress indicators and returns a risk assessment. This endpoint is used internally by the conversation pipeline but is also exposed for ERT members to analyse text from other sources (e.g., social media posts flagged by the communication platform).

**Request body:** `{ "text": "...", "context": "..." }`

**Response:**
```json
{
  "sentiment_score": -0.72,
  "risk_severity": "high",
  "distress_indicators": ["expressions of fear", "mentions of physical danger"],
  "recommended_action": "escalate_to_human"
}
```

### Wellbeing

#### `GET /wellbeing/check-ins`
Returns scheduled and completed wellbeing check-ins for the authenticated user. Citizens can see their upcoming check-ins and opt out if they do not wish to be contacted.

#### `DELETE /wellbeing/check-ins/:id`
Cancels a scheduled wellbeing check-in. The user's opt-out preference is stored so that future check-ins for the same incident are also cancelled.

### Resources

#### `GET /resources`
Returns the self-help resource library. Supports filtering by category, incident type, and locale.

**Query parameters:** `category` (breathing|grounding|hotlines|procedures|shelters), `locale` (en|ga), `incident_type`

#### `GET /resources/:id`
Returns a specific resource with full content.

---

## Vertex AI Integration

beacon-support-bot uses the Vertex AI Go SDK (`cloud.google.com/go/vertexai v0.15.0`) to access Google's Gemini model family. The model is chosen based on the use case:

- **Conversation generation**: Gemini Pro — high-quality, nuanced responses for the citizen-facing chat
- **Risk analysis**: Gemini Flash — faster and cheaper for the high-frequency classification task that runs on every message
- **Resource matching**: Embedding model — semantic search to match user queries to the most relevant resources

The Vertex AI client is initialized with the service account credentials from the GKE Workload Identity binding. The GCP project and model location (`europe-west1`) are configured via environment variables. A custom system prompt for each conversation context is stored in the database and injected at runtime, allowing ERT administrators to tune the bot's behaviour without code changes.

---

## Local Development

```bash
# Install dependencies
go mod download

# Start PostgreSQL
docker-compose up -d postgres

# Run database migrations
make migrate-up

# Authenticate with GCP (for Vertex AI access)
gcloud auth application-default login

# Set required environment variables
export GCP_PROJECT_ID=my-gcp-project
export VERTEX_AI_LOCATION=europe-west1
export BEACON_CORE_BASE_URL=http://localhost:8080

# Start the service
make run
```

---

## Project Structure

```
beacon-support-bot/
├── cmd/server/main.go           # Composition root
├── internal/
│   ├── domain/
│   │   ├── conversation/        # Message handling, context window, AI invocation
│   │   ├── risk/                # Sentiment analysis, distress detection
│   │   ├── escalation/          # Automatic + manual escalation to human agents
│   │   ├── wellbeing/           # Scheduled follow-up check-ins
│   │   └── resource/            # Self-help content library
│   ├── http/
│   │   ├── router.go
│   │   └── handlers/
│   ├── registry/
│   │   ├── commands/
│   │   └── queries/
│   └── adapters/
│       ├── database/            # PostgreSQL repositories
│       ├── microservices/
│       │   ├── genai_client.go  # Vertex AI adapter
│       │   ├── beacon_api_client.go          # beacon-core incident context
│       │   └── notification_service_client.go
│       └── i18n/                # Multilingual content support
├── migrations/
└── pkg/
```
