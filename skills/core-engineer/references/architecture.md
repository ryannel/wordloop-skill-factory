# Architecture & Layer Boundaries

## Table of Contents
- [Hexagonal Philosophy](#hexagonal-philosophy)
- [Dependency Rules](#dependency-rules)
- [Directory Layout](#directory-layout)
- [Layer Contracts](#layer-contracts)
- [The Wiring Layer](#the-wiring-layer)
- [Import Validation](#import-validation)
- [Migrations](#migrations)

---

## Hexagonal Philosophy

The wordloop-core service implements hexagonal architecture (ports and adapters) to achieve total isolation of business logic from external concerns. The core benefit is that the most volatile parts of the system — databases, frameworks, HTTP transport — can evolve independently from the most valuable part: the domain logic.

The architecture divides the application into:

- **Inner hexagon** (Domain + Gateway interfaces): Pure business logic and the contracts it requires
- **Outer hexagon** (Providers + Entrypoints): Concrete implementations of those contracts

All communication flows through well-defined interfaces. The domain never reaches outward; external technologies always reach inward.

### Why This Matters

This isolation provides three capabilities that compound over time:

1. **Swappability** — Replace Postgres with another store, or REST with gRPC, without touching business logic
2. **Testability** — Domain and service logic are testable without databases, networks, or API keys
3. **Stability** — The business core remains untouched during infrastructure upgrades, dependency updates, or transport changes

## Dependency Rules

Dependencies flow in one direction: **inward**. This is the single most important rule in the codebase.

```
                    ┌──────────────────────────────────────────┐
                    │              ENTRYPOINTS                  │
                    │    (HTTP handlers, middleware, routing)    │
                    └──────────────┬───────────────────────────┘
                                   │ depends on
                    ┌──────────────▼───────────────────────────┐
                    │              SERVICES                     │
                    │        (Use-case orchestration)           │
                    │                                           │
                    │    ┌─────────────────────────────────┐   │
                    │    │           GATEWAYS               │   │
                    │    │     (Repository interfaces)      │   │
                    │    │                                   │   │
                    │    │    ┌─────────────────────────┐   │   │
                    │    │    │         DOMAIN           │   │   │
                    │    │    │      (Pure logic)        │   │   │
                    │    │    └─────────────────────────┘   │   │
                    │    │                                   │   │
                    │    └─────────────────────────────────┘   │
                    │                                           │
                    └──────────────▲───────────────────────────┘
                                   │ depends on
                    ┌──────────────┴───────────────────────────┐
                    │              PROVIDERS                    │
                    │   (Postgres, Redis, S3, external APIs)    │
                    └──────────────────────────────────────────┘

    Entrypoints (driving/inbound) and Providers (driven/outbound) are
    sibling outer-layer adapters. Neither imports the other. Both point
    inward toward the core. Only cmd/ has visibility into all layers.
```

### Allowed Import Relationships

| Layer | Can Import | Cannot Import |
|-------|-----------|---------------|
| **Domain** | Standard library only | Everything else |
| **Gateway** | Domain | Service, Provider, Entrypoint |
| **Service** | Domain, Gateway | Provider, Entrypoint |
| **Provider** | Domain, Gateway | Service, Entrypoint |
| **Entrypoint** | Domain, Service | Provider, Gateway (directly) |

Providers and Entrypoints exist at the same depth in the hexagon, but they serve different roles:
- **Providers** fulfill gateway contracts (driven/outbound adapters)
- **Entrypoints** invoke services (driving/inbound adapters)

Neither should import the other.

### What "Inward" Means in Practice

When writing provider code, you implement a gateway interface using concrete technology. The provider knows about the gateway contract and domain types, but the service that uses that contract has no knowledge of the provider's existence.

When writing entrypoint code, you call a service method. The entrypoint knows about service interfaces and domain types, but never directly interacts with a provider.

## Directory Layout

```
wordloop-core/
├── cmd/                              # Binary entry points
│   └── api/
│       └── main.go                   # Wiring: construct all deps, start server
│
├── scripts/
│   └── migrations/                   # Timestamped SQL migrations
│       ├── 20260408120000_create_meetings.up.sql
│       └── 20260408120000_create_meetings.down.sql
│
└── internal/                         # Private — cannot be imported by external packages
    ├── core/
    │   ├── domain/                   # THE INNERMOST LAYER
    │   │   ├── meeting.go            # Entity + NewMeeting() + methods
    │   │   ├── meeting_test.go       # Pure unit tests
    │   │   ├── user.go
    │   │   └── errors.go             # Sentinel errors: ErrNotFound, ErrConflict, etc.
    │   │
    │   ├── gateway/                  # REPOSITORY INTERFACES (driven ports)
    │   │   ├── meeting.go            # MeetingRepository interface
    │   │   ├── user.go               # UserRepository interface
    │   │   └── mocks/                # Generated mocks for service-layer testing
    │   │       ├── meeting.go
    │   │       └── user.go
    │   │
    │   └── service/                  # USE-CASE ORCHESTRATION
    │       ├── meeting.go            # MeetingService struct + New constructor
    │       ├── meeting_test.go       # Tests using gateway mocks
    │       └── mocks/                # Generated mocks for entrypoint-layer testing
    │           └── meeting.go
    │
    ├── provider/                     # CONCRETE IMPLEMENTATIONS
    │   ├── postgres/
    │   │   ├── meeting.go            # PostgresMeetingRepository
    │   │   ├── meeting_test.go       # Testcontainers-based integration tests
    │   │   └── testdb_setup_test.go  # Shared test container setup
    │   ├── redis/
    │   │   └── cache.go
    │   └── s3/
    │       └── storage.go
    │
    └── entrypoints/
        └── api/
            ├── routes/               # HTTP HANDLERS
            │   ├── meeting.go        # MeetingHandler struct + route registration
            │   └── meeting_test.go   # httptest-based tests with service mocks
            ├── middleware/            # Cross-cutting concerns
            │   ├── auth.go
            │   ├── logging.go
            │   └── cors.go
            └── server.go             # HTTP server configuration
```

### Package Naming Conventions

- Avoid generic packages: no `utils`, `common`, `helpers`, or `shared`
- Use specific, descriptive names: `postgres`, `redis`, `auth`, `meeting`
- If a package grows too large, split by responsibility, not by creating deeper nesting
- Keep hierarchies shallow — deeply nested directories increase cognitive load

## Layer Contracts

### Domain Layer

The domain layer contains entities and value objects that model the business. Domain types are pure Go — no database tags, no JSON tags (those belong in the entrypoint's request/response types), no framework annotations.

Key rules:
- **Zero external dependencies** — import only the Go standard library
- **Always-valid construction** — use `New...` constructors with validation
- **Rich behavior** — if a rule only requires data from within a single struct, it must be a method on that struct
- **Side-effect free** — no database calls, no network I/O, no file system access
- **Sentinel errors** — define expected error conditions here, not in providers

### Gateway Layer

Gateways are interfaces that define what the domain **needs** from the outside world. They are the "driven ports" — contracts for persistence, external APIs, message publishing.

Key rules:
- Accept `context.Context` as the first parameter on every method
- Use domain types in method signatures (not database-specific types)
- Define interfaces with business-oriented names (e.g., `MeetingStore`, not `MeetingRepositoryInterface`)
- Keep interfaces small and focused — prefer composition of small interfaces
- Place mock implementations in `gateway/mocks/`

### Service Layer

Services orchestrate business workflows. They coordinate domain entities and gateway interfaces to fulfill use cases. They define transaction boundaries.

Key rules:
- Accept gateway interfaces via constructor injection
- Return concrete types from `New...` constructors
- Own the transaction boundary — tell providers when to begin/commit/rollback
- Contain no SQL, no HTTP, no framework-specific code
- Use `errgroup.WithContext` for parallel gateway operations

### Provider Layer

Providers are concrete implementations of gateway interfaces. This is where technology-specific code lives — SQL queries, Redis commands, S3 operations, external API calls.

Key rules:
- Implement gateway interfaces
- Wrap all driver errors with `fmt.Errorf("context: %w", err)`, mapping to domain sentinel errors where appropriate
- Never import the service layer
- Configure connection pools and resource limits here
- Place integration tests alongside provider code using testcontainers

### Entrypoint Layer

Entrypoints translate external requests into service calls. For HTTP, this means parsing requests, calling services, and writing responses.

Key rules:
- Import services, not providers
- Map domain errors to HTTP status codes
- Sanitize errors before returning to the client — never expose raw internal errors
- Log errors exactly once here — services and providers only wrap and propagate
- Handle request validation and response serialization

## The Wiring Layer

The `cmd/` directory is where all layers are assembled. This is the only place in the codebase that has visibility into every layer. The bootstrap code:

1. Loads configuration from environment variables
2. Initializes infrastructure (database pools, cache connections, telemetry providers)
3. Constructs providers with concrete dependencies
4. Constructs services with provider instances (injected as gateway interfaces)
5. Constructs entrypoint handlers with service instances
6. Starts the HTTP server with graceful shutdown support

```go
func main() {
    ctx := context.Background()
    
    cfg := config.MustLoad()
    
    // Infrastructure
    db := postgres.MustConnect(ctx, cfg.DatabaseURL)
    defer db.Close()
    
    // Providers (implement gateway interfaces)
    meetingRepo := postgres.NewMeetingRepository(db)
    
    // Services (depend on gateway interfaces)
    meetingSvc := service.NewMeetingService(meetingRepo)
    
    // Entrypoints (depend on services)
    handler := routes.NewMeetingHandler(meetingSvc)
    
    // Start server with graceful shutdown
    srv := api.NewServer(cfg, handler)
    srv.ListenAndServe(ctx)
}
```

No globals. No `init()`. Every dependency is explicit and flows through constructor parameters.

## Import Validation

When you are about to add an import, ask this question:

> "Does this import point inward (toward Domain) or outward (toward infrastructure)?"

If it points outward, the design needs to change. Common fixes:

- **Service needs a database type?** — Define a domain type and have the provider translate
- **Entrypoint needs provider logic?** — Extract the logic into a service method
- **Domain needs to call an API?** — Define a gateway interface and implement it in a provider

## Migrations

All schema changes are migration files in `scripts/migrations/`. Never modify the database schema manually or through application code.

### Naming Convention

```
YYYYMMDDHHMMSS_description.up.sql    # Apply the change
YYYYMMDDHHMMSS_description.down.sql  # Revert the change
```

### Rules

- Use `IF NOT EXISTS` and `IF EXISTS` for idempotency
- The service layer defines transaction boundaries; providers execute within them
- Each migration addresses a single concern
- Down migrations must fully reverse the up migration
- Test migrations against a clean database and against the current state
