# integration-dbconfig

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go

- **Size**: 6 nodes
- **Cohesion**: 0.0344
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| DBConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go | 16-22 |
| getDBConfig | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go | 25-40 |
| connectDB | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go | 43-78 |
| ensureIncidentsTable | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go | 81-128 |
| TestIncidentCRUD | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/integration/incident_crud_test.go | 131-580 |

## Execution Flows

- **TestIncidentCRUD** (criticality: 0.40, depth: 2)

## Dependencies

### Outgoing

- `Logf` (27 edge(s))
- `Errorf` (26 edge(s))
- `Run` (11 edge(s))
- `Scan` (11 edge(s))
- `Log` (10 edge(s))
- `Close` (8 edge(s))
- `ExecContext` (6 edge(s))
- `QueryContext` (6 edge(s))
- `Next` (6 edge(s))
- `QueryRowContext` (5 edge(s))
- `getEnv` (5 edge(s))
- `Now` (4 edge(s))
- `Background` (3 edge(s))
- `RowsAffected` (3 edge(s))
- `string` (2 edge(s))

### Incoming

- `Errorf` (26 edge(s))
- `Logf` (24 edge(s))
- `Run` (11 edge(s))
- `Scan` (11 edge(s))
- `Log` (10 edge(s))
- `Close` (7 edge(s))
- `QueryContext` (6 edge(s))
- `Next` (6 edge(s))
- `QueryRowContext` (5 edge(s))
- `Now` (4 edge(s))
- `ExecContext` (4 edge(s))
- `RowsAffected` (3 edge(s))
- `string` (2 edge(s))
- `make` (2 edge(s))
- `Background` (1 edge(s))
