# handlers-location

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go

- **Size**: 8 nodes
- **Cohesion**: 0.4118
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| LocationHandler | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 18-21 |
| NewLocationHandler | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 24-26 |
| locationUpdateRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 29-34 |
| batchIngestRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 37-39 |
| trackPoint | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 42-47 |
| userTrack | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 50-53 |
| trackResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/location_handler.go | 56-58 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `encoding/json` (1 edge(s))
- `net/http` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/location` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/response` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/tracing` (1 edge(s))
- `github.com/gorilla/mux` (1 edge(s))
