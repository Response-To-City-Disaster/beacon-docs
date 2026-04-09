# clickhouse-connection

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/clickhouse/connection.go

- **Size**: 5 nodes
- **Cohesion**: 0.1923
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| ConnectionConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/clickhouse/connection.go | 12-23 |
| Connection | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/clickhouse/connection.go | 25-27 |
| NewConnection | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/clickhouse/connection.go | 29-70 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/clickhouse/connection.go | 81-86 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Errorf` (4 edge(s))
- `WithTimeout` (2 edge(s))
- `Background` (2 edge(s))
- `cancel` (2 edge(s))
- `Ping` (2 edge(s))
- `Close` (2 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `time` (1 edge(s))
- `github.com/ClickHouse/clickhouse-go/v2` (1 edge(s))
- `github.com/ClickHouse/clickhouse-go/v2/lib/driver` (1 edge(s))
- `Sprintf` (1 edge(s))
- `Open` (1 edge(s))
