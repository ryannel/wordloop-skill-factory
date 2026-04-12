# Testing Strategy

## Table of Contents
- [Testing Philosophy](#testing-philosophy)
- [Layer-Specific Testing](#layer-specific-testing)
- [Table-Driven Tests](#table-driven-tests)
- [Mocking with Interfaces](#mocking-with-interfaces)
- [Integration Tests with Testcontainers](#integration-tests-with-testcontainers)
- [Fuzz Testing](#fuzz-testing)
- [Contract Testing](#contract-testing)
- [Race Detection](#race-detection)
- [Benchmarking](#benchmarking)
- [Test Commands Reference](#test-commands-reference)

---

## Testing Philosophy

Testing in wordloop-core follows the architecture's layer boundaries. Each layer has a specific testing strategy that matches its responsibility and dependency profile. The principle is simple: test the layer's logic in isolation, mocking only the layer directly below it.

Tests are updated with every code change. This is not optional — untested code is considered unfinished.

## Layer-Specific Testing

| Layer | Test Type | Dependencies | Strategy |
|-------|-----------|-------------|----------|
| **Domain** | Unit tests | None | Test pure business logic directly — constructors, validation, methods |
| **Service** | Unit tests with mocks | `gateway/mocks/` | Test orchestration logic using mock gateway implementations |
| **Provider** | Integration tests | Real Postgres (testcontainers) | Test SQL queries and error translation against a real database |
| **Entrypoint** | HTTP tests | `service/mocks/` | Test HTTP handlers using `httptest` with mock services |

### Domain Tests

Domain tests are the fastest — no mocks, no infrastructure. Test constructors, validation, state transitions, and business methods:

```go
func TestNewMeeting(t *testing.T) {
    tests := []struct {
        name     string
        title    string
        hostID   domain.UserID
        startsAt time.Time
        duration time.Duration
        wantErr  error
    }{
        {
            name:     "valid meeting",
            title:    "Sprint Planning",
            hostID:   "user-123",
            startsAt: time.Now().Add(time.Hour),
            duration: 30 * time.Minute,
            wantErr:  nil,
        },
        {
            name:     "empty title",
            title:    "",
            hostID:   "user-123",
            startsAt: time.Now().Add(time.Hour),
            duration: 30 * time.Minute,
            wantErr:  domain.ErrRequired,
        },
        {
            name:     "duration too short",
            title:    "Quick Sync",
            hostID:   "user-123",
            startsAt: time.Now().Add(time.Hour),
            duration: 2 * time.Minute,
            wantErr:  domain.ErrInvalidInput,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            meeting, err := domain.NewMeeting(tt.title, tt.hostID, tt.startsAt, tt.duration)
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("NewMeeting() error = %v, want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("NewMeeting() unexpected error: %v", err)
            }
            if meeting.Title() != tt.title {
                t.Errorf("Title() = %q, want %q", meeting.Title(), tt.title)
            }
            if meeting.Status() != domain.MeetingStatusScheduled {
                t.Errorf("Status() = %q, want %q", meeting.Status(), domain.MeetingStatusScheduled)
            }
        })
    }
}

func TestMeeting_Cancel(t *testing.T) {
    tests := []struct {
        name      string
        status    domain.MeetingStatus
        wantErr   bool
    }{
        {"cancel scheduled meeting", domain.MeetingStatusScheduled, false},
        {"cancel in-progress meeting", domain.MeetingStatusInProgress, false},
        {"cancel completed meeting", domain.MeetingStatusCompleted, true},
        {"cancel already cancelled meeting", domain.MeetingStatusCancelled, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            meeting := testMeetingWithStatus(t, tt.status)
            err := meeting.Cancel()
            if (err != nil) != tt.wantErr {
                t.Errorf("Cancel() error = %v, wantErr = %v", err, tt.wantErr)
            }
        })
    }
}
```

### Service Tests

Service tests verify orchestration logic using gateway mocks:

```go
func TestMeetingService_ScheduleMeeting(t *testing.T) {
    tests := []struct {
        name        string
        req         service.ScheduleMeetingRequest
        setupMock   func(*mocks.MockMeetingRepo, *mocks.MockUserRepo)
        wantErr     error
    }{
        {
            name: "successful scheduling",
            req: service.ScheduleMeetingRequest{
                Title:    "Sprint Planning",
                HostID:   "user-123",
                StartsAt: time.Now().Add(time.Hour),
                Duration: 30 * time.Minute,
            },
            setupMock: func(meetings *mocks.MockMeetingRepo, users *mocks.MockUserRepo) {
                users.GetByIDFunc = func(ctx context.Context, id domain.UserID) (*domain.User, error) {
                    return testUser(id), nil
                }
                meetings.CreateFunc = func(ctx context.Context, m *domain.Meeting) error {
                    return nil
                }
            },
            wantErr: nil,
        },
        {
            name: "host not found",
            req: service.ScheduleMeetingRequest{
                Title:  "Sprint Planning",
                HostID: "unknown-user",
            },
            setupMock: func(meetings *mocks.MockMeetingRepo, users *mocks.MockUserRepo) {
                users.GetByIDFunc = func(ctx context.Context, id domain.UserID) (*domain.User, error) {
                    return nil, fmt.Errorf("user %s: %w", id, domain.ErrNotFound)
                }
            },
            wantErr: domain.ErrNotFound,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            meetingRepo := &mocks.MockMeetingRepo{}
            userRepo := &mocks.MockUserRepo{}
            tt.setupMock(meetingRepo, userRepo)

            svc := service.NewMeetingService(meetingRepo, userRepo, &mocks.NoOpNotifier{}, slog.Default())

            result, err := svc.ScheduleMeeting(context.Background(), tt.req)
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("ScheduleMeeting() error = %v, want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("ScheduleMeeting() unexpected error: %v", err)
            }
            if result.Title() != tt.req.Title {
                t.Errorf("Title() = %q, want %q", result.Title(), tt.req.Title)
            }
        })
    }
}
```

### Entrypoint Tests

Entrypoint tests use `httptest` to verify HTTP behavior with mock services:

```go
func TestMeetingHandler_GetMeeting(t *testing.T) {
    tests := []struct {
        name       string
        meetingID  string
        setupMock  func(*mocks.MockMeetingService)
        wantStatus int
    }{
        {
            name:      "existing meeting",
            meetingID: "meeting-123",
            setupMock: func(svc *mocks.MockMeetingService) {
                svc.GetFunc = func(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error) {
                    return testMeeting(), nil
                }
            },
            wantStatus: http.StatusOK,
        },
        {
            name:      "missing meeting",
            meetingID: "meeting-unknown",
            setupMock: func(svc *mocks.MockMeetingService) {
                svc.GetFunc = func(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error) {
                    return nil, fmt.Errorf("fetch: %w", domain.ErrNotFound)
                }
            },
            wantStatus: http.StatusNotFound,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := &mocks.MockMeetingService{}
            tt.setupMock(svc)

            handler := routes.NewMeetingHandler(svc)
            
            req := httptest.NewRequest(http.MethodGet, "/meetings/"+tt.meetingID, nil)
            rec := httptest.NewRecorder()
            
            handler.GetMeeting(rec, req)
            
            if rec.Code != tt.wantStatus {
                t.Errorf("status = %d, want %d", rec.Code, tt.wantStatus)
            }
        })
    }
}
```

## Table-Driven Tests

All tests in wordloop-core use table-driven formatting with subtests. This pattern provides:

- Clear test case documentation through descriptive names
- Easy addition of new cases
- Parallel execution capability via `t.Parallel()`
- Consistent structure across the codebase

### Structure

```go
func TestSubject(t *testing.T) {
    tests := []struct {
        name     string        // Descriptive test case name
        // ... inputs
        wantErr  error         // Expected error (nil for success)
        // ... expected outputs
    }{
        // Test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Arrange, Act, Assert
        })
    }
}
```

### Parallel Tests

For tests that don't share state:

```go
for _, tt := range tests {
    tt := tt // capture range variable
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

## Mocking with Interfaces

Mocks are hand-written structs that implement gateway or service interfaces using function fields. This keeps them simple and type-safe:

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

func (m *MockMeetingRepo) Update(ctx context.Context, meeting *domain.Meeting) error {
    return m.UpdateFunc(ctx, meeting)
}

func (m *MockMeetingRepo) Delete(ctx context.Context, id domain.MeetingID) error {
    return m.DeleteFunc(ctx, id)
}
```

### Mock Placement

- `internal/core/gateway/mocks/` — Mocks of gateway interfaces, used by service tests
- `internal/core/service/mocks/` — Mocks of service interfaces, used by entrypoint tests

### Test Helpers

Mark helper functions with `t.Helper()` to improve error output:

```go
func testMeeting(t *testing.T) *domain.Meeting {
    t.Helper()
    m, err := domain.NewMeeting("Test Meeting", "user-123", time.Now().Add(time.Hour), 30*time.Minute)
    if err != nil {
        t.Fatalf("failed to create test meeting: %v", err)
    }
    return m
}
```

## Integration Tests with Testcontainers

Provider-layer tests run against real Postgres instances using testcontainers-go. This catches SQL syntax errors, constraint violations, and version-specific behavior that mocks cannot replicate.

### Container Setup

```go
// provider/postgres/testdb_setup_test.go
package postgres_test

import (
    "context"
    "database/sql"
    "testing"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
    ctx := context.Background()

    container, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    if err != nil {
        panic(err)
    }
    defer container.Terminate(ctx)

    connStr, err := container.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        panic(err)
    }

    testDB, err = sql.Open("postgres", connStr)
    if err != nil {
        panic(err)
    }

    // Run migrations
    if err := runMigrations(testDB); err != nil {
        panic(err)
    }

    os.Exit(m.Run())
}
```

### Provider Integration Test

```go
func TestPostgresMeetingRepo_Create(t *testing.T) {
    repo := postgres.NewMeetingRepository(testDB)
    ctx := context.Background()

    meeting, err := domain.NewMeeting("Integration Test Meeting", "user-1", time.Now().Add(time.Hour), 30*time.Minute)
    if err != nil {
        t.Fatalf("create test meeting: %v", err)
    }

    // Test Create
    if err := repo.Create(ctx, meeting); err != nil {
        t.Fatalf("Create() error: %v", err)
    }

    // Test GetByID
    fetched, err := repo.GetByID(ctx, meeting.ID())
    if err != nil {
        t.Fatalf("GetByID() error: %v", err)
    }
    if fetched.Title() != meeting.Title() {
        t.Errorf("Title() = %q, want %q", fetched.Title(), meeting.Title())
    }
}

func TestPostgresMeetingRepo_GetByID_NotFound(t *testing.T) {
    repo := postgres.NewMeetingRepository(testDB)
    ctx := context.Background()

    _, err := repo.GetByID(ctx, "nonexistent-id")
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("GetByID() error = %v, want %v", err, domain.ErrNotFound)
    }
}
```

### Test Isolation

Reuse the container across tests but reset state between tests using table truncation:

```go
func cleanupTables(t *testing.T, db *sql.DB) {
    t.Helper()
    _, err := db.Exec("TRUNCATE meetings, users CASCADE")
    if err != nil {
        t.Fatalf("cleanup: %v", err)
    }
}
```

## Fuzz Testing

Use native Go fuzzing on trust boundaries — HTTP handlers, parsers, and deserialization logic:

```go
func FuzzParseMeetingDuration(f *testing.F) {
    // Seed corpus with representative inputs
    f.Add("30m")
    f.Add("1h30m")
    f.Add("2h")
    f.Add("0s")
    f.Add("")
    f.Add("invalid")

    f.Fuzz(func(t *testing.T, input string) {
        duration, err := parseMeetingDuration(input)
        if err != nil {
            return // Invalid input is fine — we just shouldn't panic
        }
        if duration < 0 {
            t.Errorf("parseMeetingDuration(%q) = %v, negative duration", input, duration)
        }
    })
}
```

### Fuzzing Best Practices

- **Seed corpus** — Provide representative inputs to guide the fuzzer
- **Time-bound in CI** — Use `-fuzztime` to prevent CI from running indefinitely
- **Commit crash cases** — When the fuzzer finds a failure, it generates a test case in `testdata/`; commit it as a regression test
- **Focus on trust boundaries** — Fuzz inputs that cross a trust boundary (user input, external API responses)

## Contract Testing

Contract tests verify that entrypoint implementations conform to their OpenAPI or AsyncAPI specifications. This is distinct from unit tests (which test logic) and integration tests (which test infrastructure) — contract tests catch **spec drift** between what the API documentation promises and what the handlers actually produce.

### Response Schema Validation

Validate that handler responses match the declared OpenAPI schema:

```go
func TestMeetingEndpoints_ConformToSpec(t *testing.T) {
    // Load the OpenAPI spec
    spec, err := openapi3.NewLoader().LoadFromFile("../../../api/openapi.yaml")
    if err != nil {
        t.Fatalf("load spec: %v", err)
    }

    router, err := gorillamux.NewRouter(spec)
    if err != nil {
        t.Fatalf("create router: %v", err)
    }

    tests := []struct {
        name   string
        method string
        path   string
        body   string
    }{
        {"list meetings", http.MethodGet, "/v1/meetings", ""},
        {"create meeting", http.MethodPost, "/v1/meetings", `{"title":"Test","host_id":"user-1"}`},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            var bodyReader io.Reader
            if tt.body != "" {
                bodyReader = strings.NewReader(tt.body)
            }

            req := httptest.NewRequest(tt.method, tt.path, bodyReader)
            req.Header.Set("Content-Type", "application/json")
            rec := httptest.NewRecorder()

            handler.ServeHTTP(rec, req)

            // Validate the response against the OpenAPI spec
            input := &openapi3filter.RequestValidationInput{
                Request:    req,
                PathParams: extractPathParams(router, req),
                Route:      findRoute(router, req),
            }
            responseInput := &openapi3filter.ResponseValidationInput{
                RequestValidationInput: input,
                Status:                 rec.Code,
                Header:                 rec.Header(),
                Body:                   io.NopCloser(rec.Body),
            }

            if err := openapi3filter.ValidateResponse(context.Background(), responseInput); err != nil {
                t.Errorf("response does not match OpenAPI spec: %v", err)
            }
        })
    }
}
```

### What Contract Tests Catch

| Issue | Example | Without Contract Tests |
|-------|---------|----------------------|
| **Missing fields** | Spec says `created_at` is required, handler omits it | Silent API breakage |
| **Wrong types** | Spec says `status` is an enum, handler returns free-form string | Client deserialization failure |
| **Undocumented endpoints** | Handler exists but spec doesn't describe it | Invisible API surface |
| **Status code mismatch** | Spec says 201 for create, handler returns 200 | Client error handling breaks |

Run contract tests in CI alongside unit and integration tests. Spec drift caught in CI is far cheaper than spec drift caught in production.

## Race Detection

Always run tests with `-race`:

```bash
go test -race ./...
```

The race detector identifies data races at runtime. In CI, every test run must pass with `-race`. If a race is detected, it must be fixed before merging — races are latent bugs that cause unpredictable production failures.

### Common Race Patterns to Avoid

```go
// RACE — shared variable modified concurrently without synchronization
var counter int
go func() { counter++ }()

// FIXED — use sync.Mutex or atomic operations
var counter atomic.Int64
go func() { counter.Add(1) }()
```

## Benchmarking

Write benchmarks for performance-critical paths:

```go
func BenchmarkMeetingValidation(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _, _ = domain.NewMeeting("Bench Meeting", "user-1", time.Now().Add(time.Hour), 30*time.Minute)
    }
}

func BenchmarkMeetingRepo_GetByID(b *testing.B) {
    repo := postgres.NewMeetingRepository(testDB)
    ctx := context.Background()

    // Setup
    meeting := createTestMeeting(b)
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        _, _ = repo.GetByID(ctx, meeting.ID())
    }
}
```

Use `b.ReportAllocs()` to track allocations and `b.RunParallel` for concurrent benchmarks.

## Test Commands Reference

| Command | Description |
|---------|-------------|
| `go test ./...` | Run all tests |
| `go test -v ./...` | Verbose output |
| `go test -race ./...` | Run with race detector |
| `go test -run TestName ./...` | Run specific test |
| `go test -cover ./...` | Show coverage summary |
| `go test -coverprofile=coverage.out ./...` | Generate coverage profile |
| `go tool cover -html=coverage.out` | View HTML coverage report |
| `go test -bench . ./...` | Run benchmarks |
| `go test -fuzz FuzzName -fuzztime 30s` | Run fuzzer for 30 seconds |
| `go test -short ./...` | Skip long-running tests |
