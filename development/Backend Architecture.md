# Backend Architecture

> This guide will help you understand the Beacon microservice architecture, patterns used, and how to contribute effectively to the codebase. Whether you're new to Go or backend development, this guide has you covered.

---

## Introduction to Go (Golang)

### What is Go?

Go (or Golang) is a **statically typed, compiled programming language** designed at Google. It's known for:

- ⚡ **Speed**: Compiles directly to machine code
- 🔄 **Concurrency**: Built-in support for goroutines (lightweight threads)
- 📦 **Simplicity**: Clean syntax, no complex features like inheritance
- 🛠️ **Standard Library**: Rich built-in packages for web, networking, etc.

### Why Go for Microservices?

```
┌──────────────────────────────────────────────────────┐
│              Why Go is Perfect for Microservices      │
├──────────────────────────────────────────────────────┤
│ ✓ Fast startup time (important for container scaling) │
│ ✓ Low memory footprint                                │
│ ✓ Built-in HTTP server                                │
│ ✓ Easy to build and deploy (single binary)            │
│ ✓ Strong typing catches errors at compile time        │
└──────────────────────────────────────────────────────┘
```

### Basic Go Syntax Quick Reference

```go
// Package declaration - every Go file belongs to a package
package main

// Import dependencies
import (
    "fmt"          // Standard library for formatting
    "net/http"     // Standard library for HTTP
)

// Main function - entry point of the application
func main() {
    fmt.Println("Hello, Beacon!")
}

// Function with parameters and return type
func add(a int, b int) int {
    return a + b
}

// Struct - Go's way of defining custom types (like classes in other languages)
type Incident struct {
    ID          string
    Title       string
    Severity    string
}

// Method on a struct (receiver function)
func (i *Incident) CanClose() bool {
    return i.Severity != "critical"
}

// Interface - defines a contract that types must implement
type Repository interface {
    Create(incident *Incident) error
    GetByID(id string) (*Incident, error)
}
```

> 💡 **Key Insight**: Unlike Java/C#, Go uses **composition over inheritance**. There are no classes, only structs with methods.

<!-- 🖼️ IMAGE PLACEHOLDER: Go syntax diagram showing package, imports, functions, structs -->

---

## Understanding the Architecture

### High-Level Overview

Beacon Core follows a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          🌐 HTTP LAYER (API)                                 │
│     Receives HTTP requests, validates input, returns HTTP responses          │
│     Files: internal/http/router.go, internal/http/handlers/                  │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       📋 REGISTRY LAYER (CQRS)                               │
│     Separates read operations (Queries) from write operations (Commands)     │
│     Files: internal/registry/commands/, internal/registry/queries/           │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        💼 DOMAIN LAYER (Core)                                │
│     Contains business logic, domain models, and repository interfaces        │
│     Files: internal/domain/                                                  │
│     ⚠️ THIS LAYER HAS ZERO EXTERNAL DEPENDENCIES!                            │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       🔌 ADAPTERS LAYER (Integration)                        │
│     Implements the interfaces defined in domain layer                        │
│     Files: internal/adapters/database/, internal/adapters/microservices/     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Happens When You Make an API Request?

Let's trace a request to create an incident step by step:

```
1️⃣ HTTP Request arrives at POST /v1/incidents
                    │
                    ▼
2️⃣ Router (router.go) matches the route and calls IncidentHandler
                    │
                    ▼
3️⃣ IncidentHandler.CreateIncident() receives the request
   - Parses JSON body
   - Creates a CreateIncidentCommand
                    │
                    ▼
4️⃣ CreateIncidentHandler.Handle() is called
   - This is the CQRS Command Handler
   - Converts command to domain entity
                    │
                    ▼
5️⃣ Service.CreateIncident() in domain layer is called
   - Validates business rules
   - Generates ID, sets timestamps
                    │
                    ▼
6️⃣ MasterRepository.Create() is called
   - This is an interface defined in domain layer
   - Actual implementation is in adapters layer (PostgresMasterRepository)
                    │
                    ▼
7️⃣ PostgresMasterRepository.Create() inserts into database
   - SQL INSERT statement is executed
                    │
                    ▼
8️⃣ Response travels back up the chain
   - Incident entity → HTTP Response → Client
```

<!-- 🖼️ IMAGE PLACEHOLDER: Request flow diagram with all layers -->

---

## Hexagonal Architecture (Ports & Adapters)

### What is Hexagonal Architecture?

Hexagonal Architecture (also called **Ports & Adapters**) is a design pattern that makes your application:

- ✅ **Testable** - Easy to swap real databases with mocks
- ✅ **Maintainable** - Changes in one area don't affect others
- ✅ **Flexible** - Easy to switch databases, APIs, etc.

### The Core Concept

```
                    ┌─────────────────────────┐
                    │                         │
        ┌──────────►│    DOMAIN (Core)        │◄──────────┐
        │           │    Business Logic       │           │
        │           │    No Dependencies!     │           │
        │           └─────────────────────────┘           │
        │                    │    │                       │
   ╔════╧════╗          ╔════╧════╧════╗           ╔════╧════╗
   ║  PORT   ║          ║    PORT      ║           ║  PORT   ║
   ║  (HTTP) ║          ║ (Repository) ║           ║ (Cache) ║
   ╚════╤════╝          ╚════╤════╤════╝           ╚════╤════╝
        │                    │    │                     │
   ┌────┴────┐          ┌────┴────┼─────┐          ┌────┴────┐
   │ ADAPTER │          │ ADAPTER │     │          │ ADAPTER │
   │ HTTP    │          │PostgreSQL│    │          │  Redis  │
   │ Handler │          │ Master   │    │          │  Cache  │
   └─────────┘          └──────────┘    │          └─────────┘
                                   ┌────┴────┐
                                   │ ADAPTER │
                                   │PostgreSQL│
                                   │  Slave   │
                                   └──────────┘
```

### Understanding Ports

**Ports are interfaces** that define how the outside world can interact with your domain:

```go
// This is a PORT - defined in the DOMAIN layer
// File: internal/domain/incident/repo.go

// MasterRepository defines write operations (PORT)
type MasterRepository interface {
    Create(ctx context.Context, incident *Incident) error
    Update(ctx context.Context, incident *Incident) error
    Delete(ctx context.Context, id string) error
}

// SlaveRepository defines read operations (PORT)
type SlaveRepository interface {
    GetByID(ctx context.Context, id string) (*Incident, error)
    List(ctx context.Context, opts ListOptions) (*ListResult, error)
    GetMetrics(ctx context.Context) (*IncidentMetrics, error)
}
```

> 💡 **Why Interfaces?** The domain layer only knows about the interface, not the implementation. This means you can swap PostgreSQL for MongoDB without changing domain code!

### Understanding Adapters

**Adapters implement the interfaces** defined in the domain layer:

```go
// This is an ADAPTER - implements the MasterRepository interface
// File: internal/adapters/database/incident/postgres_master.go

type PostgresMasterRepository struct {
    db *sql.DB  // Actual database connection
}

func NewPostgresMasterRepository(db *sql.DB) *PostgresMasterRepository {
    return &PostgresMasterRepository{db: db}
}

// Implements MasterRepository.Create
func (r *PostgresMasterRepository) Create(ctx context.Context, inc *incident.Incident) error {
    query := `INSERT INTO incidents (...) VALUES (...)`
    _, err := r.db.ExecContext(ctx, query, ...)
    return err
}
```

### Why This Matters for Testing

With Hexagonal Architecture, you can easily create **mock implementations** for testing:

```go
// Mock adapter for testing - no real database needed!
type MockMasterRepository struct {
    incidents map[string]*incident.Incident
}

func (m *MockMasterRepository) Create(ctx context.Context, inc *incident.Incident) error {
    m.incidents[inc.ID] = inc  // Just store in memory
    return nil
}
```

<!-- 🖼️ IMAGE PLACEHOLDER: Hexagonal architecture diagram with ports and adapters -->

---

## CQRS Pattern (Command Query Responsibility Segregation)

### What is CQRS?

CQRS is a pattern that **separates read operations from write operations**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CQRS Pattern                              │
├─────────────────────────────┬───────────────────────────────────┤
│        COMMANDS (Write)     │         QUERIES (Read)            │
├─────────────────────────────┼───────────────────────────────────┤
│ • CreateIncident            │ • GetIncident                     │
│ • UpdateIncident            │ • ListIncidents                   │
│ • CloseIncident             │ • GetIncidentMetrics              │
│ • EscalateIncident          │                                   │
│ • AlertCitizens             │                                   │
├─────────────────────────────┼───────────────────────────────────┤
│ Uses: Master Database       │ Uses: Slave Database              │
│ (Optimized for writes)      │ (Optimized for reads)             │
└─────────────────────────────┴───────────────────────────────────┘
```

### Why Separate Reads and Writes?

1. **Performance**: Read-heavy applications can scale read replicas independently
2. **Optimization**: Write models can be normalized, read models can be denormalized
3. **Security**: Different permissions for read vs write operations
4. **Clarity**: Code is easier to understand when separated by intent

### How CQRS is Implemented in Beacon Core

#### Commands (Write Operations)

```go
// File: internal/registry/commands/incident/create_incident.go

// Command - describes WHAT we want to do
type CreateIncidentCommand struct {
    Type           incident.IncidentType
    Severity       incident.SeverityLevel
    Title          string
    Description    string
    Location       incident.Location
    AffectedRadius float64
    ReportedBy     string
}

// Handler - executes the command
type CreateIncidentHandler struct {
    service *incident.Service
}

func (h *CreateIncidentHandler) Handle(ctx context.Context, cmd CreateIncidentCommand) (*incident.Incident, error) {
    // Convert command to domain entity
    inc := &incident.Incident{
        Type:     cmd.Type,
        Severity: cmd.Severity,
        Title:    cmd.Title,
        // ... other fields
    }
    
    // Call domain service
    return h.service.CreateIncident(ctx, inc)
}
```

#### Queries (Read Operations)

```go
// File: internal/registry/queries/incident/get_incident.go

// Query - describes WHAT data we want
type GetIncidentQuery struct {
    ID string
}

// Handler - fetches the data
type GetIncidentHandler struct {
    service *incident.Service
}

func (h *GetIncidentHandler) Handle(ctx context.Context, query GetIncidentQuery) (*incident.Incident, error) {
    return h.service.GetIncident(ctx, query.ID)
}
```

### Database Separation

Notice how the service uses **different repositories** for read and write:

```go
// File: internal/domain/incident/service.go

type Service struct {
    masterRepo MasterRepository  // For WRITE operations
    slaveRepo  SlaveRepository   // For READ operations
    // ...
}

// Write operation uses master
func (s *Service) CreateIncident(ctx context.Context, incident *Incident) (*Incident, error) {
    err := s.masterRepo.Create(ctx, incident)  // ← Uses MASTER
    return incident, err
}

// Read operation uses slave
func (s *Service) GetIncident(ctx context.Context, id string) (*Incident, error) {
    return s.slaveRepo.GetByID(ctx, id)  // ← Uses SLAVE
}
```

<!-- 🖼️ IMAGE PLACEHOLDER: CQRS diagram showing command and query paths -->

---

## Adapter Pattern

### What is the Adapter Pattern?

The Adapter Pattern is like a **power adapter** when you travel - it converts one interface to another:

```
┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
│   Your Domain       │      │     Adapter         │      │   External System   │
│   (Needs data       │ ───► │   (Translates       │ ───► │   (PostgreSQL,      │
│    in format A)     │      │    A to B)          │      │    Redis, APIs)     │
└─────────────────────┘      └─────────────────────┘      └─────────────────────┘
```

### Types of Adapters in Beacon Core

#### 1. Database Adapters

```
internal/adapters/database/incident/
├── postgres_master.go    # Writes to PostgreSQL master
├── postgres_slave.go     # Reads from PostgreSQL replica
└── redis_cache.go        # Caching layer
```

#### 2. Microservice Adapters

```
internal/adapters/microservices/
├── notification_client.go  # Talks to Notification service
├── iam_client.go           # Talks to Identity service
├── map_client.go           # Talks to Map service
└── ...
```

### Real Example: Notification Adapter

```go
// File: internal/adapters/microservices/notification_client.go

// NotificationClient adapts external notification service to our domain
type NotificationClient struct {
    baseURL string
    client  *http.Client
}

func NewNotificationClient(baseURL string) *NotificationClient {
    return &NotificationClient{
        baseURL: baseURL,
        client:  &http.Client{Timeout: 10 * time.Second},
    }
}

// Implements the NotificationClient interface from domain layer
func (c *NotificationClient) SendIncidentNotification(ctx context.Context, title, description string) error {
    // Prepare request for external service
    payload := map[string]string{
        "title":   title,
        "message": description,
    }
    
    // Call external service
    resp, err := c.client.Post(c.baseURL + "/send", "application/json", ...)
    
    return err
}
```

> 💡 **Key Insight**: The domain layer doesn't know (or care) how notifications are sent. It just calls the interface. The adapter handles all the HTTP details!

<!-- 🖼️ IMAGE PLACEHOLDER: Adapter pattern diagram showing interface translation -->

---

## Microservices Architecture

### What are Microservices?

Microservices is an architectural style where an application is **composed of small, independent services**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Beacon Platform                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │   Beacon     │   │ Notification │   │     IAM      │   │     Map      │      │
│  │    Core      │   │   Service    │   │   Service    │   │   Service    │      │
│  │ (This repo!) │   │              │   │              │   │              │      │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘      │
│         │                  │                  │                  │               │
│         └──────────────────┼──────────────────┼──────────────────┘               │
│                            │                  │                                  │
│                      ┌─────┴──────────────────┴─────┐                           │
│                      │      Service Communication    │                           │
│                      │         (HTTP/gRPC)           │                           │
│                      └───────────────────────────────┘                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Beacon Core's Responsibilities

| Service | Responsibility | Communication |
|---------|---------------|---------------|
| **Beacon Core** | Incident management, dispatching, evacuation | REST API |
| **Notification** | Send alerts to citizens | Called by Beacon Core |
| **IAM** | Authentication & authorization | Token validation |
| **Map** | Geographic features | Location data |

### How Services Communicate

```go
// In main.go - services are configured and injected

// Initialize IAM client (for authentication)
iamClient := microservices.NewIAMClient(cfg.IAM.URL)

// Initialize notification client (uses IAM for auth)
notificationClient := microservices.NewNotificationClient(
    cfg.Notification.URL,
    iamClient,  // Dependency injection
    log,
)

// Services are injected into the domain layer
incidentService := incident.NewService(
    incidentMasterRepo,
    incidentSlaveRepo,
    notificationClient,  // Can send notifications
    incidentCacheRepo,
)
```

### Benefits of Microservices

| Benefit | Explanation |
|---------|------------|
| **Independent Deployment** | Update Beacon Core without touching Notification service |
| **Technology Freedom** | Each service can use different languages/databases |
| **Fault Isolation** | If Notification crashes, Beacon Core keeps running |
| **Scalability** | Scale busy services independently |
| **Team Ownership** | Different teams can own different services |

<!-- 🖼️ IMAGE PLACEHOLDER: Microservices communication diagram -->

---

## Project Structure Explained

```
beacon-core/
├── 📁 cmd/                          # Application entry points
│   └── 📁 server/
│       └── 📄 main.go               # 🚀 START HERE! Application bootstrap
│
├── 📁 internal/                     # Private application code
│   ├── 📁 adapters/                 # External integrations
│   │   ├── 📁 database/             # Database implementations
│   │   │   └── 📁 incident/
│   │   │       ├── 📄 postgres_master.go  # Write operations
│   │   │       ├── 📄 postgres_slave.go   # Read operations
│   │   │       └── 📄 redis_cache.go      # Caching
│   │   ├── 📁 microservices/        # External service clients
│   │   │   ├── 📄 notification_client.go
│   │   │   └── 📄 iam_client.go
│   │   └── 📁 external_api/         # Third-party APIs
│   │
│   ├── 📁 domain/                   # 💎 CORE BUSINESS LOGIC
│   │   └── 📁 incident/
│   │       ├── 📄 domain.go         # Entity definitions
│   │       ├── 📄 service.go        # Business operations
│   │       ├── 📄 repo.go           # Repository interfaces (PORTS)
│   │       ├── 📄 mapper.go         # Data transformation
│   │       └── 📁 chain/            # Chain of Responsibility pattern
│   │
│   ├── 📁 http/                     # HTTP layer
│   │   ├── 📄 router.go             # Route definitions
│   │   ├── 📄 middleware.go         # HTTP middleware
│   │   └── 📁 handlers/             # Request handlers
│   │
│   └── 📁 registry/                 # CQRS handlers
│       ├── 📁 commands/             # Write operations
│       │   └── 📁 incident/
│       │       ├── 📄 create_incident.go
│       │       ├── 📄 update_incident.go
│       │       └── 📄 close_incident.go
│       └── 📁 queries/              # Read operations
│           └── 📁 incident/
│               ├── 📄 get_incident.go
│               └── 📄 list_incidents.go
│
├── 📁 pkg/                          # Shared packages (can be imported by others)
│   ├── 📁 config/                   # Configuration loading
│   ├── 📁 errors/                   # Error handling
│   ├── 📁 logger/                   # Logging utilities
│   ├── 📁 postgres/                 # Database connections
│   ├── 📁 redis/                    # Redis connections
│   └── 📁 response/                 # HTTP response utilities
│
├── 📁 tests/                        # Tests
│   ├── 📁 unit/                     # Unit tests
│   └── 📁 integration/              # Integration tests
│
├── 📁 migrations/                   # Database migrations
├── 📁 docs/                         # Documentation & Swagger
├── 📁 k8s/                          # Kubernetes manifests
│
├── 📄 config.yaml                   # Application configuration
├── 📄 Dockerfile                    # Container definition
├── 📄 Makefile                      # Build automation
└── 📄 go.mod                        # Go dependencies
```

### Key Directories Explained

| Directory | Purpose | When to Modify |
|-----------|---------|----------------|
| `cmd/server/` | Application bootstrap, dependency injection | Adding new services/handlers |
| `internal/domain/` | Business logic, entities, interfaces | Adding business rules |
| `internal/adapters/` | External integrations | Changing database, adding APIs |
| `internal/http/` | HTTP routing, handlers | Adding endpoints |
| `internal/registry/` | CQRS commands & queries | Adding new operations |
| `pkg/` | Reusable utilities | Adding shared functionality |
| `tests/` | Test files | Writing tests (always!) |

<!-- 🖼️ IMAGE PLACEHOLDER: Folder structure diagram with color coding -->

---

## Configuration Management

### Understanding config.yaml

The configuration file is the **central place** for all application settings:

```yaml
# config.yaml - Application Configuration

# Server settings
server:
  port: 8080                    # Which port to listen on
  timeout: 30s                  # Request timeout

# Database configuration
database:
  master:                       # For WRITE operations
    host: "34.147.222.243"      # Database IP/hostname
    port: 5432                  # PostgreSQL default port
    user: "core"                # Database username
    password: "secret"          # Database password (use env vars in production!)
    dbname: "core"              # Database name
    max_open_conns: 25          # Maximum open connections
    max_idle_conns: 5           # Idle connections to keep
  slave:                        # For READ operations
    host: "34.147.222.243"      # Can be different host (replica)
    port: 5432
    user: "core"
    password: "secret"
    dbname: "core"
    max_open_conns: 50          # More connections for reads
    max_idle_conns: 10

# Logging settings
logging:
  level: "info"                 # debug, info, warn, error
  format: "console"             # console or json

# External services
notification:
  url: "https://beacon-tcd.tech/api/notification/v1"

iam:
  url: "https://beacon-tcd.tech/api/iam/v1"

# Redis cache
redis:
  host: "35.230.138.109"
  port: 6379
  password: ""
  db: 0
  ttl: 3600                     # Cache TTL in seconds (1 hour)
```

### Configuration Best Practices

#### 1. Never Commit Secrets

```yaml
# ❌ BAD - Never do this in real config.yaml
password: "actual-password-123"

# ✅ GOOD - Use environment variables
password: ${BEACON_DB_PASSWORD}
```

#### 2. Use Environment Variable Overrides

The config system supports environment variable overrides:

```bash
# Override database host via environment variable
export BEACON_DATABASE_MASTER_HOST="new-host.example.com"

# The application will use environment variable over config.yaml
```

#### 3. Different Configs for Different Environments

```
config/
├── config.yaml          # Default/development
├── config.staging.yaml  # Staging environment
└── config.prod.yaml     # Production environment
```

### How Configuration is Loaded

```go
// File: pkg/config/config.go

func Load(configPath string) (*Config, error) {
    v := viper.New()
    
    // Set config file path
    v.SetConfigFile(configPath)
    v.SetConfigType("yaml")
    
    // Read from file
    if err := v.ReadInConfig(); err != nil {
        return nil, err
    }
    
    // Allow environment variable overrides
    v.SetEnvPrefix("BEACON")                           // BEACON_*
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_")) // database.master → DATABASE_MASTER
    v.AutomaticEnv()
    
    // Parse into struct
    var cfg Config
    v.Unmarshal(&cfg)
    
    return &cfg, nil
}
```

<!-- 🖼️ IMAGE PLACEHOLDER: Configuration loading flow diagram -->

---

## Test-Driven Development (TDD)

### What is TDD?

TDD is a development approach where you **write tests before writing code**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TDD Cycle (Red-Green-Refactor)               │
│                                                                  │
│         ┌──────────┐                                            │
│         │  1. RED  │ ◄─────────────────────────────────┐        │
│         │ Write a  │                                    │        │
│         │ failing  │                                    │        │
│         │  test    │                                    │        │
│         └────┬─────┘                                    │        │
│              │                                          │        │
│              ▼                                          │        │
│         ┌──────────┐                              ┌────┴─────┐  │
│         │ 2. GREEN │                              │3.REFACTOR│  │
│         │ Write    │ ──────────────────────────►  │ Improve  │  │
│         │ minimum  │                              │  code    │  │
│         │ code to  │                              │ quality  │  │
│         │  pass    │                              └──────────┘  │
│         └──────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why TDD?

| Benefit | Explanation |
|---------|------------|
| **Catches bugs early** | Tests run before code is written |
| **Better design** | Forces you to think about interfaces first |
| **Documentation** | Tests show how code should be used |
| **Confidence** | Refactor without fear of breaking things |
| **Faster debugging** | Tests pinpoint exactly what's broken |

### Step-by-Step TDD Example

Let's add a new feature: **Checking if an incident can be reopened**

#### Step 1: Write the Test First (RED 🔴)

```go
// File: internal/domain/incident/domain_test.go

func TestIncident_CanReopen(t *testing.T) {
    tests := []struct {
        name   string
        status IncidentStatus
        want   bool
    }{
        {
            name:   "closed incident can be reopened",
            status: StatusClosed,
            want:   true,
        },
        {
            name:   "resolved incident can be reopened",
            status: StatusResolved,
            want:   true,
        },
        {
            name:   "active incident cannot be reopened",
            status: StatusActive,
            want:   false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            incident := &Incident{Status: tt.status}
            
            if got := incident.CanReopen(); got != tt.want {
                t.Errorf("CanReopen() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

Run the test - it will **fail** (because `CanReopen` doesn't exist):

```bash
$ go test ./internal/domain/incident/... -run TestIncident_CanReopen

# ERROR: incident.CanReopen undefined
```

#### Step 2: Write Minimum Code to Pass (GREEN 🟢)

```go
// File: internal/domain/incident/domain.go

// CanReopen checks if incident can be reopened
func (i *Incident) CanReopen() bool {
    return i.Status == StatusClosed || i.Status == StatusResolved
}
```

Run the test again - it should **pass**:

```bash
$ go test ./internal/domain/incident/... -run TestIncident_CanReopen

# PASS
```

#### Step 3: Refactor if Needed (REFACTOR 🔵)

```go
// If needed, improve the code while keeping tests passing
// In this case, the code is simple enough
```

### Testing Patterns in Beacon Core

#### 1. Table-Driven Tests

This is Go's **standard pattern** for testing multiple cases:

```go
func TestIncident_ValidateStatus(t *testing.T) {
    tests := []struct {
        name      string          // Descriptive name
        current   IncidentStatus  // Input state
        next      IncidentStatus  // What we're testing
        wantValid bool            // Expected result
    }{
        {
            name:      "reported to verified",
            current:   StatusReported,
            next:      StatusVerified,
            wantValid: true,
        },
        // Add more test cases...
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            incident := &Incident{Status: tt.current}
            got := incident.ValidateStatus(tt.next)
            
            if got != tt.wantValid {
                t.Errorf("ValidateStatus() = %v, want %v", got, tt.wantValid)
            }
        })
    }
}
```

#### 2. Using Mock Repositories

```go
// File: tests/unit/alert_citizens_test.go

// Create mock implementations
func TestAlertCitizens_Success(t *testing.T) {
    // Setup mocks
    masterRepo := NewMockMasterRepository()
    slaveRepo := NewMockSlaveRepository()
    mockNotif := &MockNotificationClient{ShouldFail: false}
    
    // Create service with mocks
    service := incident.NewService(masterRepo, slaveRepo, mockNotif, nil)
    
    // Add test data
    slaveRepo.AddIncident(&incident.Incident{
        ID:     "test-123",
        Status: incident.StatusActive,
        Title:  "Test Incident",
    })
    
    // Test the functionality
    handler := incidentcmd.NewAlertCitizensHandler(service)
    err := handler.Handle(context.Background(), incidentcmd.AlertCitizensCommand{
        IncidentID: "test-123",
    })
    
    // Assertions
    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    if mockNotif.CallCount != 1 {
        t.Errorf("Expected 1 notification call, got %d", mockNotif.CallCount)
    }
}
```

### Running Tests

```bash
# Run all tests
make test

# Run specific test file
go test -v ./internal/domain/incident/...

# Run specific test function
go test -v ./internal/domain/incident/... -run TestIncident_ValidateStatus

# Run with coverage
go test -cover ./...

# Run integration tests
make test-crud-integration
```

<!-- 🖼️ IMAGE PLACEHOLDER: TDD cycle diagram with code examples -->

---

## Good Coding Practices

### 1. Follow Go Naming Conventions

```go
// ✅ GOOD
type IncidentService struct {}      // Exported (capital letter)
func (s *IncidentService) Create()  // Exported method

type incidentHelper struct {}       // Unexported (lowercase)
func (h *incidentHelper) validate() // Unexported method

// Variable naming
var userID string     // camelCase for variables
const MaxRetries = 3  // PascalCase for exported constants
const maxRetries = 3  // camelCase for unexported constants

// ❌ BAD
type Incident_Service struct {}  // No underscores in type names
var user_id string               // No underscores in variables
```

### 2. Error Handling

```go
// ✅ GOOD - Always handle errors explicitly
result, err := repository.GetByID(ctx, id)
if err != nil {
    return nil, fmt.Errorf("failed to get incident: %w", err)
}

// ✅ GOOD - Use custom error types
if err != nil {
    return nil, errors.NotFound("incident not found")
}

// ❌ BAD - Ignoring errors
result, _ := repository.GetByID(ctx, id)  // Never do this!
```

### 3. Context Usage

```go
// ✅ GOOD - Pass context as first parameter
func (s *Service) CreateIncident(ctx context.Context, incident *Incident) error {
    // Use context for cancellation and deadlines
    if err := s.masterRepo.Create(ctx, incident); err != nil {
        return err
    }
    return nil
}

// ❌ BAD - Using background context inappropriately
func (s *Service) CreateIncident(incident *Incident) error {
    ctx := context.Background()  // Loses cancellation info
    // ...
}
```

### 4. Interface Segregation

```go
// ✅ GOOD - Small, focused interfaces
type Reader interface {
    GetByID(ctx context.Context, id string) (*Incident, error)
}

type Writer interface {
    Create(ctx context.Context, incident *Incident) error
}

// ❌ BAD - Fat interfaces
type Repository interface {
    GetByID(ctx context.Context, id string) (*Incident, error)
    List(ctx context.Context) ([]*Incident, error)
    Create(ctx context.Context, incident *Incident) error
    Update(ctx context.Context, incident *Incident) error
    Delete(ctx context.Context, id string) error
    GetMetrics(ctx context.Context) (*Metrics, error)
    SendNotification(ctx context.Context, id string) error
    // Too many responsibilities!
}
```

### 5. Documentation

```go
// ✅ GOOD - Document exported functions
// CreateIncident creates a new incident and sends notifications.
// It validates the incident data, assigns an ID, and persists to database.
// Returns the created incident or an error if validation/persistence fails.
func (s *Service) CreateIncident(ctx context.Context, incident *Incident) (*Incident, error) {
    // ...
}

// ✅ GOOD - Use godoc conventions
// Package incident provides domain logic for incident management.
// 
// The incident domain handles creation, updates, and lifecycle
// management of incidents in the disaster response system.
package incident
```

### 6. Code Organization

```go
// ✅ GOOD - Group related code together
type Service struct {
    // Dependencies first
    masterRepo MasterRepository
    slaveRepo  SlaveRepository
    cache      CacheRepository
}

// Constructor immediately after struct
func NewService(master MasterRepository, slave SlaveRepository, cache CacheRepository) *Service {
    return &Service{
        masterRepo: master,
        slaveRepo:  slave,
        cache:      cache,
    }
}

// Public methods next
func (s *Service) Create(ctx context.Context, i *Incident) error { ... }
func (s *Service) Get(ctx context.Context, id string) (*Incident, error) { ... }

// Private methods last
func (s *Service) validateIncident(i *Incident) error { ... }
```

### 7. Early Returns (Guard Clauses)

```go
// ✅ GOOD - Early returns for cleaner code
func (s *Service) GetIncident(ctx context.Context, id string) (*Incident, error) {
    if id == "" {
        return nil, errors.BadRequest("incident ID is required")
    }
    
    incident, err := s.slaveRepo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }
    
    return incident, nil
}

// ❌ BAD - Nested conditionals
func (s *Service) GetIncident(ctx context.Context, id string) (*Incident, error) {
    if id != "" {
        incident, err := s.slaveRepo.GetByID(ctx, id)
        if err == nil {
            return incident, nil
        } else {
            return nil, err
        }
    } else {
        return nil, errors.BadRequest("incident ID is required")
    }
}
```

<!-- 🖼️ IMAGE PLACEHOLDER: Code organization diagram -->

---

## Using GitHub Copilot

GitHub Copilot is an **AI-powered coding assistant** that can significantly boost your productivity.

### Setting Up Copilot in VS Code

1. Install the **GitHub Copilot** extension in VS Code
2. Sign in with your GitHub account
3. Authorize the extension

### Basic Copilot Usage

#### 1. Inline Suggestions

Just start typing and Copilot will suggest completions:

```go
// Type a function signature and Copilot suggests the body
func (s *Service) GetActiveIncidents(ctx context.Context) ([]*Incident, error) {
    // Copilot will suggest implementation based on context
```

> Press `Tab` to accept, `Esc` to dismiss

#### 2. Writing Comments First

```go
// GetActiveIncidents retrieves all incidents with status "active"
// sorted by severity in descending order
func (s *Service) GetActiveIncidents(ctx context.Context) ([]*Incident, error) {
    // Copilot uses the comment to generate accurate code
}
```

#### 3. Using Copilot Chat

Press `Ctrl+I` (or `Cmd+I` on Mac) to open Copilot Chat. Ask questions like:

- "Explain this function"
- "Write a test for this method"
- "How do I implement pagination?"
- "Find bugs in this code"

### Copilot for Common Tasks

#### Writing Tests

```go
// In a test file, type:
// TestService_CreateIncident tests the CreateIncident method
func TestService_CreateIncident(t *testing.T) {
    // Copilot will generate test cases
}
```

#### Implementing Interfaces

```go
// Type the struct and the interface to implement
type PostgresRepository struct {
    db *sql.DB
}

// Copilot can implement all interface methods
var _ incident.MasterRepository = (*PostgresRepository)(nil)

// Start typing and Copilot suggests implementations
func (r *PostgresRepository) Create
```

#### Writing SQL Queries

```go
// Copilot is great at generating SQL
query := `
    SELECT id, type, severity, status, title, description
    FROM incidents
    WHERE status = $1
    ORDER BY created_at DESC
    LIMIT $2 OFFSET $3
`
```

### Copilot Best Practices

| Do ✅ | Don't ❌ |
|-------|---------|
| Review all suggestions carefully | Blindly accept without reading |
| Use descriptive comments | Accept code you don't understand |
| Break complex tasks into steps | Expect perfect code every time |
| Learn from suggestions | Forget to write tests |
| Use Copilot Chat for explanations | Share sensitive data in prompts |

### Copilot Keyboard Shortcuts

| Action | Windows/Linux | Mac |
|--------|---------------|-----|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| See next suggestion | `Alt + ]` | `Option + ]` |
| See previous suggestion | `Alt + [` | `Option + [` |
| Open Copilot Chat | `Ctrl + I` | `Cmd + I` |
| Explain code | Select + right-click → "Explain" | Same |

### Example: Using Copilot to Add a New Feature

Let's add a "Reopen Incident" feature:

1. **Start with a comment describing what you want:**

```go
// ReopenIncident reopens a closed or resolved incident
// It validates that the incident can be reopened, updates its status to "reported",
// and clears the resolved_at and closed_at timestamps
func (s *Service) ReopenIncident(ctx context.Context, id string) (*Incident, error) {
    // Copilot will suggest implementation
}
```

2. **Ask Copilot Chat to write tests:**

```
@workspace Write tests for the ReopenIncident function
```

3. **Use Copilot to implement the command handler:**

```go
// ReopenIncidentCommand represents a command to reopen a closed incident
type ReopenIncidentCommand struct {
    IncidentID string
}

// Copilot will suggest the handler implementation
```

<!-- 🖼️ IMAGE PLACEHOLDER: Copilot interface screenshots -->

---

## Running the Application

### Prerequisites

1. **Go 1.24+** installed
2. **Docker** (for local database)
3. **Make** utility

### Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/Response-To-City-Disaster/beacon-core.git
cd beacon-core

# 2. Install dependencies
make deps

# 3. Start PostgreSQL (using Docker)
docker run -d \
  --name beacon-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=beacon \
  -p 5432:5432 \
  postgres:15-alpine

# 4. Run database migrations
export DB_URL="postgresql://postgres:postgres@localhost:5432/beacon?sslmode=disable"
make migrate-up

# 5. Run the application
go run ./cmd/server/main.go

# Or use Make
make run
```

### Verify It's Running

```bash
# Check health endpoint
curl http://localhost:8080/health

# Expected response:
# {"status": "healthy"}
```

### View API Documentation

Open Swagger UI in your browser:
```
http://localhost:8080/swagger/index.html
```

---

## Common Tasks & Commands

### Makefile Commands

```bash
# Show all available commands
make help

# Install development tools
make install-tools

# Run linters
make lint

# Run all tests
make test

# Run integration tests
make test-crud-integration

# Build the application
make build

# Run the application
make run

# Generate Swagger documentation
make swagger

# Build Docker image
make docker-build
```

### Database Commands

```bash
# Run migrations up
make migrate-up

# Rollback migrations
make migrate-down

# Create new migration
migrate create -ext sql -dir migrations -seq create_new_table
```

### Git Workflow

```bash
# Create a feature branch
git checkout -b feature/add-reopen-incident

# Make changes and commit
git add .
git commit -m "feat: add reopen incident functionality"

# Push and create PR
git push origin feature/add-reopen-incident
```

### Debugging Tips

```bash
# Run with debug logging
export BEACON_LOGGING_LEVEL=debug
go run ./cmd/server/main.go

# Check database connection
psql -h localhost -U postgres -d beacon

# View application logs
make run 2>&1 | tee app.log
```

---

## Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│                    BEACON CORE QUICK REFERENCE                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📁 KEY DIRECTORIES                                             │
│  ─────────────────                                              │
│  cmd/server/main.go    → Application entry point               │
│  internal/domain/      → Business logic (no dependencies!)     │
│  internal/adapters/    → Database/API implementations          │
│  internal/registry/    → CQRS commands and queries             │
│  internal/http/        → HTTP handlers and routing             │
│                                                                 │
│  🏗️ ARCHITECTURE LAYERS                                         │
│  ──────────────────────                                         │
│  HTTP → Registry (CQRS) → Domain → Adapters                    │
│                                                                 │
│  📝 ADD NEW FEATURE CHECKLIST                                   │
│  ───────────────────────────                                    │
│  □ Write tests first (TDD)                                     │
│  □ Add domain logic in internal/domain/                        │
│  □ Add command/query in internal/registry/                     │
│  □ Add HTTP handler in internal/http/handlers/                 │
│  □ Add route in internal/http/router.go                        │
│  □ Update Swagger docs                                         │
│  □ Run tests: make test                                        │
│                                                                 │
│  🔧 COMMON COMMANDS                                             │
│  ─────────────────                                              │
│  make run          → Run the application                       │
│  make test         → Run all tests                             │
│  make lint         → Check code quality                        │
│  make swagger      → Generate API docs                         │
│                                                                 │
│  🆘 NEED HELP?                                                  │
│  ────────────                                                   │
│  • Check README.md                                              │
│  • Ask in team Slack channel                                    │
│  • Use GitHub Copilot Chat                                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Summary

Congratulations! 🎉 You now understand:

1. ✅ **Go basics** - Packages, structs, interfaces, methods
2. ✅ **Hexagonal Architecture** - Ports (interfaces) and Adapters (implementations)
3. ✅ **CQRS Pattern** - Separate commands (writes) from queries (reads)
4. ✅ **Adapter Pattern** - Translating between domain and external systems
5. ✅ **Microservices** - Independent services communicating over HTTP
6. ✅ **Project Structure** - Where to find and add code
7. ✅ **Configuration** - How settings are managed
8. ✅ **TDD** - Write tests before code
9. ✅ **Coding Practices** - How to write clean Go code
10. ✅ **GitHub Copilot** - AI-powered productivity

### Next Steps

1. Set up your local development environment
2. Run the application and explore the API
3. Pick a small task from the issue tracker
4. Use TDD to implement the feature
5. Create a pull request for review

**Happy coding! 🚀**

---

## Learning Resources

This section contains curated resources to deepen your understanding of Go, architecture patterns, and backend development.

### 📚 Go (Golang)

#### Official Resources
| Resource | Type | Description |
|----------|------|-------------|
| [A Tour of Go](https://go.dev/tour/) | Interactive | Official interactive tutorial - **START HERE** |
| [Effective Go](https://go.dev/doc/effective_go) | Documentation | Official guide on writing idiomatic Go |
| [Go by Example](https://gobyexample.com/) | Examples | Hands-on introduction with annotated examples |
| [Go Documentation](https://go.dev/doc/) | Documentation | Official Go documentation |

#### Video Courses
| Resource | Platform | Description |
|----------|----------|-------------|
| [Learn Go Programming](https://www.youtube.com/watch?v=YS4e4q9oBaU) | YouTube (freeCodeCamp) | 7-hour comprehensive course for beginners |
| [Go Programming – Golang Course](https://www.youtube.com/watch?v=un6ZyFkqFKo) | YouTube (freeCodeCamp) | Updated full course with projects |
| [Golang Tutorial for Beginners](https://www.youtube.com/watch?v=yyUHQIec83I) | YouTube (TechWorld with Nana) | 3-hour beginner-friendly tutorial |
| [Go: The Complete Developer's Guide](https://www.udemy.com/course/go-the-complete-developers-guide/) | Udemy | Paid comprehensive course by Stephen Grider |
| [Learn How To Code: Google's Go](https://www.udemy.com/course/learn-how-to-code/) | Udemy | Highly rated course by Todd McLeod |

#### Books
| Book | Author | Level |
|------|--------|-------|
| [Learning Go](https://www.oreilly.com/library/view/learning-go/9781492077206/) | Jon Bodner | Beginner |
| [The Go Programming Language](https://www.gopl.io/) | Donovan & Kernighan | Intermediate |
| [Concurrency in Go](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/) | Katherine Cox-Buday | Advanced |
| [100 Go Mistakes and How to Avoid Them](https://www.manning.com/books/100-go-mistakes-and-how-to-avoid-them) | Teiva Harsanyi | Intermediate |

#### Blogs & Articles
| Resource | Description |
|----------|-------------|
| [Go Blog](https://go.dev/blog/) | Official Go team blog |
| [Dave Cheney's Blog](https://dave.cheney.net/) | Deep dives into Go internals |
| [Ardan Labs Blog](https://www.ardanlabs.com/blog/) | Production Go patterns |
| [Alex Edwards' Blog](https://www.alexedwards.net/blog) | Practical Go web development |

---

### 🏗️ Hexagonal Architecture

#### Articles
| Resource | Description |
|----------|-------------|
| [Hexagonal Architecture - Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/) | Original article by the pattern creator |
| [Ports and Adapters Pattern](https://herbertograca.com/2017/09/14/ports-adapters-architecture/) | Comprehensive explanation by Herberto Graça |
| [Ready for changes with Hexagonal Architecture](https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749) | Netflix's implementation |
| [Hexagonal Architecture in Go](https://medium.com/@matiasvarela/hexagonal-architecture-in-go-cfd4e436faa3) | Go-specific implementation guide |

#### Videos
| Resource | Platform | Description |
|----------|----------|-------------|
| [Hexagonal Architecture](https://www.youtube.com/watch?v=bDWApqAUjEI) | YouTube | Clear explanation with examples |
| [Clean Architecture in Go](https://www.youtube.com/watch?v=goC-gCNWhS4) | YouTube (GopherCon) | Go-specific clean architecture talk |

---

### 📋 CQRS Pattern

#### Articles
| Resource | Description |
|----------|-------------|
| [CQRS - Martin Fowler](https://martinfowler.com/bliki/CQRS.html) | Introduction by Martin Fowler |
| [CQRS Pattern - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) | Microsoft's comprehensive guide |
| [CQRS in Go](https://threedots.tech/post/cqrs-in-go/) | Go-specific implementation by Three Dots Labs |
| [When to use CQRS](https://www.eventstore.com/cqrs-pattern) | Understanding when CQRS makes sense |

#### Videos
| Resource | Platform | Description |
|----------|----------|-------------|
| [CQRS and Event Sourcing](https://www.youtube.com/watch?v=JHGkaShoyNs) | YouTube (Greg Young) | Talk by one of the pattern pioneers |
| [CQRS in Practice](https://www.youtube.com/watch?v=5CqjYsfNnkI) | YouTube | Practical implementation examples |

---

### 🔌 Microservices

#### Courses
| Resource | Platform | Description |
|----------|----------|-------------|
| [Microservices with Go](https://www.youtube.com/playlist?list=PLmD8u-IFdreyh6EUfevBcbiuCKzFk0EW_) | YouTube (Nic Jackson) | Free video series from HashiCorp |
| [Building Microservices with Go](https://www.youtube.com/playlist?list=PLJbE2Yu2zumAixEws7gtptADSLmZ_pscP) | YouTube | Practical microservices series |
| [gRPC in Go](https://www.youtube.com/watch?v=2Sm_O75I7H0) | YouTube | gRPC fundamentals for Go |

#### Books
| Book | Author | Description |
|------|--------|-------------|
| [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/) | Sam Newman | Industry standard book |
| [Microservices Patterns](https://www.manning.com/books/microservices-patterns) | Chris Richardson | Patterns and best practices |
| [Production-Ready Microservices](https://www.oreilly.com/library/view/production-ready-microservices/9781491965962/) | Susan Fowler | Operationalizing microservices |

#### Blogs
| Resource | Description |
|----------|-------------|
| [Microservices.io](https://microservices.io/) | Patterns and best practices by Chris Richardson |
| [Three Dots Labs](https://threedots.tech/) | Go microservices tutorials |
| [Go Kit](https://gokit.io/) | Microservices toolkit for Go |

---

### 🧪 Testing in Go

#### Articles & Tutorials
| Resource | Description |
|----------|-------------|
| [Learn Go with Tests](https://quii.gitbook.io/learn-go-with-tests/) | **HIGHLY RECOMMENDED** - TDD approach to learning Go |
| [Testing in Go](https://blog.alexellis.io/golang-writing-unit-tests/) | Practical testing guide |
| [Table Driven Tests](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) | Go testing best practices |
| [Go Testing By Example](https://research.swtch.com/testing) | Testing philosophy by Russ Cox |

#### Videos
| Resource | Platform | Description |
|----------|----------|-------------|
| [Advanced Testing in Go](https://www.youtube.com/watch?v=8hQG7QlcLBk) | YouTube (GopherCon) | Advanced testing techniques |
| [Test-Driven Development in Go](https://www.youtube.com/watch?v=EZ05e7EMOLM) | YouTube | TDD workflow demonstration |

---

### 🗄️ Databases & SQL

#### PostgreSQL
| Resource | Description |
|----------|-------------|
| [PostgreSQL Tutorial](https://www.postgresqltutorial.com/) | Comprehensive PostgreSQL guide |
| [Use The Index, Luke](https://use-the-index-luke.com/) | SQL indexing and performance |
| [PostgreSQL Exercises](https://pgexercises.com/) | Interactive SQL practice |

#### Go + Databases
| Resource | Description |
|----------|-------------|
| [Go database/sql tutorial](http://go-database-sql.org/) | Official patterns for database/sql |
| [sqlc](https://sqlc.dev/) | Generate type-safe Go from SQL |
| [GORM](https://gorm.io/docs/) | Popular Go ORM documentation |

---

### 🐳 Docker & Kubernetes

#### Docker
| Resource | Type | Description |
|----------|------|-------------|
| [Docker for Beginners](https://docker-curriculum.com/) | Tutorial | Comprehensive Docker intro |
| [Docker Official Tutorial](https://www.docker.com/101-tutorial/) | Tutorial | Official getting started |
| [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) | Guide | Official best practices |

#### Kubernetes
| Resource | Type | Description |
|----------|------|-------------|
| [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/) | Tutorial | Official interactive tutorial |
| [Learn Kubernetes](https://www.youtube.com/watch?v=X48VuDVv0do) | YouTube (TechWorld with Nana) | 4-hour comprehensive course |
| [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) | Guide | Deep dive by Kelsey Hightower |

---

### 🛠️ Development Tools

#### Git
| Resource | Description |
|----------|-------------|
| [Git Tutorial](https://www.atlassian.com/git/tutorials) | Atlassian's comprehensive Git guide |
| [Learn Git Branching](https://learngitbranching.js.org/) | Interactive visual Git learning |
| [Conventional Commits](https://www.conventionalcommits.org/) | Commit message conventions |

#### VS Code & Copilot
| Resource | Description |
|----------|-------------|
| [Go in VS Code](https://code.visualstudio.com/docs/languages/go) | Official VS Code Go setup |
| [GitHub Copilot Docs](https://docs.github.com/en/copilot) | Official Copilot documentation |
| [Copilot Tips](https://github.blog/2023-06-20-how-to-write-better-prompts-for-github-copilot/) | Writing effective prompts |

---

### 📺 YouTube Channels to Follow

| Channel | Focus |
|---------|-------|
| [Golang Dojo](https://www.youtube.com/@GolangDojo) | Go tutorials and tips |
| [Anthony GG](https://www.youtube.com/@anthonygg_) | Go projects and best practices |
| [Melkey](https://www.youtube.com/@MelkeyDev) | Go development and streaming |
| [Dreams of Code](https://www.youtube.com/@dreamsofcode) | Developer productivity |
| [ThePrimeagen](https://www.youtube.com/@ThePrimeagen) | Performance and Vim |
| [Hussein Nasser](https://www.youtube.com/@haborMaker) | Backend architecture deep dives |

---

### 🎯 Practice Platforms

| Platform | Description |
|----------|-------------|
| [Exercism - Go Track](https://exercism.org/tracks/go) | Free mentored exercises |
| [LeetCode](https://leetcode.com/) | Algorithm practice |
| [Gophercises](https://gophercises.com/) | Go-specific project exercises |
| [Advent of Code](https://adventofcode.com/) | Annual coding challenges |

---

### 📖 Recommended Learning Path

```
Week 1-2: Go Fundamentals
├── Complete "A Tour of Go"
├── Watch freeCodeCamp Go course
└── Do first 10 Exercism exercises

Week 3-4: Go in Practice
├── Read "Learn Go with Tests"
├── Build a simple REST API
└── Understand testing patterns

Week 5-6: Architecture Patterns
├── Read Hexagonal Architecture articles
├── Study CQRS pattern
└── Review Beacon Core codebase

Week 7-8: Production Skills
├── Docker fundamentals
├── Database patterns
└── Start contributing to Beacon Core!
```
