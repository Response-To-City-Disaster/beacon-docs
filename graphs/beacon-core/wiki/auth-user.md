# auth-user

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go

- **Size**: 20 nodes
- **Cohesion**: 0.1879
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| contextKey | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 15-15 |
| AuthConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 44-48 |
| DefaultAuthConfig | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 51-57 |
| authoriseData | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 66-71 |
| AuthoriseResponse | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 74-78 |
| Middleware | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 81-83 |
| MiddlewareWithConfig | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 86-166 |
| isInternalK8sRequest | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 169-200 |
| validateTokenWithIAM | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 203-266 |
| RequireRole | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 269-297 |
| hasAnyRole | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 300-309 |
| GetUserRoles | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 312-318 |
| HasRole | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 321-329 |
| IsAdmin | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 332-334 |
| GetUserToken | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 337-343 |
| GetUserID | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 346-352 |
| GetUserIDOrDefault | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 355-361 |
| GetUserName | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 364-367 |
| GetUserPrimaryRole | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/auth/auth.go | 370-384 |

## Execution Flows

- **Middleware** (criticality: 0.62, depth: 2)
- **RequireRole** (criticality: 0.61, depth: 1)
- **GetUserPrimaryRole** (criticality: 0.53, depth: 1)
- **IsAdmin** (criticality: 0.50, depth: 2)
- **GetUserIDOrDefault** (criticality: 0.49, depth: 1)

## Dependencies

### Outgoing

- `Msg` (15 edge(s))
- `Str` (15 edge(s))
- `Warn` (8 edge(s))
- `WithValue` (7 edge(s))
- `Error` (7 edge(s))
- `InternalError` (6 edge(s))
- `Value` (5 edge(s))
- `ServeHTTP` (5 edge(s))
- `len` (4 edge(s))
- `Context` (4 edge(s))
- `Err` (4 edge(s))
- `EqualFold` (3 edge(s))
- `Unauthorized` (3 edge(s))
- `HandlerFunc` (2 edge(s))
- `Debug` (2 edge(s))
