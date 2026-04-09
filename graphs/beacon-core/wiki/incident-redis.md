# incident-redis

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/incident/redis_cache.go

- **Size**: 4 nodes
- **Cohesion**: 0.2667
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| RedisCacheRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/incident/redis_cache.go | 13-15 |
| NewRedisCacheRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/incident/redis_cache.go | 18-22 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/incident/redis_cache.go | 58-61 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Sprintf` (2 edge(s))
- `context` (1 edge(s))
- `encoding/json` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/redis` (1 edge(s))
- `Marshal` (1 edge(s))
- `Errorf` (1 edge(s))
- `Set` (1 edge(s))
- `Delete` (1 edge(s))
