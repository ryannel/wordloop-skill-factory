# Service Layer

## Table of Contents
- [Service Design](#service-design)
- [Constructor Injection](#constructor-injection)
- [Service Method Pattern](#service-method-pattern)
- [Service-Level Request Types](#service-level-request-types)
- [Transaction Boundaries](#transaction-boundaries)

---

## Service Design

Services are the orchestrators of business intent. They sit between the domain and the outside world, coordinating workflows by calling domain methods and gateway interfaces. Services contain no technology-specific code — no SQL, no HTTP semantics, no framework references.

### Service Anatomy

```go
// MeetingService orchestrates meeting-related use cases.
type MeetingService struct {
    meetings gateway.MeetingRepository
    users    gateway.UserRepository
    notifier gateway.Notifier
    logger   *slog.Logger
}

// NewMeetingService constructs a MeetingService with its required dependencies.
// Accept interfaces, return the concrete type.
func NewMeetingService(
    meetings gateway.MeetingRepository,
    users gateway.UserRepository,
    notifier gateway.Notifier,
    logger *slog.Logger,
) *MeetingService {
    return &MeetingService{
        meetings: meetings,
        users:    users,
        notifier: notifier,
        logger:   logger,
    }
}
```

Key properties:
- Dependencies are gateway **interfaces**, not concrete providers
- The constructor accepts interfaces and returns a concrete `*MeetingService`
- A structured logger is injected — never use package-level loggers
- `context.Context` is the first parameter of every public method, but is not stored on the struct

## Constructor Injection

Every struct in wordloop-core uses constructor injection. This is the "gold thread" of the architecture:

```go
// Accept interfaces
func NewMeetingService(repo gateway.MeetingRepository) *MeetingService {
    return &MeetingService{meetings: repo}
}

// Return concrete types
func NewPostgresMeetingRepo(db *sql.DB) *PostgresMeetingRepo {
    return &PostgresMeetingRepo{db: db}
}
```

Rules:
- **Accept interfaces** — constructor parameters are abstract contracts
- **Return concrete types** — constructors return `*ConcreteType`, not an interface
- **No globals** — database pools, loggers, and configuration are all injected at `cmd/`
- **No `init()`** — all wiring happens explicitly at the application entry point

## Service Method Pattern

```go
// ScheduleMeeting creates a new meeting and notifies the host.
func (s *MeetingService) ScheduleMeeting(ctx context.Context, req ScheduleMeetingRequest) (*domain.Meeting, error) {
    // 1. Validate the actor exists
    host, err := s.users.GetByID(ctx, req.HostID)
    if err != nil {
        return nil, fmt.Errorf("verify host %s: %w", req.HostID, err)
    }

    // 2. Construct the domain entity (validation happens here)
    meeting, err := domain.NewMeeting(req.Title, host.ID(), req.StartsAt, req.Duration)
    if err != nil {
        return nil, fmt.Errorf("create meeting: %w", err)
    }

    // 3. Persist through the gateway
    if err := s.meetings.Create(ctx, meeting); err != nil {
        return nil, fmt.Errorf("save meeting: %w", err)
    }

    // 4. Fire side effects (best effort — don't fail the operation)
    if err := s.notifier.MeetingScheduled(ctx, meeting); err != nil {
        slog.WarnContext(ctx, "failed to send meeting notification",
            slog.String("meeting.id", meeting.ID().String()),
            slog.String("error", err.Error()),
        )
    }

    return meeting, nil
}
```

## Service-Level Request Types

Define request structs to group parameters for service methods. These live in the service package and are distinct from both domain entities and HTTP request bodies:

```go
// ScheduleMeetingRequest contains the data needed to schedule a meeting.
type ScheduleMeetingRequest struct {
    Title    string
    HostID   domain.UserID
    StartsAt time.Time
    Duration time.Duration
}
```

The entrypoint layer translates HTTP requests into these service-level request types. This keeps the service independent of transport concerns.

## Transaction Boundaries

The service layer defines transaction boundaries. Providers execute within them. There are two approaches, depending on complexity:

### Simple: Per-Operation Transactions

For straightforward CRUD, providers handle their own transactions internally:

```go
func (s *MeetingService) Cancel(ctx context.Context, id domain.MeetingID) error {
    meeting, err := s.meetings.GetByID(ctx, id)
    if err != nil {
        return fmt.Errorf("fetch meeting: %w", err)
    }

    if err := meeting.Cancel(); err != nil {
        return fmt.Errorf("cancel meeting: %w", err)
    }

    if err := s.meetings.Update(ctx, meeting); err != nil {
        return fmt.Errorf("persist cancelled meeting: %w", err)
    }
    return nil
}
```

### Complex: Service-Managed Transactions

When multiple gateway operations must succeed or fail together, the service manages the transaction through a gateway-defined abstraction:

```go
// In gateway package — define a transactional wrapper
type Transactor interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}

// In service — use the transactor
func (s *TransferService) Transfer(ctx context.Context, fromID, toID domain.AccountID, amount int) error {
    return s.tx.WithTransaction(ctx, func(txCtx context.Context) error {
        from, err := s.accounts.GetByID(txCtx, fromID)
        if err != nil {
            return fmt.Errorf("fetch source account: %w", err)
        }

        if err := from.Debit(amount); err != nil {
            return fmt.Errorf("debit: %w", err)
        }

        to, err := s.accounts.GetByID(txCtx, toID)
        if err != nil {
            return fmt.Errorf("fetch destination account: %w", err)
        }

        to.Credit(amount)

        if err := s.accounts.Update(txCtx, from); err != nil {
            return fmt.Errorf("persist debit: %w", err)
        }
        if err := s.accounts.Update(txCtx, to); err != nil {
            return fmt.Errorf("persist credit: %w", err)
        }
        return nil
    })
}
```

The `Transactor` is a gateway interface, implemented by the provider using `sql.Tx`. The service never sees `*sql.Tx`.
