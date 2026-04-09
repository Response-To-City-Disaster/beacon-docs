# server-adapter

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go

- **Size**: 8 nodes
- **Cohesion**: 0.0327
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| incidentReporterAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 61-63 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 65-90 |
| iamRecipientResolver | Class | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 94-96 |
| incidentApprovalAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 104-106 |
| dispatchActivityAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 126-128 |
| evacuationActivityAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 145-147 |
| main | Function | /home/neiltaurogemini/ase-clean/beacon-core/cmd/server/main.go | 162-513 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Msg` (22 edge(s))
- `Info` (13 edge(s))
- `Err` (9 edge(s))
- `Close` (6 edge(s))
- `Fatal` (6 edge(s))
- `NewService` (5 edge(s))
- `Background` (4 edge(s))
- `Error` (3 edge(s))
- `Str` (3 edge(s))
- `NewPostgresMasterRepository` (3 edge(s))
- `NewPostgresSlaveRepository` (3 edge(s))
- `SetAuditClient` (3 edge(s))
- `NewClient` (2 edge(s))
- `SetIncidentReader` (2 edge(s))
- `NewListByIncidentHandler` (2 edge(s))
