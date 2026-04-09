# eventbus-adapter

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go

- **Size**: 10 nodes
- **Cohesion**: 0.2258
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| IncidentPublisherAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 68-68 |
| NewIncidentPublisherAdapter | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 70-72 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 369-478 |
| DispatchPublisherAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 189-189 |
| NewDispatchPublisherAdapter | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 191-193 |
| EvacuationPublisherAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 283-283 |
| NewEvacuationPublisherAdapter | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 285-287 |
| EvacuationAdvisoryAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 354-359 |
| NewEvacuationAdvisoryAdapter | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/eventbus/adapters.go | 362-366 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Sprintf` (24 edge(s))
- `string` (24 edge(s))
- `PublishNotification` (13 edge(s))
- `FormatLocation` (5 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/dispatch` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/evacuation` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/incident` (1 edge(s))
- `GetActiveFCMTokens` (1 edge(s))
