# Domain Modeling

## Table of Contents
- [Domain Purity](#domain-purity)
- [Entity Design](#entity-design)
- [Value Objects](#value-objects)
- [Constructors and Validation](#constructors-and-validation)
- [Sentinel Errors](#sentinel-errors)
- [Domain Methods](#domain-methods)
- [Functional Options for Domain Config](#functional-options-for-domain-config)

---

## Domain Purity

The domain layer is the heart of the wordloop-core service. It contains the business rules, entities, and invariants that define what the application does. The domain is the most stable and valuable part of the codebase because it has zero external dependencies — no database drivers, no HTTP frameworks, no third-party libraries.

This purity provides:
- **Independent testability** — Test thousands of cases in milliseconds without infrastructure
- **Framework independence** — Swap databases, HTTP routers, or deployment platforms without touching business logic
- **Readability** — Domain code reads like a description of the business, not a tour of framework APIs

### What Belongs in Domain

- Entity structs with business-relevant fields
- `New...` constructors that enforce invariants
- `Validate()` methods for complex validation rules
- Business-logic methods on entities (e.g., `meeting.IsUpcoming()`, `task.Complete()`)
- Sentinel errors for expected failure conditions
- Value objects and enumerations
- Pure computation functions

### What Does Not Belong in Domain

- JSON tags (those belong on entrypoint request/response types)
- SQL tags or database annotations
- Framework-specific decorators
- I/O operations (database queries, HTTP calls, file access)
- Logging calls
- Any import outside the Go standard library

## Entity Design

Entities represent the core business objects. Design them so they cannot exist in an invalid state once constructed.

### Always-Valid Types

Design types so that the zero value is either useful or construction is gated through a constructor that enforces validity:

```go
// MeetingID is a typed identifier — prevents accidentally passing a UserID where a MeetingID is expected
type MeetingID string

// Meeting is the core domain entity
type Meeting struct {
    id        MeetingID   // unexported to prevent direct assignment
    title     string
    hostID    UserID
    startsAt  time.Time
    duration  time.Duration
    status    MeetingStatus
    createdAt time.Time
}
```

Unexported fields force all construction through `NewMeeting()`, ensuring invariants are always satisfied.

### Accessor Methods

Provide accessor methods for fields that external layers need to read:

```go
func (m *Meeting) ID() MeetingID         { return m.id }
func (m *Meeting) Title() string          { return m.title }
func (m *Meeting) HostID() UserID         { return m.hostID }
func (m *Meeting) StartsAt() time.Time    { return m.startsAt }
func (m *Meeting) Duration() time.Duration { return m.duration }
func (m *Meeting) Status() MeetingStatus  { return m.status }
```

This pattern provides read access while preventing invalid mutations. When the provider layer scans from the database, it may use a separate `fromRow` function within its own package that constructs the entity with internal knowledge, or it can use a `Hydrate` function exposed for this purpose.

### Enumerations with Typed Constants

Use typed string or int constants for domain enumerations. This prevents invalid values and makes the code self-documenting:

```go
type MeetingStatus string

const (
    MeetingStatusScheduled  MeetingStatus = "scheduled"
    MeetingStatusInProgress MeetingStatus = "in_progress"
    MeetingStatusCompleted  MeetingStatus = "completed"
    MeetingStatusCancelled  MeetingStatus = "cancelled"
)

// IsTerminal reports whether this status represents a final state.
func (s MeetingStatus) IsTerminal() bool {
    return s == MeetingStatusCompleted || s == MeetingStatusCancelled
}
```

## Value Objects

Value objects are immutable types defined by their attributes rather than an identity. They are compared by value, not by reference.

```go
// TimeRange represents a bounded period of time.
type TimeRange struct {
    start time.Time
    end   time.Time
}

// NewTimeRange creates a validated time range.
func NewTimeRange(start, end time.Time) (TimeRange, error) {
    if end.Before(start) {
        return TimeRange{}, fmt.Errorf("end time %v is before start time %v: %w", end, start, ErrInvalidTimeRange)
    }
    return TimeRange{start: start, end: end}, nil
}

func (tr TimeRange) Duration() time.Duration { return tr.end.Sub(tr.start) }
func (tr TimeRange) Contains(t time.Time) bool {
    return !t.Before(tr.start) && !t.After(tr.end)
}
func (tr TimeRange) Overlaps(other TimeRange) bool {
    return tr.start.Before(other.end) && other.start.Before(tr.end)
}
```

## Constructors and Validation

Every entity requires a constructor. Constructors enforce invariants at creation time so that no code in the service or provider layers needs to worry about invalid entities.

### Basic Constructor Pattern

```go
// NewMeeting creates a valid Meeting entity.
// It returns an error if the provided data violates business rules.
func NewMeeting(title string, hostID UserID, startsAt time.Time, duration time.Duration) (*Meeting, error) {
    if title == "" {
        return nil, fmt.Errorf("meeting title: %w", ErrRequired)
    }
    if len(title) > 255 {
        return nil, fmt.Errorf("meeting title exceeds 255 characters: %w", ErrInvalidInput)
    }
    if hostID == "" {
        return nil, fmt.Errorf("meeting host ID: %w", ErrRequired)
    }
    if duration < 5*time.Minute {
        return nil, fmt.Errorf("meeting duration must be at least 5 minutes: %w", ErrInvalidInput)
    }

    return &Meeting{
        id:        MeetingID(uuid.NewString()),
        title:     title,
        hostID:    hostID,
        startsAt:  startsAt,
        duration:  duration,
        status:    MeetingStatusScheduled,
        createdAt: time.Now(),
    }, nil
}
```

### Complex Construction with Functional Options

When entities have many optional configuration parameters, use the functional options pattern to keep constructors clean:

```go
type MeetingOption func(*Meeting)

func WithRecordingEnabled(enabled bool) MeetingOption {
    return func(m *Meeting) {
        m.recordingEnabled = enabled
    }
}

func WithMaxParticipants(max int) MeetingOption {
    return func(m *Meeting) {
        m.maxParticipants = max
    }
}

func NewMeeting(title string, hostID UserID, startsAt time.Time, duration time.Duration, opts ...MeetingOption) (*Meeting, error) {
    m := &Meeting{
        id:               MeetingID(uuid.NewString()),
        title:            title,
        hostID:           hostID,
        startsAt:         startsAt,
        duration:         duration,
        status:           MeetingStatusScheduled,
        maxParticipants:  50, // sensible default
        createdAt:        time.Now(),
    }

    for _, opt := range opts {
        opt(m)
    }

    if err := m.validate(); err != nil {
        return nil, err
    }
    return m, nil
}

func (m *Meeting) validate() error {
    if m.title == "" {
        return fmt.Errorf("meeting title: %w", ErrRequired)
    }
    if m.maxParticipants < 1 {
        return fmt.Errorf("max participants must be positive: %w", ErrInvalidInput)
    }
    return nil
}
```

## Sentinel Errors

Define expected error conditions as sentinel errors in a dedicated `errors.go` file within the domain package. These errors cross layer boundaries and enable services and entrypoints to make decisions without importing infrastructure-specific error types.

```go
// errors.go
package domain

import "errors"

// Sentinel errors for expected business conditions.
// Providers wrap driver-specific errors into these sentinels using %w.
// Entrypoints map these to HTTP status codes.
var (
    // ErrNotFound is returned when a requested entity does not exist.
    ErrNotFound = errors.New("not found")

    // ErrConflict is returned when an operation violates a uniqueness constraint.
    ErrConflict = errors.New("conflict")

    // ErrRequired is returned when a required field is missing or empty.
    ErrRequired = errors.New("required")

    // ErrInvalidInput is returned when input fails validation.
    ErrInvalidInput = errors.New("invalid input")

    // ErrForbidden is returned when the caller lacks permission for the operation.
    ErrForbidden = errors.New("forbidden")

    // ErrExpired is returned when an entity or token has passed its validity window.
    ErrExpired = errors.New("expired")
)
```

### Error Flow Across Layers

```
Provider:   sql.ErrNoRows → fmt.Errorf("meeting %s: %w", id, domain.ErrNotFound)
Service:    wraps with context → fmt.Errorf("get meeting: %w", err)
Entrypoint: errors.Is(err, domain.ErrNotFound) → 404 Not Found
            (logs detailed error, returns sanitized message to client)
```

### Type-Safe Error Extraction

For custom error types that carry additional context, use `errors.AsType` for compile-time type safety:

```go
// Domain — custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Entrypoint — extract with type safety
if ve, ok := errors.AsType[*ValidationError](err); ok {
    writeJSON(w, http.StatusUnprocessableEntity, map[string]string{
        "field":   ve.Field,
        "message": ve.Message,
    })
    return
}
```

## Domain Methods

Encapsulate behavior on the entity that only requires data within that entity. If a method only reads fields from its own struct, it belongs as a method on that struct.

```go
// IsUpcoming reports whether the meeting has not yet started.
func (m *Meeting) IsUpcoming() bool {
    return m.status == MeetingStatusScheduled && time.Now().Before(m.startsAt)
}

// Cancel transitions the meeting to cancelled status.
// It returns an error if the meeting is already in a terminal state.
func (m *Meeting) Cancel() error {
    if m.status.IsTerminal() {
        return fmt.Errorf("cannot cancel meeting in %s state: %w", m.status, ErrInvalidInput)
    }
    m.status = MeetingStatusCancelled
    return nil
}

// Start transitions the meeting to in-progress.
func (m *Meeting) Start() error {
    if m.status != MeetingStatusScheduled {
        return fmt.Errorf("can only start a scheduled meeting, current: %s: %w", m.status, ErrInvalidInput)
    }
    m.status = MeetingStatusInProgress
    return nil
}

// EndTime returns the computed end time of the meeting.
func (m *Meeting) EndTime() time.Time {
    return m.startsAt.Add(m.duration)
}
```

### Methods That Do Not Belong on Domain

Methods should not:
- Make database queries — that is a provider concern
- Send notifications — define a gateway interface for that
- Log anything — the domain is silent, pure logic
- Access request headers or HTTP concepts — those are entrypoint concerns

If you need data from outside the entity to make a decision, the service layer should fetch it and pass it to the domain method as a parameter.

## Functional Options for Domain Config

When you need to configure aspects of domain behavior, prefer functional options over config structs with many fields. This pattern provides clear defaults, self-documenting call sites, and easy extensibility:

```go
type TaskOption func(*Task)

func WithPriority(p Priority) TaskOption {
    return func(t *Task) { t.priority = p }
}

func WithDueDate(d time.Time) TaskOption {
    return func(t *Task) { t.dueDate = &d }
}

func WithAssignee(userID UserID) TaskOption {
    return func(t *Task) { t.assigneeID = &userID }
}

// Call site is self-documenting:
// task, err := domain.NewTask("Review PR", projectID,
//     domain.WithPriority(domain.PriorityHigh),
//     domain.WithDueDate(deadline),
//     domain.WithAssignee(reviewerID),
// )
```
