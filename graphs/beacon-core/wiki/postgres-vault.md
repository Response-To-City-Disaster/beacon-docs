# postgres-vault

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go

- **Size**: 9 nodes
- **Cohesion**: 0.2897
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| VaultRepository | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 15-17 |
| NewVaultRepository | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 20-22 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 997-1008 |
| scanFileRow | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 687-728 |
| marshalMetadata | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 731-740 |
| unmarshalMetadata | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 743-752 |
| nullableString | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 755-760 |
| scanFolderRowWithStats | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/db/postgres/vault_repo.go | 765-801 |

## Execution Flows

- **error** (criticality: 0.36, depth: 1)
- **scanFileRow** (criticality: 0.28, depth: 1)

## Dependencies

### Outgoing

- `DatabaseError` (22 edge(s))
- `ExecContext` (17 edge(s))
- `RowsAffected` (8 edge(s))
- `NotFound` (8 edge(s))
- `Sprintf` (8 edge(s))
- `len` (2 edge(s))
- `Scan` (2 edge(s))
- `context` (1 edge(s))
- `database/sql` (1 edge(s))
- `encoding/json` (1 edge(s))
- `fmt` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/internal/domain/vault` (1 edge(s))
- `github.com/Response-To-City-Disaster/beacon-core/pkg/errors` (1 edge(s))
- `github.com/google/uuid` (1 edge(s))
- `Marshal` (1 edge(s))
