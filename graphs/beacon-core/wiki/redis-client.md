# redis-client

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/redis/connection.go

- **Size**: 4 nodes
- **Cohesion**: 0.2609
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| Client | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/redis/connection.go | 12-15 |
| NewClient | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/redis/connection.go | 18-37 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/redis/connection.go | 59-61 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Err` (3 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `time` (1 edge(s))
- `github.com/redis/go-redis/v9` (1 edge(s))
- `Sprintf` (1 edge(s))
- `WithTimeout` (1 edge(s))
- `Background` (1 edge(s))
- `cancel` (1 edge(s))
- `Ping` (1 edge(s))
- `Errorf` (1 edge(s))
- `Duration` (1 edge(s))
- `Set` (1 edge(s))
- `Del` (1 edge(s))
- `Close` (1 edge(s))
