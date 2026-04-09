# evacuation-incident

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go

- **Size**: 9 nodes
- **Cohesion**: 0.1525
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| IncidentEventReporter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 19-21 |
| IncidentReader | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 24-26 |
| IncidentTypeReader | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 29-31 |
| EventPublisher | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 34-38 |
| RecipientResolver | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 41-43 |
| Service | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 46-55 |
| NewService | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 73-87 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/domain/evacuation/service.go | 402-445 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Msg` (8 edge(s))
- `Str` (8 edge(s))
- `Info` (5 edge(s))
- `Err` (3 edge(s))
- `Error` (3 edge(s))
- `ValidationFailed` (2 edge(s))
- `GetUserIDOrDefault` (2 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `strconv` (1 edge(s))
- `strings` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-auditing` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
