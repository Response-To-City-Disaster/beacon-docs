# gcs-storage

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go

- **Size**: 6 nodes
- **Cohesion**: 0.1818
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| StorageConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go | 15-19 |
| StorageProvider | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go | 22-27 |
| GCSAdapter | Class | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go | 31-35 |
| NewGCSAdapter | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go | 39-63 |
| error | Function | /home/neiltaurogemini/ase-clean/beacon-core/internal/adapters/storage/gcs/gcs_client.go | 149-165 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Errorf` (9 edge(s))
- `Close` (3 edge(s))
- `append` (2 edge(s))
- `context` (1 edge(s))
- `fmt` (1 edge(s))
- `io` (1 edge(s))
- `time` (1 edge(s))
- `cloud.google.com/go/iam/credentials/apiv1` (1 edge(s))
- `cloud.google.com/go/iam/credentials/apiv1/credentialspb` (1 edge(s))
- `cloud.google.com/go/storage` (1 edge(s))
- `NewClient` (1 edge(s))
- `NewIamCredentialsClient` (1 edge(s))
- `Object` (1 edge(s))
- `Bucket` (1 edge(s))
- `Delete` (1 edge(s))
