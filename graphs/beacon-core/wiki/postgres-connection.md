# postgres-connection

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go

- **Size**: 6 nodes
- **Cohesion**: 0.2593
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| Config | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go | 12-20 |
| ConnectionPools | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go | 23-26 |
| NewConnectionPools | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go | 29-45 |
| connect | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go | 48-71 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/postgres/connection.go | 74-94 |

## Execution Flows

- **NewConnectionPools** (criticality: 0.61, depth: 1)

## Dependencies

### Outgoing

- `Errorf` (6 edge(s))
- `Close` (4 edge(s))
- `database/sql` (1 edge(s))
- `fmt` (1 edge(s))
- `time` (1 edge(s))
- `github.com/lib/pq` (1 edge(s))
- `Sprintf` (1 edge(s))
- `Open` (1 edge(s))
- `SetMaxOpenConns` (1 edge(s))
- `SetMaxIdleConns` (1 edge(s))
- `SetConnMaxLifetime` (1 edge(s))
- `Ping` (1 edge(s))
