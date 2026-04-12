# Interfaces

## Table of Contents
- [Accept Interfaces, Return Structs](#accept-interfaces-return-structs)
- [Keep Interfaces Small](#keep-interfaces-small)
- [Define Interfaces at the Consumer](#define-interfaces-at-the-consumer)
- [Compile-Time Verification](#compile-time-verification)
- [Functional Options](#functional-options)
- [Interface Antipatterns](#interface-antipatterns)

---

## Accept Interfaces, Return Structs

This is the foundational rule for constructors in wordloop-core:

```go
// Constructor accepts an interface — allows testing with mocks
func NewMeetingService(repo gateway.MeetingRepository) *MeetingService {
    return &MeetingService{repo: repo}
}
```

- **Accept interfaces** — constructor parameters are abstract contracts, enabling substitution with mocks in tests
- **Return concrete types** — constructors return `*ConcreteType`, not an interface, preserving full API surface for callers

## Keep Interfaces Small

Prefer single-method or few-method interfaces and compose them when needed:

```go
// Good: composable, focused interfaces
type MeetingReader interface {
    GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error)
}

type MeetingWriter interface {
    Create(ctx context.Context, m *domain.Meeting) error
    Update(ctx context.Context, m *domain.Meeting) error
}

// Compose when a consumer needs both
type MeetingReadWriter interface {
    MeetingReader
    MeetingWriter
}
```

Small interfaces are easier to implement, easier to mock, and easier to satisfy across different contexts. A consumer that only reads should accept `MeetingReader`, not `MeetingReadWriter`.

## Define Interfaces at the Consumer

Interfaces belong in the package that **uses** them, not the package that implements them. In wordloop-core, this means gateway interfaces are defined in the `gateway` package (where services consume them), not in the `provider` package (where they are implemented):

```go
// gateway/meeting.go — defines what the service needs
type MeetingRepository interface {
    Create(ctx context.Context, m *domain.Meeting) error
    GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error)
}

// provider/postgres/meeting.go — implements the gateway contract
type PostgresMeetingRepo struct { db *sql.DB }
// ... method implementations
```

This ensures the interface is shaped by genuine consumer needs, not by what the implementation happens to expose.

## Compile-Time Verification

Always include a compile-time check that your provider satisfies the gateway interface:

```go
var _ gateway.MeetingRepository = (*PostgresMeetingRepo)(nil)
```

This catches interface drift at compile time rather than runtime. Place this declaration immediately after the struct definition in the provider file.

## Functional Options

Use functional options for complex configuration that would otherwise require many constructor parameters or boolean flags:

```go
type ServerOption func(*Server)

func WithReadTimeout(d time.Duration) ServerOption {
    return func(s *Server) { s.readTimeout = d }
}

func WithMaxConnections(n int) ServerOption {
    return func(s *Server) { s.maxConns = n }
}

func WithTLS(certFile, keyFile string) ServerOption {
    return func(s *Server) {
        s.certFile = certFile
        s.keyFile = keyFile
    }
}

func NewServer(addr string, opts ...ServerOption) *Server {
    s := &Server{
        addr:        addr,
        readTimeout: 10 * time.Second, // sensible default
        maxConns:    100,              // sensible default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Call site is self-documenting:
// srv := NewServer(":8080",
//     WithReadTimeout(30 * time.Second),
//     WithMaxConnections(200),
//     WithTLS("cert.pem", "key.pem"),
// )
```

### When to Use Functional Options vs. Config Structs

| Pattern | Use When |
|---------|----------|
| Functional options | Optional settings with sensible defaults, public APIs, builder-like construction |
| Config struct | All fields are required, internal-only code, few options |

## Interface Antipatterns

### Interface Pollution

Defining interfaces before multiple consumers exist adds unnecessary abstraction:

```go
// WRONG — premature interface in the exporter package
package meeting
type MeetingRepositoryInterface interface { ... }

// CORRECT — interface defined by the consumer
package gateway
type MeetingRepository interface { ... }
```

### God Structs

Types managing unrelated responsibilities create fragile coupling:

```go
// WRONG — mixes concerns
type AppManager struct {
    DB     *sql.DB
    Cache  *redis.Client
    Logger *slog.Logger
    Config *config.Config
}

// CORRECT — single responsibility per struct
type MeetingService struct { meetings gateway.MeetingRepository }
type UserService struct { users gateway.UserRepository }
```

### Naked Parameters

Raw booleans and integers at call sites are unreadable:

```go
// WRONG — what does "true" mean?
db.Connect(30, true)

// CORRECT — self-documenting
db.Connect(
    postgres.WithTimeout(30 * time.Second),
    postgres.WithReadOnly(true),
)
```
