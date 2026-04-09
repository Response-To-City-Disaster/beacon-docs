# beacon-event-bus

[![CI](https://github.com/Response-To-City-Disaster/beacon-event-bus/actions/workflows/ci.yml/badge.svg)](https://github.com/Response-To-City-Disaster/beacon-event-bus/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/Response-To-City-Disaster/beacon-event-bus.svg)](https://pkg.go.dev/github.com/Response-To-City-Disaster/beacon-event-bus)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A production-grade Go library for publishing and subscribing to events over GCP Pub/Sub, purpose-built for the Beacon emergency-response platform. It wraps the Pub/Sub SDK with a CloudEvents-compatible event model, priority levels, circuit breaker protection, exponential backoff retry, and a mock backend for hermetic testing.

---

## Why This Library Exists

The Beacon platform uses an asynchronous event-driven architecture to decouple services. When `beacon-core` approves an incident, it does not call `beacon-notification` directly — it publishes an `incident.approved` event to the event bus, and any interested service consumes it independently. This means `beacon-core` is never held up waiting for push notifications to be sent, and new consumers can be added without modifying the publisher.

`beacon-event-bus` is the shared library that all Beacon services import to participate in this event bus. It provides a single, consistent interface so that every service publishes and subscribes the same way — with the same error handling semantics, the same retry policy, and the same CloudEvents envelope format.

---

## Architecture

```
Publisher service                          Consumer service
      │                                          │
      │  client.Publish(ctx, topic, event)       │  client.Subscribe(ctx, sub, handler)
      ▼                                          ▼
┌──────────────────┐                   ┌──────────────────┐
│  Circuit Breaker │                   │  Filter engine   │
│  (open/half/     │                   │  (type, source,  │
│   closed)        │                   │   attributes)    │
└──────────┬───────┘                   └──────────┬───────┘
           │                                      │
           ▼                                      ▼
┌──────────────────┐                   ┌──────────────────┐
│  Retry policy    │                   │  Handler func    │
│  (exp. backoff)  │                   │  (user code)     │
└──────────┬───────┘                   └──────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  GCP Pub/Sub                         │
│  (or local emulator for dev/test)    │
└──────────────────────────────────────┘
```

The circuit breaker prevents cascading failures when Pub/Sub is degraded: after a configurable number of consecutive publish failures, the breaker opens and new publish calls fail immediately (without attempting the network) until a cooldown period passes. This protects the caller from accumulating goroutines blocked on a degraded transport.

---

## Installation

```bash
go get github.com/Response-To-City-Disaster/beacon-event-bus
```

Requires Go 1.21+ and a GCP project with the Pub/Sub API enabled. For local development, the Pub/Sub emulator is supported via the `EVENTBUS_EMULATOR_HOST` environment variable.

---

## Quick Start

```go
package main

import (
    "context"
    "log"
    "time"

    eventbus "github.com/Response-To-City-Disaster/beacon-event-bus"
)

func main() {
    ctx := context.Background()

    // Create a client (reads project ID and credentials from environment)
    client, err := eventbus.New(ctx)
    if err != nil {
        log.Fatalf("eventbus: connect failed: %v", err)
    }
    defer client.Close()

    // Publish an event
    err = client.Publish(ctx, "beacon-incidents", eventbus.Event{
        Type:    "incident.approved",
        Source:  "beacon-core",
        Subject: "inc-9c1d72e8",
        Data: map[string]interface{}{
            "incident_id": "inc-9c1d72e8",
            "severity":    "P0",
            "approved_by": "admin-f4e3b29a",
        },
        Priority: eventbus.PriorityHigh,
        Time:     time.Now(),
    })
    if err != nil {
        log.Printf("eventbus: publish failed: %v", err)
    }
}
```

---

## Configuration

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `EVENTBUS_PROJECT_ID` | Yes | GCP project ID that owns the Pub/Sub topics |
| `EVENTBUS_CREDENTIALS_FILE` | No | Path to a GCP service account JSON key. If omitted, Application Default Credentials are used (recommended in GKE) |
| `EVENTBUS_EMULATOR_HOST` | No | Set to `localhost:8085` to use the Pub/Sub emulator instead of GCP (for local dev and CI) |

In production (GKE), the pod's Workload Identity service account is used automatically when `EVENTBUS_CREDENTIALS_FILE` is not set. This is the preferred authentication method because it avoids managing long-lived key files.

### Programmatic Configuration

```go
client, err := eventbus.New(ctx,
    eventbus.WithProjectID("my-gcp-project"),
    eventbus.WithCredentialsFile("/var/secrets/sa.json"),
    eventbus.WithRetryPolicy(eventbus.RetryPolicy{
        MaxAttempts:     5,
        InitialInterval: 100 * time.Millisecond,
        MaxInterval:     30 * time.Second,
        Multiplier:      2.0,
    }),
    eventbus.WithCircuitBreaker(eventbus.CircuitBreakerConfig{
        Threshold:   10,             // open after 10 consecutive failures
        Timeout:     60 * time.Second,
    }),
    eventbus.WithDeliveryGuarantee(eventbus.AtLeastOnce),
)
```

---

## Event Model

Every event published through this library is wrapped in a CloudEvents-compatible envelope:

```go
type Event struct {
    // Routing and identification
    ID            string            // Auto-generated UUID v4 if empty
    Type          string            // Required. "domain.verb" — e.g. "incident.approved"
    Source        string            // Required. Service emitting the event — e.g. "beacon-core"
    Subject       string            // Optional. Primary entity ID — e.g. "inc-9c1d72e8"

    // Payload
    Data          interface{}       // Required. Arbitrary structured data (JSON-serialized)

    // Metadata
    Time          time.Time         // Event timestamp (defaults to time.Now() if zero)
    Priority      Priority          // Low | Normal | High | Critical
    Headers       map[string]string // Optional. Custom message attributes for routing/filtering
    TraceID       string            // Optional. Distributed trace identifier
    CorrelationID string            // Optional. Correlation ID for saga/request tracking

    // Delivery
    DeliveryGuarantee DeliveryGuarantee // AtLeastOnce (default) | ExactlyOnce
}
```

### Priority Levels

Priority is stored as a Pub/Sub message attribute and can be used by consumers to prioritize processing or route to dedicated workers.

| Constant | Value | Intended Use |
|---|---|---|
| `eventbus.PriorityLow` | `"low"` | Background analytics, non-urgent notifications |
| `eventbus.PriorityNormal` | `"normal"` | Standard lifecycle events |
| `eventbus.PriorityHigh` | `"high"` | Dispatch updates, evacuation advisories |
| `eventbus.PriorityCritical` | `"critical"` | SOS events, mass casualty incidents |

### Delivery Guarantees

| Constant | Behaviour |
|---|---|
| `eventbus.AtLeastOnce` | Standard Pub/Sub delivery — message may be delivered more than once. Consumers must be idempotent. Default for all Beacon services. |
| `eventbus.ExactlyOnce` | Enables Pub/Sub exactly-once delivery. Requires the subscription to have exactly-once delivery enabled in GCP. Higher latency. |

---

## Publishing

### `Publish(ctx, topic, event) error`

Publishes a single event to a Pub/Sub topic. The event is serialized to JSON and published with metadata stored as Pub/Sub message attributes (type, source, subject, priority, trace ID, correlation ID).

The call respects the circuit breaker state: if the breaker is open (too many recent failures), the call returns an error immediately without attempting a network connection. If the circuit is closed, the call attempts the publish with exponential backoff retry on transient errors.

```go
err := client.Publish(ctx, "beacon-evacuations", eventbus.Event{
    Type:     "evacuation.advisory.broadcast",
    Source:   "beacon-core",
    Subject:  "evac-zone-7b",
    Data:     advisoryPayload,
    Priority: eventbus.PriorityCritical,
})
if eventbus.IsCircuitOpen(err) {
    logger.Warn().Msg("event bus circuit open — advisory dropped")
}
```

### `PublishBatch(ctx, topic, events []Event) error`

Publishes multiple events to the same topic in a single batch operation. More efficient than calling `Publish` in a loop for bulk scenarios like publishing a history of events during incident replay.

---

## Subscribing

### `Subscribe(ctx, subscriptionID, handler, opts...) error`

Starts consuming messages from a Pub/Sub subscription. Blocks until the context is cancelled. The handler function is called for each received message.

```go
err := client.Subscribe(ctx, "beacon-notification-incidents", func(ctx context.Context, msg eventbus.Message) error {
    var payload IncidentApprovedPayload
    if err := json.Unmarshal(msg.Data, &payload); err != nil {
        return err // returning an error NACKs the message; it will be redelivered
    }

    if err := sendPushNotifications(ctx, payload); err != nil {
        return err
    }

    return nil // returning nil ACKs the message
}, eventbus.WithFilter(eventbus.Filter{
    Types: []string{"incident.approved", "incident.closed"},
}))
```

Returning a non-nil error from the handler NACKs the Pub/Sub message, which causes it to be redelivered after the subscription's acknowledgement deadline. Returning nil ACKs it and removes it from the queue.

### Message Filtering

Filters are evaluated before the handler is called, allowing a consumer to express interest in a subset of event types on a topic without building that logic into the handler:

```go
eventbus.WithFilter(eventbus.Filter{
    Types:   []string{"incident.approved"},           // exact or prefix match
    Sources: []string{"beacon-core"},                  // only events from this source
    Attributes: map[string]string{
        "severity": "P0",                              // only P0 incidents
    },
})
```

Wildcard patterns are supported for type matching: `"incident.*"` matches `"incident.approved"`, `"incident.closed"`, etc.

---

## Topic Management

```go
// Create a topic (idempotent)
err := client.CreateTopic(ctx, "beacon-new-domain")

// Check if a topic exists
exists, err := client.TopicExists(ctx, "beacon-incidents")

// List all topics in the project
topics, err := client.ListTopics(ctx)

// Delete a topic (use with caution in production)
err := client.DeleteTopic(ctx, "beacon-old-domain")
```

---

## Error Handling

```go
if eventbus.IsCircuitOpen(err) {
    // The circuit breaker is open — Pub/Sub has been failing recently.
    // Log the dropped event and continue; do not fail the primary request.
}
if eventbus.IsTransient(err) {
    // Network timeout, quota exceeded, etc. The retry policy already
    // exhausted its attempts. Consider whether the caller should retry.
}
if eventbus.IsPublishFailed(err) {
    // Pub/Sub rejected the message (e.g. topic does not exist, payload too large).
    // This is a configuration or programming error, not a transient failure.
}
```

---

## Testing With the Mock Client

The `mock` sub-package provides an in-memory event bus that records all published events. Use it in unit tests to assert that the correct events are emitted without a real GCP connection:

```go
import "github.com/Response-To-City-Disaster/beacon-event-bus/mock"

func TestDispatchAcceptancePublishesEvent(t *testing.T) {
    mockBus := mock.New()

    svc := dispatch.NewService(mockRepo, mockBus)
    err := svc.AcceptDispatch(ctx, "disp-001", "staff-007")
    require.NoError(t, err)

    published := mockBus.GetPublishedEvents("beacon-dispatches")
    require.Len(t, published, 1)
    assert.Equal(t, "dispatch.accepted", published[0].Type)
    assert.Equal(t, "disp-001", published[0].Subject)
}
```

For integration tests against a real Pub/Sub environment, set `EVENTBUS_EMULATOR_HOST=localhost:8085` and start the emulator with:

```bash
gcloud beta emulators pubsub start --project=local-test
```

---

## Events Published by Beacon Services

The following event types flow through the bus in production:

| Topic | Event Type | Published By | Consumed By |
|---|---|---|---|
| `beacon-incidents` | `incident.created` | beacon-core | beacon-notification, beacon-dashboard |
| `beacon-incidents` | `incident.approved` | beacon-core | beacon-notification, beacon-dashboard |
| `beacon-incidents` | `incident.closed` | beacon-core | beacon-notification, beacon-dashboard |
| `beacon-dispatches` | `dispatch.accepted` | beacon-core | beacon-notification |
| `beacon-dispatches` | `dispatch.completed` | beacon-core | beacon-notification |
| `beacon-evacuations` | `evacuation.advisory.broadcast` | beacon-core | beacon-notification |

---

## License

MIT — see [LICENSE](LICENSE) for details.
