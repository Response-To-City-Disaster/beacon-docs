# handlers-dispatch

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/dispatch_handler.go

- **Size**: 5 nodes
- **Cohesion**: 0.2500
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| SSESubscriber | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/dispatch_handler.go | 20-22 |
| DispatchHandler | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/dispatch_handler.go | 25-38 |
| NewDispatchHandler | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/dispatch_handler.go | 41-69 |
| CreateDispatchRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/dispatch_handler.go | 72-78 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `encoding/json` (1 edge(s))
- `fmt` (1 edge(s))
- `net/http` (1 edge(s))
- `strconv` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/dispatch` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/commands/dispatch` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/queries/dispatch` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/response` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/tracing` (1 edge(s))
- `github.com/gorilla/mux` (1 edge(s))
