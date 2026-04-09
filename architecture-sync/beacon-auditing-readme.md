# beacon-auditing

[![CI](https://github.com/Response-To-City-Disaster/beacon-auditing/actions/workflows/ci.yml/badge.svg)](https://github.com/Response-To-City-Disaster/beacon-auditing/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/Response-To-City-Disaster/beacon-auditing.svg)](https://pkg.go.dev/github.com/Response-To-City-Disaster/beacon-auditing)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A production-grade, non-blocking audit logging library for Go. Built for the Beacon emergency-response platform, it provides a high-throughput interface for recording structured audit events into ClickHouse with automatic batching, graceful degradation under load, and rich observability hooks.

---

## Why This Library Exists

Every meaningful state mutation in the Beacon platform — approving an incident, dispatching a ground-staff team, broadcasting an evacuation advisory — requires an immutable audit trail. `beacon-auditing` is the library all Beacon microservices import to record those events without adding latency to the primary request path.

The central design decision is that **audit logging must never block the caller**. Events are accepted into a buffered in-process channel and flushed to ClickHouse in background goroutines on a time-and-count schedule. If the buffer fills because ClickHouse is temporarily unavailable, the library returns a typed `ErrBufferFull` error so the caller can choose to log and continue — the user-facing request is never held up waiting for an analytics write to complete.

ClickHouse is the right backend here because it is column-oriented and purpose-built for high-volume append workloads and analytical time-range scans. The `audit_events` table carries a one-year TTL, so records expire automatically without any maintenance job.

---

## Architecture

```
Caller goroutine
      │
      │  client.LogAuditEvent(ctx, event)
      ▼
┌─────────────────────┐
│  Validation         │  Checks required fields synchronously (no I/O)
└──────────┬──────────┘
           │  non-blocking channel send
           ▼
┌─────────────────────┐
│  Buffered channel   │  Default capacity: 1000 events
│  (in-process queue) │  Returns ErrBufferFull immediately if at capacity
└──────────┬──────────┘
           │  background goroutine drains the queue
           ▼
┌─────────────────────┐
│  Batch assembler    │  Accumulates up to AUDIT_BATCH_SIZE events
│                     │  OR waits AUDIT_FLUSH_INTERVAL, whichever fires first
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  ClickHouse         │  Batch INSERT INTO audit_events (TCP or HTTP)
│  (TCP/8443 HTTPS)   │
└─────────────────────┘
```

The dual-trigger flush strategy (count _or_ time) ensures the system behaves well under both high throughput (batches fill quickly and flush often) and low throughput (partial batches are never held indefinitely — they flush at the interval boundary).

---

## Installation

```bash
go get github.com/Response-To-City-Disaster/beacon-auditing
```

No CGO dependencies. Safe to use from multiple goroutines without external synchronization.

---

## Quick Start

```go
package main

import (
    "context"
    "log"
    "time"

    auditing "github.com/Response-To-City-Disaster/beacon-auditing"
)

func main() {
    ctx := context.Background()

    // Build the client from environment variables (see Configuration section)
    client, err := auditing.NewClientFromEnv(ctx)
    if err != nil {
        log.Fatalf("audit: connect failed: %v", err)
    }
    defer client.Close()

    // Create the audit_events table if it doesn't exist (idempotent)
    if err := client.InitSchema(ctx); err != nil {
        log.Fatalf("audit: schema init failed: %v", err)
    }

    // Enqueue an audit event — returns immediately, never blocks on ClickHouse
    err = client.LogAuditEvent(ctx, auditing.AuditEvent{
        ServiceName:  "beacon-core",
        ActorID:      "user-f4e3b29a",
        ActorRole:    "Admin",
        Action:       "incident.approved",
        ResourceType: "incident",
        ResourceID:   "inc-9c1d72e8",
        Metadata:     map[string]interface{}{"severity": "P0", "location": "Dublin 2"},
        OccurredAt:   time.Now(),
    })
    if auditing.IsBufferFull(err) {
        log.Printf("audit buffer full — ClickHouse may be degraded, event dropped")
        // Do not return an error to the HTTP caller; the audit trail is best-effort
    } else if err != nil {
        log.Fatalf("audit: unexpected error: %v", err)
    }
}
```

---

## Client Construction

### `NewClientFromEnv(ctx context.Context) (*Client, error)`

Creates a client from environment variables. This is the standard approach for containerized deployments where secrets are injected via Kubernetes `secretKeyRef`.

```bash
CLICKHOUSE_HOST=clickhouse.internal
CLICKHOUSE_PORT=9000                   # 8443 for ClickHouse Cloud HTTPS
CLICKHOUSE_DATABASE=beacon_audit
CLICKHOUSE_USERNAME=audit_user
CLICKHOUSE_PASSWORD=s3cr3t
CLICKHOUSE_SECURE=false                # true for ClickHouse Cloud
AUDIT_BUFFER_SIZE=1000                 # events held in memory before ErrBufferFull
AUDIT_BATCH_SIZE=100                   # events per ClickHouse INSERT
AUDIT_FLUSH_INTERVAL=5s                # max wait before flushing a partial batch
```

| Variable | Default | Description |
|---|---|---|
| `CLICKHOUSE_HOST` | `localhost` | ClickHouse server hostname |
| `CLICKHOUSE_PORT` | `9000` | TCP port. Use `8443` for ClickHouse Cloud |
| `CLICKHOUSE_DATABASE` | `default` | Database to write events into |
| `CLICKHOUSE_USERNAME` | `default` | ClickHouse username |
| `CLICKHOUSE_PASSWORD` | *(empty)* | ClickHouse password |
| `CLICKHOUSE_SECURE` | `false` | Enable TLS. Required for ClickHouse Cloud |
| `AUDIT_BUFFER_SIZE` | `1000` | In-memory queue depth before `LogAuditEvent` returns `ErrBufferFull` |
| `AUDIT_BATCH_SIZE` | `100` | Number of events per ClickHouse INSERT statement |
| `AUDIT_FLUSH_INTERVAL` | `5s` | Maximum time before a partial batch is flushed |

### `NewClient(ctx context.Context, opts ...Option) (*Client, error)`

Creates a client using functional options. Use this when configuration is already parsed from a config struct rather than environment variables.

```go
client, err := auditing.NewClient(ctx,
    auditing.WithClickHouseHost("clickhouse.internal"),
    auditing.WithClickHousePort(9000),
    auditing.WithDatabase("beacon_audit"),
    auditing.WithCredentials("audit_user", "s3cr3t"),
    auditing.WithBatchSize(200),
    auditing.WithBufferSize(5000),
    auditing.WithFlushInterval(10 * time.Second),
    auditing.WithSecureConnection(true),
)
```

---

## Core API

### `LogAuditEvent(ctx context.Context, event AuditEvent) error`

Enqueues an audit event for background delivery. The call validates required fields synchronously, then places the event on the internal channel and returns. It does not wait for the event to reach ClickHouse.

**Required fields** (returns `ErrValidation` if any are missing or zero):
- `ServiceName` — the Beacon microservice emitting the event (e.g., `"beacon-core"`)
- `ActorID` — UUID of the user or service principal that triggered the action
- `Action` — the operation in `domain.verb` format (e.g., `"incident.approved"`, `"dispatch.created"`)
- `OccurredAt` — the time the action actually happened (not the time of the function call)

**Error behaviour:**
- `ErrBufferFull` — the internal channel is at capacity. This means ClickHouse is unavailable or the flush goroutine is congested. The caller should log the dropped event but continue serving the primary request.
- `ErrValidation` — a required field is missing. This is a programming error and should be treated as fatal in tests.
- `nil` — the event was accepted into the queue. Delivery to ClickHouse is not yet guaranteed, but will happen within `AUDIT_FLUSH_INTERVAL`.

### `QueryAuditLog(ctx context.Context, filter AuditFilter) ([]AuditEvent, error)`

Executes a synchronous SELECT against `audit_events`. Intended for audit dashboards and compliance queries — not for the hot request path. Supports filtering by service name, actor, resource type, resource ID, and time range, with limit and offset for pagination.

```go
events, err := client.QueryAuditLog(ctx, auditing.AuditFilter{
    ServiceName:  "beacon-core",
    ResourceType: "incident",
    From:         time.Now().Add(-24 * time.Hour),
    To:           time.Now(),
    Limit:        100,
    Offset:       0,
})
```

All filter fields are optional. Results are ordered by `occurred_at` descending (most recent first).

### `InitSchema(ctx context.Context) error`

Creates the `audit_events` table in ClickHouse if it does not already exist. The DDL uses `ReplicatedMergeTree` for clustered deployments, partitions by month (`toYYYYMM(occurred_at)`) for efficient time pruning, and adds a `TTL occurred_at + INTERVAL 1 YEAR DELETE` clause so records are automatically purged after one year with no maintenance job. Safe to call on every startup — it is fully idempotent.

### `Close() error`

Signals the background flush goroutine to drain the queue and shut down. Waits for any in-flight batches to complete up to a configurable shutdown timeout, then closes the ClickHouse connection. Always call this during graceful shutdown — events remaining in the buffer at process exit are lost.

---

## AuditEvent Schema

```go
type AuditEvent struct {
    ID           string                 // Auto-generated UUID v4 if left empty
    ServiceName  string                 // Required. Which Beacon service is logging
    ActorID      string                 // Required. UUID of the user or system agent
    ActorRole    string                 // Optional. Role at the time of the action
    Action       string                 // Required. "domain.verb" — e.g. "dispatch.accepted"
    ResourceType string                 // Optional. Entity type — e.g. "incident", "dispatch"
    ResourceID   string                 // Optional. UUID of the affected entity
    Metadata     map[string]interface{} // Optional. Arbitrary structured context for the event
    IPAddress    string                 // Optional. Client IP for HTTP-originated actions
    OccurredAt   time.Time              // Required. When the action happened
}
```

---

## Observability Hooks

Three callbacks let callers instrument the audit pipeline without polling. They are invoked from the background flush goroutine, so any shared state accessed inside them must be synchronized.

```go
// Called every time an event is successfully enqueued (before flush)
client.SetOnEventLogged(func(event auditing.AuditEvent) {
    auditEnqueuedCounter.Inc()
})

// Called each time a batch is successfully flushed to ClickHouse
client.SetOnFlush(func(batchSize int, duration time.Duration) {
    auditFlushSizeHistogram.Observe(float64(batchSize))
    auditFlushLatencyHistogram.Observe(float64(duration.Milliseconds()))
})

// Called when a batch fails to flush — includes the dropped events for logging
client.SetOnError(func(err error, droppedBatch []auditing.AuditEvent) {
    logger.Error().Err(err).Int("dropped", len(droppedBatch)).Msg("audit flush failed")
    auditFlushErrorCounter.Inc()
})
```

---

## Error Types

The library uses typed sentinel errors. Avoid string-matching `err.Error()` — use the provided predicates:

```go
if auditing.IsBufferFull(err) {
    // ClickHouse is degraded. Log and continue — do not fail the HTTP request.
}
if auditing.IsValidationError(err) {
    // Required fields missing — programming error, should never reach production.
}
if auditing.IsConnectionError(err) {
    // Could not connect at startup. Service should fail to start.
}
if auditing.IsSchemaError(err) {
    // InitSchema failed — table creation error. Service should fail to start.
}
if auditing.IsQueryError(err) {
    // QueryAuditLog read failed — ClickHouse query error.
}
```

---

## Testing With the Mock Client

The package ships a `mock` sub-package with an in-memory client that implements the same interface. Use it in unit tests to assert that audit events are emitted with the correct fields without requiring a running ClickHouse instance.

```go
import "github.com/Response-To-City-Disaster/beacon-auditing/mock"

func TestApproveIncidentAuditsCorrectly(t *testing.T) {
    mockAudit := mock.NewClient()

    svc := incident.NewService(mockRepo, mockAudit)
    err := svc.ApproveIncident(ctx, "inc-001", "admin-007")
    require.NoError(t, err)

    logged := mockAudit.GetLoggedEvents()
    require.Len(t, logged, 1)
    assert.Equal(t, "incident.approved", logged[0].Action)
    assert.Equal(t, "inc-001", logged[0].ResourceID)
    assert.Equal(t, "admin-007", logged[0].ActorID)
}
```

---

## How Beacon Services Use This Library

All Beacon microservices inject the audit client at startup through their domain services. In `beacon-core`, the wiring lives in `cmd/server/main.go`:

```go
auditClient, err := auditing.NewClientFromEnv(ctx)
if err != nil {
    logger.Fatal().Err(err).Msg("failed to connect to audit ClickHouse")
}
if err := auditClient.InitSchema(ctx); err != nil {
    logger.Fatal().Err(err).Msg("failed to initialize audit schema")
}

incidentService.SetAuditClient(auditClient)
dispatchService.SetAuditClient(auditClient)
evacuationService.SetAuditClient(auditClient)
```

The domain service calls `LogAuditEvent` after each successful state transition. If ClickHouse is degraded, the event is dropped and a warning is logged — the state transition itself is not rolled back.

---

## License

MIT — see [LICENSE](LICENSE) for details.
