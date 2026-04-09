# handlers-evacuation

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/evacuation_handler.go

- **Size**: 5 nodes
- **Cohesion**: 0.2857
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| EvacuationHandler | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/evacuation_handler.go | 18-28 |
| NewEvacuationHandler | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/evacuation_handler.go | 31-53 |
| CreateEvacuationRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/evacuation_handler.go | 56-77 |
| UpdateEvacuationRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/evacuation_handler.go | 80-100 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `encoding/json` (1 edge(s))
- `net/http` (1 edge(s))
- `strconv` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/evacuation` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/commands/evacuation` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/queries/evacuation` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/response` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/tracing` (1 edge(s))
- `github.com/gorilla/mux` (1 edge(s))
