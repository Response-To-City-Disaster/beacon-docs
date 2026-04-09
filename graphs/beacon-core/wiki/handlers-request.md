# handlers-request

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go

- **Size**: 16 nodes
- **Cohesion**: 0.5769
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| VaultHandler | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 19-22 |
| NewVaultHandler | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 25-30 |
| CreateFolderRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 33-36 |
| InitiateUploadRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 39-44 |
| attachmentResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 222-228 |
| fileDTO | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 231-247 |
| AttachFileToIncidentRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 312-314 |
| fileResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 389-404 |
| UpdateFileRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 428-430 |
| AddTagRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 472-475 |
| MoveFileRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 651-653 |
| BulkDeleteRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 753-755 |
| BulkMoveRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 757-760 |
| BulkStarRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 762-764 |
| BulkRestoreRequest | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/http/handlers/vault_handler.go | 766-768 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `encoding/json` (1 edge(s))
- `net/http` (1 edge(s))
- `time` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/vault` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/registry/vault` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/auth` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/logger` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/response` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/tracing` (1 edge(s))
- `github.com/gorilla/mux` (1 edge(s))
