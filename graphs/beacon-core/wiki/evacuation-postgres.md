# evacuation-postgres

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_slave.go

- **Size**: 4 nodes
- **Cohesion**: 0.3000
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| PostgresSlaveRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_slave.go | 13-15 |
| NewPostgresSlaveRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_slave.go | 17-19 |
| scanEvacuation | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_slave.go | 30-44 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `context` (1 edge(s))
- `database/sql` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/evacuation` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/rs/zerolog/log` (1 edge(s))
- `Scan` (1 edge(s))
