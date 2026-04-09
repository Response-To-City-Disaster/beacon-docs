# dispatch-event

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go

- **Size**: 14 nodes
- **Cohesion**: 0.2241
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| IAMClient | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 21-24 |
| EventPublisher | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 27-32 |
| SSEBroker | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 35-37 |
| IncidentEventReporter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 40-42 |
| IncidentReader | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 45-47 |
| ObservabilityClient | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 50-52 |
| Service | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 55-66 |
| NewService | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 79-99 |
| stationCandidate | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 102-105 |
| availResult | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 157-161 |
| rankedStation | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 208-212 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 461-510 |
| collectFCMTokens | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/dispatch/service.go | 796-804 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Msg` (5 edge(s))
- `Str` (5 edge(s))
- `Err` (3 edge(s))
- `Info` (2 edge(s))
- `Error` (2 edge(s))
- `GetByID` (2 edge(s))
- `Background` (2 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `math` (1 edge(s))
- `sort` (1 edge(s))
- `strconv` (1 edge(s))
- `strings` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-auditing` (1 edge(s))
