# clickhouse-location

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/clickhouse/location_repo.go

- **Size**: 4 nodes
- **Cohesion**: 0.2500
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| LocationRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/clickhouse/location_repo.go | 11-13 |
| NewLocationRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/clickhouse/location_repo.go | 15-17 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/clickhouse/location_repo.go | 19-37 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `context` (1 edge(s))
- `github.com/ClickHouse/clickhouse-go/v2/lib/driver` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/location` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/clickhouse` (1 edge(s))
- `len` (1 edge(s))
- `PrepareBatch` (1 edge(s))
- `Append` (1 edge(s))
- `Abort` (1 edge(s))
- `Send` (1 edge(s))
