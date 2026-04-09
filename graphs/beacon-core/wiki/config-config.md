# config-config

## Overview

File-based community: /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go

- **Size**: 17 nodes
- **Cohesion**: 0.5161
- **Dominant Language**: go

## Members

| Name | Kind | File | Lines |
|------|------|------|-------|
| Config | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 12-26 |
| EventBusConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 28-32 |
| ServerConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 35-38 |
| DatabaseConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 41-44 |
| DBConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 47-55 |
| LoggingConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 58-61 |
| NotificationConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 64-66 |
| IAMConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 69-73 |
| ObservabilityConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 76-79 |
| DispatchConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 82-84 |
| TomTomConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 87-90 |
| ClickHouseConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 93-99 |
| AuditingConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 102-109 |
| RedisConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 112-118 |
| StorageConfig | Class | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 121-125 |
| Load | Function | /home/neiltaurogemini/ase-clean/beacon-core/pkg/config/config.go | 128-154 |

## Execution Flows

No execution flows pass through this community.

## Dependencies

### Outgoing

- `Errorf` (2 edge(s))
- `fmt` (1 edge(s))
- `strings` (1 edge(s))
- `time` (1 edge(s))
- `github.com/spf13/viper` (1 edge(s))
- `New` (1 edge(s))
- `SetConfigFile` (1 edge(s))
- `SetConfigType` (1 edge(s))
- `ReadInConfig` (1 edge(s))
- `SetEnvPrefix` (1 edge(s))
- `SetEnvKeyReplacer` (1 edge(s))
- `NewReplacer` (1 edge(s))
- `AutomaticEnv` (1 edge(s))
- `Unmarshal` (1 edge(s))
