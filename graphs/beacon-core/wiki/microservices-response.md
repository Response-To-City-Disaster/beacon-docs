# microservices-response

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go

- **Size**: 11 nodes
- **Cohesion**: 0.4000
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| IAMClient | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 16-24 |
| NewIAMClient | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 30-39 |
| Device | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 42-58 |
| DevicesResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 61-65 |
| loginResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 67-72 |
| string | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 78-103 |
| GroundStaffMember | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 181-189 |
| groundStaffListResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 191-194 |
| bulkStaffCountsRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 236-239 |
| bulkStaffCountsResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/microservices/iam_client.go | 242-245 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `bytes` (1 edge(s))
- `context` (1 edge(s))
- `encoding/json` (1 edge(s))
- `fmt` (1 edge(s))
- `net/http` (1 edge(s))
- `sync` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `Lock` (1 edge(s))
- `Unlock` (1 edge(s))
- `Before` (1 edge(s))
- `Add` (1 edge(s))
- `Now` (1 edge(s))
- `login` (1 edge(s))
- `GetUserToken` (1 edge(s))
