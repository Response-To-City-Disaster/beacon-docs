# dispatch-postgres

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/dispatch/postgres_slave.go

- **Size**: 4 nodes
- **Cohesion**: 0.2727
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| PostgresSlaveRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/dispatch/postgres_slave.go | 15-17 |
| NewPostgresSlaveRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/dispatch/postgres_slave.go | 20-22 |
| scanDispatch | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/dispatch/postgres_slave.go | 28-37 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `context` (1 edge(s))
- `database/sql` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/dispatch` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/lib/pq` (1 edge(s))
- `github.com/rs/zerolog/log` (1 edge(s))
- `Scan` (1 edge(s))
