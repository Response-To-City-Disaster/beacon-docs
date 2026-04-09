# unit-incident

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go

- **Size**: 22 nodes
- **Cohesion**: 0.5000
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| MockEventPublisher | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 14-19 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 122-125 |
| MockMasterRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 48-50 |
| NewMockMasterRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 52-56 |
| MockSlaveRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 74-76 |
| NewMockSlaveRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 78-82 |
| MockEventMasterRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 112-114 |
| NewMockEventMasterRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 116-120 |
| MockEventSlaveRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 128-130 |
| NewMockEventSlaveRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 132-136 |
| newTestService | Function | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 151-153 |
| TestRecordIncidentUpdate_Success | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 156-192 |
| TestRecordIncidentUpdate_EmptyID | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 195-214 |
| TestRecordIncidentUpdate_IncidentNotFound | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 217-236 |
| TestRecordIncidentUpdate_EmptyMessage | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 239-266 |
| TestRecordIncidentUpdate_AlertDisabled | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 269-295 |
| TestRecordIncidentUpdate_WithUpdateMessage | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 298-330 |
| TestRecordIncidentUpdateHandler_Success | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 333-374 |
| TestRecordIncidentUpdateHandler_EmptyID | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 377-398 |
| TestRecordIncidentUpdateHandler_EmptyMessage | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 401-430 |
| TestRecordIncidentUpdateResult_Structure | Test | /home/neiltaurogemini/ase-clean/beacon-core/tests/unit/record_incident_update_test.go | 433-451 |

## Execution Flows

- **TestRecordIncidentUpdate_Success** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdate_EmptyID** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdate_IncidentNotFound** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdate_EmptyMessage** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdate_AlertDisabled** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdate_WithUpdateMessage** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdateHandler_Success** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdateHandler_EmptyID** (criticality: 0.34, depth: 1)
- **TestRecordIncidentUpdateHandler_EmptyMessage** (criticality: 0.34, depth: 1)

## Dependencies

### Outgoing

- `Errorf` (12 edge(s))
- `Background` (9 edge(s))
- `Error` (8 edge(s))
- `AddIncident` (6 edge(s))
- `RecordIncidentUpdate` (6 edge(s))
- `make` (4 edge(s))
- `NewRecordIncidentUpdateHandler` (3 edge(s))
- `Handle` (3 edge(s))
- `GetAppError` (3 edge(s))
- `Fatalf` (2 edge(s))
- `context` (1 edge(s))
- `testing` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/commands/incident` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))

### Incoming

- `Errorf` (12 edge(s))
- `Background` (9 edge(s))
- `Error` (8 edge(s))
- `AddIncident` (6 edge(s))
- `RecordIncidentUpdate` (6 edge(s))
- `NewRecordIncidentUpdateHandler` (3 edge(s))
- `Handle` (3 edge(s))
- `GetAppError` (3 edge(s))
- `Fatalf` (2 edge(s))
- `Fatal` (1 edge(s))
