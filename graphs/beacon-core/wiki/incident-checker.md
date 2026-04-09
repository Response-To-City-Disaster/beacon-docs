# incident-checker

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go

- **Size**: 11 nodes
- **Cohesion**: 0.1864
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| DispatchChecker | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 16-18 |
| EvacuationChecker | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 21-23 |
| EventPublisher | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 26-32 |
| RecipientResolver | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 35-37 |
| CacheRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 40-44 |
| Service | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 47-58 |
| NewService | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 76-94 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 746-766 |
| computeChanges | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 723-744 |
| FormatLocation | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/incident/service.go | 769-771 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Msg` (6 edge(s))
- `Str` (6 edge(s))
- `ValidationFailed` (6 edge(s))
- `string` (3 edge(s))
- `Info` (3 edge(s))
- `Error` (3 edge(s))
- `BadRequest` (2 edge(s))
- `Err` (2 edge(s))
- `InternalError` (2 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-auditing` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
