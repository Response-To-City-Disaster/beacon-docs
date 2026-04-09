# handlers-incident

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go

- **Size**: 10 nodes
- **Cohesion**: 0.3600
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| IncidentHandler | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 22-35 |
| NewIncidentHandler | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 38-66 |
| CreateIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 69-78 |
| UpdateIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 81-90 |
| EscalateIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 93-96 |
| CloseIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 99-101 |
| ApproveIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 104-106 |
| RecordIncidentUpdateRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 109-113 |
| notFoundError | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/incident_handler.go | 709-711 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `encoding/json` (1 edge(s))
- `fmt` (1 edge(s))
- `net/http` (1 edge(s))
- `strconv` (1 edge(s))
- `strings` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/commands/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/queries/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/response` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/tracing` (1 edge(s))
- `github.com/gorilla/mux` (1 edge(s))
- `NotFound` (1 edge(s))
