# evacuation-nullable

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go

- **Size**: 8 nodes
- **Cohesion**: 0.2907
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| PostgresMasterRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 13-15 |
| NewPostgresMasterRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 17-19 |
| nullableFloat | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 21-26 |
| nullableString | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 28-33 |
| nullableAdvisoryType | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 35-40 |
| nullableTransportMode | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 42-47 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/database/evacuation/postgres_master.go | 124-138 |

## Execution Flows

- **error** (criticality: 0.36, depth: 1)

## Dependencies

### Outgoing

- `Msg` (11 edge(s))
- `Str` (11 edge(s))
- `Info` (6 edge(s))
- `ExecContext` (3 edge(s))
- `Err` (3 edge(s))
- `Error` (3 edge(s))
- `DatabaseError` (3 edge(s))
- `string` (3 edge(s))
- `FromEntity` (2 edge(s))
- `RowsAffected` (2 edge(s))
- `Warn` (2 edge(s))
- `NotFound` (2 edge(s))
- `Sprintf` (2 edge(s))
- `Int64` (2 edge(s))
- `context` (1 edge(s))
