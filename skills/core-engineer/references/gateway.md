# Gateway Layer

## Table of Contents
- [Gateway Design](#gateway-design)
- [Interface Sizing](#interface-sizing)
- [Gateway Naming Conventions](#gateway-naming-conventions)
- [Composition Over Inheritance](#composition-over-inheritance)
- [Transactor Pattern](#transactor-pattern)
- [Mock Generation Strategy](#mock-generation-strategy)
- [Gateway Placement and Package Layout](#gateway-placement-and-package-layout)

---

## Gateway Design

Gateways are the **outbound contracts** of the hexagon. They define what the inner core (services) needs from the outside world — without specifying *how* those needs are fulfilled. Gateways are interfaces, never implementations; the implementations live in the provider layer.

Key rules:
- Gateways are **always interfaces**, defined in `internal/core/gateway/`
- They describe **capabilities needed by the domain**, not features of a technology
- Gateway interfaces are shaped by consumer needs (services), not by what providers happen to expose
- Every method takes `context.Context` as its first parameter
- Return domain types — never driver-specific types like `sql.Row` or `*redis.Result`
- Method names use **business language**, not technology language

## Interface Sizing

Design gateway interfaces to be as small as possible. A single-method interface is ideal; compose larger contracts from small ones. This enables fine-grained mocking and makes it clear what each consumer actually needs.

```go
// Good: focused, single-concern interfaces
type MeetingReader interface {
    GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error)
    ListByHost(ctx context.Context, hostID domain.UserID) ([]*domain.Meeting, error)
}

type MeetingWriter interface {
    Create(ctx context.Context, m *domain.Meeting) error
    Update(ctx context.Context, m *domain.Meeting) error
    Delete(ctx context.Context, id domain.MeetingID) error
}
```

A service that only reads meetings should accept `MeetingReader`, not a wider interface. This is the **Interface Segregation Principle** applied to Go:

```go
// ReportService only needs read access — accepts the narrow interface
func NewReportService(meetings gateway.MeetingReader) *ReportService {
    return &ReportService{meetings: meetings}
}
```

### Sizing Guidelines

| Interface Size | Use When | Example |
|---------------|----------|---------|
| 1 method | Single responsibility | `Notifier`, `IDGenerator` |
| 2-3 methods | Related CRUD subset | `MeetingReader`, `MeetingWriter` |
| 4+ methods | Composed from smaller interfaces | `MeetingRepository` (= Reader + Writer) |
| 5+ methods | Likely too large — split further | Re-evaluate if all methods share a single consumer |

## Gateway Naming Conventions

Gateway names should describe **what the core needs**, not the technology that fulfills it. This keeps the domain vocabulary clean and the architecture technology-agnostic.

| ❌ Technology-Oriented | ✅ Business-Oriented | Rationale |
|----------------------|---------------------|-----------|
| `PostgresMeetingStore` | `MeetingRepository` | The gateway doesn't know about Postgres |
| `RedisCache` | `MeetingCache` | The interface defines caching behavior, not Redis |
| `SMTPEmailClient` | `Notifier` | The domain needs to notify, not send SMTP |
| `S3FileUploader` | `AttachmentStore` | The domain stores attachments; S3 is an implementation detail |
| `KafkaEventPublisher` | `EventPublisher` | Publishing events is the capability; Kafka is the mechanism |

```go
// gateway/notification.go
type Notifier interface {
    MeetingScheduled(ctx context.Context, meeting *domain.Meeting) error
    MeetingCancelled(ctx context.Context, meeting *domain.Meeting) error
}

// gateway/storage.go
type AttachmentStore interface {
    Upload(ctx context.Context, meetingID domain.MeetingID, name string, data io.Reader) (string, error)
    Download(ctx context.Context, url string) (io.ReadCloser, error)
}
```

## Composition Over Inheritance

Compose larger interfaces from small ones when a consumer genuinely needs multiple capabilities:

```go
// Small, focused contracts
type MeetingReader interface {
    GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error)
    ListByHost(ctx context.Context, hostID domain.UserID) ([]*domain.Meeting, error)
}

type MeetingWriter interface {
    Create(ctx context.Context, m *domain.Meeting) error
    Update(ctx context.Context, m *domain.Meeting) error
    Delete(ctx context.Context, id domain.MeetingID) error
}

// Composed interface — used only when a consumer needs both read and write
type MeetingRepository interface {
    MeetingReader
    MeetingWriter
}
```

The composed `MeetingRepository` is useful at the wiring layer (`cmd/`) and for providers that implement both sides. But individual services should accept the narrowest interface they need.

## Transactor Pattern

When multiple gateway operations must succeed or fail atomically, define a `Transactor` interface in the gateway layer. This gives the service layer transaction control without exposing database internals:

```go
// gateway/transactor.go
type Transactor interface {
    // WithTransaction executes fn within a transaction.
    // If fn returns nil, the transaction is committed.
    // If fn returns an error, the transaction is rolled back.
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```

The provider implements this using `sql.Tx`, but the service never sees the transaction object:

```go
// provider/postgres/transactor.go
type PostgresTransactor struct {
    db *sql.DB
}

func (t *PostgresTransactor) WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := t.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }

    // Store tx in context so repository methods can retrieve it
    txCtx := context.WithValue(ctx, txKey, tx)

    if err := fn(txCtx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    return nil
}
```

Service usage:

```go
func (s *TransferService) Transfer(ctx context.Context, fromID, toID domain.AccountID, amount int) error {
    return s.tx.WithTransaction(ctx, func(txCtx context.Context) error {
        from, err := s.accounts.GetByID(txCtx, fromID)
        if err != nil {
            return fmt.Errorf("fetch source: %w", err)
        }
        if err := from.Debit(amount); err != nil {
            return fmt.Errorf("debit: %w", err)
        }
        to, err := s.accounts.GetByID(txCtx, toID)
        if err != nil {
            return fmt.Errorf("fetch destination: %w", err)
        }
        to.Credit(amount)

        if err := s.accounts.Update(txCtx, from); err != nil {
            return fmt.Errorf("persist debit: %w", err)
        }
        return s.accounts.Update(txCtx, to)
    })
}
```

## Mock Generation Strategy

Gateway interfaces are the primary mocking surface in wordloop-core. Mocks are hand-written structs with function fields — this keeps them simple, type-safe, and free from code generation dependencies.

```go
// gateway/mocks/meeting.go
type MockMeetingRepo struct {
    CreateFunc  func(ctx context.Context, m *domain.Meeting) error
    GetByIDFunc func(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error)
    UpdateFunc  func(ctx context.Context, m *domain.Meeting) error
    DeleteFunc  func(ctx context.Context, id domain.MeetingID) error
}

func (m *MockMeetingRepo) Create(ctx context.Context, meeting *domain.Meeting) error {
    return m.CreateFunc(ctx, meeting)
}

func (m *MockMeetingRepo) GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error) {
    return m.GetByIDFunc(ctx, id)
}
```

### Compile-Time Verification for Mocks

Always verify that mocks satisfy the gateway interface:

```go
var _ gateway.MeetingRepository = (*MockMeetingRepo)(nil)
```

### When to Mock

| Layer Under Test | What to Mock | Mock Location |
|-----------------|-------------|---------------|
| Service | Gateway interfaces | `internal/core/gateway/mocks/` |
| Entrypoint | Service interfaces | `internal/core/service/mocks/` |
| Domain | Nothing | Domain is pure — no dependencies |
| Provider | Nothing | Use testcontainers against real infrastructure |

## Gateway Placement and Package Layout

```
internal/core/gateway/
├── meeting.go         # MeetingReader, MeetingWriter, MeetingRepository
├── user.go            # UserReader, UserWriter, UserRepository
├── notification.go    # Notifier
├── storage.go         # AttachmentStore
├── transactor.go      # Transactor
└── mocks/
    ├── meeting.go     # MockMeetingRepo
    ├── user.go        # MockUserRepo
    ├── notification.go # MockNotifier
    └── storage.go     # MockAttachmentStore
```

All gateway interfaces live in a single `gateway` package. This keeps the import path simple and ensures that all outbound contracts are visible in one place. Mocks live in a `mocks/` subdirectory within `gateway/`.
