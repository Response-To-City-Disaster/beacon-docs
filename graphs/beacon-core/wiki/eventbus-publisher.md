# eventbus-publisher

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/publisher.go

- **Size**: 5 nodes
- **Cohesion**: 0.1136
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| NotificationEvent | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/publisher.go | 28-43 |
| Publisher | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/publisher.go | 46-49 |
| NewPublisher | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/publisher.go | 52-68 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/publisher.go | 80-117 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Msg` (4 edge(s))
- `Str` (4 edge(s))
- `Errorf` (3 edge(s))
- `Err` (3 edge(s))
- `Error` (3 edge(s))
- `Sprintf` (2 edge(s))
- `Unwrap` (2 edge(s))
- `context` (1 edge(s))
- `encoding/json` (1 edge(s))
- `errors` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/config` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-event-bus` (1 edge(s))
- `github.com/rs/zerolog` (1 edge(s))
- `WithProjectID` (1 edge(s))
