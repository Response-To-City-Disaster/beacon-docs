# http-middleware

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/middleware.go

- **Size**: 4 nodes
- **Cohesion**: 0.1071
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| LoggingMiddleware | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/middleware.go | 11-34 |
| CORSMiddleware | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/middleware.go | 37-50 |
| AuthenticationMiddleware | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/middleware.go | 53-61 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `HandlerFunc` (3 edge(s))
- `ServeHTTP` (3 edge(s))
- `Set` (3 edge(s))
- `Header` (3 edge(s))
- `Msg` (2 edge(s))
- `Str` (2 edge(s))
- `Info` (2 edge(s))
- `net/http` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `WriteHeader` (1 edge(s))
- `Now` (1 edge(s))
- `Dur` (1 edge(s))
- `Since` (1 edge(s))
