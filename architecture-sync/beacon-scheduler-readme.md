# beacon-scheduler

[![Go Reference](https://pkg.go.dev/badge/github.com/Response-To-City-Disaster/beacon-scheduler.svg)](https://pkg.go.dev/github.com/Response-To-City-Disaster/beacon-scheduler)
[![Go Report Card](https://goreportcard.com/badge/github.com/Response-To-City-Disaster/beacon-scheduler)](https://goreportcard.com/report/github.com/Response-To-City-Disaster/beacon-scheduler)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A Go adapter library for GCP Cloud Scheduler that makes it straightforward to create, manage, and monitor scheduled jobs from application code. Built for the Beacon emergency-response platform to handle time-based notification delivery, scheduled reports, and maintenance jobs.

---

## Why This Library Exists

GCP Cloud Scheduler is a managed cron service that triggers Pub/Sub messages or HTTP endpoints on a schedule. Using its Go SDK directly requires significant boilerplate: project and location configuration, job name construction, client lifecycle management, and interpreting gRPC errors. `beacon-scheduler` wraps all of that behind a clean, testable interface that Beacon services can call in a few lines.

The library follows the adapter pattern: it defines a `SchedulerClient` interface, provides a real implementation backed by GCP, and ships a mock implementation for unit tests. Services depend on the interface, not the implementation, making it straightforward to test scheduling logic without real GCP credentials.

---

## Installation

```bash
go get github.com/Response-To-City-Disaster/beacon-scheduler
```

Requires Go 1.21+ and a GCP project with the Cloud Scheduler API enabled. Application Default Credentials or a service account key file are used for authentication.

---

## Quick Start

```go
package main

import (
    "context"
    "log"
    "time"

    scheduler "github.com/Response-To-City-Disaster/beacon-scheduler"
)

func main() {
    ctx := context.Background()

    // Create the scheduler client
    sched, err := scheduler.New(ctx, scheduler.Config{
        ProjectID: "my-gcp-project",
        Location:  "europe-west1",
    })
    if err != nil {
        log.Fatalf("scheduler: connect failed: %v", err)
    }

    // Schedule a message to be published to a Pub/Sub topic at a specific time
    job, err := sched.Schedule(ctx, scheduler.JobRequest{
        Payload:       []byte(`{"type":"notification.batch","batch_id":"batch-001"}`),
        ScheduledTime: time.Now().Add(30 * time.Minute).Format(time.RFC3339),
        TargetTopic:   "beacon-notifications",
    })
    if err != nil {
        log.Fatalf("scheduler: failed to create job: %v", err)
    }
    log.Printf("scheduled job: %s", job.Name)
}
```

---

## Configuration

### `Config` Struct

```go
type Config struct {
    ProjectID     string                     // Required. GCP project that owns the scheduler
    Location      string                     // Required. Cloud Scheduler region, e.g. "europe-west1"
    ClientOptions []option.ClientOption      // Optional. GCP client options (credentials, endpoint, etc.)
}
```

The location must match the region where your Pub/Sub topics and subscribers run. Using a different region adds cross-region latency to each scheduled trigger.

### Authentication

In GKE with Workload Identity configured, no explicit credentials are needed — the pod's service account is used automatically. For local development:

```bash
gcloud auth application-default login
```

Or provide a service account key file:

```go
import "google.golang.org/api/option"

sched, err := scheduler.New(ctx, scheduler.Config{
    ProjectID: "my-gcp-project",
    Location:  "europe-west1",
    ClientOptions: []option.ClientOption{
        option.WithCredentialsFile("/var/secrets/sa.json"),
    },
})
```

---

## Core API

### `Schedule(ctx, req JobRequest) (*Job, error)`

Creates a new Cloud Scheduler job that publishes a message to a Pub/Sub topic at a given time. This is the primary operation used by `beacon-notification` to defer notification delivery — for example, scheduling a follow-up wellness check 24 hours after an incident is closed.

```go
type JobRequest struct {
    Payload       []byte // Required. The message body to publish to the target topic
    ScheduledTime string // Required. RFC3339 timestamp for when the job should fire
    TargetTopic   string // Required. Pub/Sub topic to publish to (short name, not full path)
    Description   string // Optional. Human-readable job description for Cloud Console
}
```

The `ScheduledTime` must be a valid RFC3339 timestamp in the future. Scheduling in the past returns `ErrInvalidScheduleTime`.

### `GetJob(ctx, jobName string) (*Job, error)`

Retrieves a Cloud Scheduler job by its fully-qualified name. Returns `ErrJobNotFound` if the job does not exist or has been deleted. Use this to check the status of a scheduled job before cancelling it.

```go
job, err := sched.GetJob(ctx, "projects/my-project/locations/europe-west1/jobs/job-uuid")
if scheduler.IsNotFound(err) {
    // The job has already run or was cancelled
}
```

### `ListJobs(ctx) ([]*Job, error)`

Returns all jobs in the configured project and location. Useful for operational monitoring — `beacon-notification` can use this to list all pending scheduled batches.

### `DeleteJob(ctx, jobName string) error`

Cancels and deletes a scheduled job. Use this when a notification batch is cancelled before its scheduled delivery time. Returns `ErrJobNotFound` if the job does not exist (this is treated as a no-op in most callers, since the goal — the job not running — is already achieved).

---

## Error Handling

The library exposes typed sentinel errors and predicate functions so callers can handle each case without string matching:

| Error | Predicate | Meaning |
|---|---|---|
| `ErrInvalidPayload` | `scheduler.IsInvalidPayload(err)` | Payload is nil or exceeds Cloud Scheduler's 1 MB message limit |
| `ErrInvalidScheduleTime` | `scheduler.IsInvalidScheduleTime(err)` | `ScheduledTime` is not valid RFC3339 or is in the past |
| `ErrJobNotFound` | `scheduler.IsNotFound(err)` | `GetJob` or `DeleteJob` called for a job that does not exist |
| `ErrSchedulerTimeout` | `scheduler.IsTimeout(err)` | GCP API call exceeded the deadline. The caller should retry. |

```go
job, err := sched.GetJob(ctx, jobName)
switch {
case scheduler.IsNotFound(err):
    return nil // job already ran or was deleted, nothing to do
case scheduler.IsTimeout(err):
    return fmt.Errorf("scheduler API timed out, will retry: %w", err)
case err != nil:
    return fmt.Errorf("unexpected scheduler error: %w", err)
}
```

---

## Testing With the Mock Client

The library ships a `mock` implementation of the `SchedulerClient` interface that stores jobs in memory. Use it in unit tests to assert that scheduling calls are made with the correct parameters without needing GCP credentials:

```go
import "github.com/Response-To-City-Disaster/beacon-scheduler/mock"

func TestWellbeingCheckIsScheduled(t *testing.T) {
    mockSched := mock.NewClient()

    svc := wellbeing.NewService(mockSched)
    err := svc.ScheduleFollowUp(ctx, "citizen-001", time.Now().Add(24*time.Hour))
    require.NoError(t, err)

    jobs := mockSched.ListScheduledJobs()
    require.Len(t, jobs, 1)
    assert.Equal(t, "beacon-notifications", jobs[0].TargetTopic)

    var payload WellbeingPayload
    json.Unmarshal(jobs[0].Payload, &payload)
    assert.Equal(t, "citizen-001", payload.CitizenID)
}
```

The mock also supports simulating errors:

```go
mockSched := mock.NewClient(mock.WithScheduleError(scheduler.ErrSchedulerTimeout))
```

---

## How beacon-notification Uses This Library

`beacon-notification` uses `beacon-scheduler` to defer time-sensitive notifications. For example:

- **Wellness follow-ups**: After a citizen reports an SOS incident, a wellness check message is scheduled for 24 hours later via `Schedule()`. If the incident is resolved earlier, the scheduled job is cancelled via `DeleteJob()`.
- **Batch notifications**: When a bulk broadcast is created (e.g. evacuating an entire district), the delivery can be staggered by scheduling multiple Pub/Sub messages at slightly different times to avoid overwhelming FCM.
- **Recurring reports**: Scheduled summaries for ERT commanders are triggered by recurring Cloud Scheduler jobs managed through this library.

---

## License

MIT — see [LICENSE](LICENSE) for details.
