# response-response

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go

- **Size**: 11 nodes
- **Cohesion**: 0.2927
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| Response | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 11-16 |
| ErrorInfo | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 19-22 |
| PaginatedResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 25-31 |
| Pagination | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 34-39 |
| JSON | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 42-54 |
| Error | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 57-77 |
| Paginated | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 80-93 |
| Success | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 96-98 |
| Created | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 101-103 |
| NoContent | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/response/response.go | 106-109 |

## Execution Flows

- **Success** (criticality: 0.36, depth: 1)
- **Created** (criticality: 0.36, depth: 1)

## Dependencies

### Outgoing

- `Set` (7 edge(s))
- `Header` (7 edge(s))
- `WriteHeader` (4 edge(s))
- `Encode` (3 edge(s))
- `NewEncoder` (3 edge(s))
- `encoding/json` (1 edge(s))
- `net/http` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `GetAppError` (1 edge(s))
- `InternalError` (1 edge(s))
