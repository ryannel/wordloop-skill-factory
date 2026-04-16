# Test Data Management

This module defines patterns for creating, seeding, isolating, and resetting test data across all test types. Effective test data management is the difference between a suite that runs reliably in parallel and one that suffers from mysterious, environment-dependent failures.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Database Reset Strategies](#database-reset-strategies)
3. [Fixture Patterns](#fixture-patterns)
4. [Factory Functions](#factory-functions)
5. [Referential Integrity](#referential-integrity)
6. [Test Data Isolation](#test-data-isolation)
7. [Seeding Strategies by Test Type](#seeding-strategies-by-test-type)

---

## Core Principle

Test data is a **first-class architectural concern**, not an afterthought. Every test must control its own data lifecycle:

- **Create** exactly the data it needs
- **Isolate** that data from all other tests
- **Verify** against that data
- **Clean up** after itself (or rely on infrastructure-level reset)

The golden rule: **no test should ever depend on data created by another test.** Shared mutable state is the #1 cause of flaky tests.

---

## Database Reset Strategies

Three strategies exist for ensuring a clean database between tests. Each has trade-offs:

### Strategy 1: Container Recreation (Recommended Default)

Destroy and recreate the entire database container between test suites.

```go
func TestMain(m *testing.M) {
    // Start fresh container per suite
    container := startPostgresContainer()
    runMigrations(container.DSN)
    
    code := m.Run()
    
    // Container destroyed automatically on exit
    container.Terminate(ctx)
    os.Exit(code)
}
```

| Pros | Cons |
|------|------|
| Absolute isolation — zero residual state | Slower startup (~2-5s per suite) |
| Migrations run every time — validates schema | Cannot share containers between suites |
| Simple — no cleanup logic needed | |

**Use when:** Service Tests where you need full confidence in schema correctness.

### Strategy 2: Transaction Rollback

Wrap each test in a database transaction and roll back after assertion.

```go
func setupTest(t *testing.T, db *sql.DB) *sql.Tx {
    tx, err := db.Begin()
    require.NoError(t, err)
    
    t.Cleanup(func() {
        tx.Rollback() // Undo everything this test did
    })
    
    return tx
}
```

| Pros | Cons |
|------|------|
| Very fast — no container restart | Cannot test commit behavior or triggers |
| Perfect isolation between tests | Requires passing `tx` instead of `db` through all layers |
| Works with shared container instances | Some ORM features behave differently inside transactions |

**Use when:** You need fast iteration within a single suite and can accept the transaction boundary constraint.

### Strategy 3: Truncation

Delete all data from tables between tests but keep the schema intact.

```go
func truncateAll(t *testing.T, db *sql.DB) {
    t.Helper()
    tables := []string{"tasks", "meetings", "users"}
    for _, table := range tables {
        _, err := db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", table))
        require.NoError(t, err)
    }
}
```

| Pros | Cons |
|------|------|
| Faster than container recreation | Must know all tables and truncation order |
| Schema persists — no migration cost | Foreign key constraints can make CASCADE complex |
| Shared container across entire suite | Residual sequences (auto-increment IDs) persist |

**Use when:** System Tests where container restart is too slow but you need clean tables between scenarios.

### Decision Matrix

| Test Type | Recommended Reset Strategy |
|-----------|--------------------------|
| **Service Test** | Container Recreation (singleton per suite) or Transaction Rollback |
| **System Test** | Truncation between scenarios, container recreation per suite |
| **Live Service Test** | Transaction Rollback (minimize side effects on shared dev databases) |
| **Live System Test** | Truncation + explicit cleanup of external state |

---

## Fixture Patterns

### The Builder Pattern

Use builders to create test data with sensible defaults and selective overrides:

```go
type MeetingBuilder struct {
    id     string
    title  string
    status string
    userID string
}

func NewMeetingBuilder() *MeetingBuilder {
    return &MeetingBuilder{
        id:     uuid.New().String(),
        title:  "Test Meeting",
        status: "active",
        userID: "user_default",
    }
}

func (b *MeetingBuilder) WithTitle(title string) *MeetingBuilder {
    b.title = title
    return b
}

func (b *MeetingBuilder) WithStatus(status string) *MeetingBuilder {
    b.status = status
    return b
}

func (b *MeetingBuilder) Build(t *testing.T, db *sql.DB) Meeting {
    t.Helper()
    meeting := Meeting{ID: b.id, Title: b.title, Status: b.status, UserID: b.userID}
    _, err := db.Exec(
        "INSERT INTO meetings (id, title, status, user_id) VALUES ($1, $2, $3, $4)",
        meeting.ID, meeting.Title, meeting.Status, meeting.UserID,
    )
    require.NoError(t, err)
    return meeting
}
```

**Usage:**
```go
func TestGetMeeting_ShouldReturn200_WhenMeetingExists(t *testing.T) {
    meeting := NewMeetingBuilder().
        WithTitle("Team Standup").
        WithStatus("active").
        Build(t, db)
    
    resp := httpGet(t, "/api/v1/meetings/" + meeting.ID)
    assert.Equal(t, 200, resp.StatusCode)
}
```

### Python: Fixture Factories with pytest

```python
import pytest
from uuid import uuid4

@pytest.fixture
def meeting_factory(db_session):
    """Factory fixture that creates meetings with sensible defaults."""
    created = []
    
    def _create(title="Test Meeting", status="active", user_id="user_default"):
        meeting = Meeting(
            id=str(uuid4()),
            title=title,
            status=status,
            user_id=user_id,
        )
        db_session.add(meeting)
        db_session.flush()
        created.append(meeting)
        return meeting
    
    yield _create
    
    # Cleanup
    for m in created:
        db_session.delete(m)
    db_session.flush()

def test_get_meeting_should_return_200_when_meeting_exists(client, meeting_factory):
    meeting = meeting_factory(title="Team Standup")
    response = client.get(f"/api/v1/meetings/{meeting.id}")
    assert response.status_code == 200
```

---

## Factory Functions

### Why Factories Over Raw SQL

| Approach | Problem |
|----------|---------|
| **Raw SQL fixtures** | Tightly coupled to schema. Every migration requires updating every fixture. No type safety. |
| **JSON/YAML seed files** | Same coupling problem. Additionally, hard to generate unique IDs and handle referential integrity dynamically. |
| **ORM model creation** | Better, but still verbose when you need sensible defaults and selective overrides. |
| **Factory functions** | Best. Encapsulate defaults, handle ID generation, provide fluent override API, and maintain type safety. |

### Factory Design Rules

1. **Sensible defaults for everything.** A factory called with no arguments must produce a valid, insertable entity.
2. **Override any field.** The caller must be able to customize any field without knowing about the others.
3. **Unique IDs by default.** Every factory call produces a unique entity. No hardcoded IDs.
4. **Handle foreign keys.** If a meeting requires a user, the factory should auto-create one or accept an explicit override.
5. **Return the created entity.** The caller needs the generated ID and other auto-populated fields for assertions.

---

## Referential Integrity

### The Problem

Real databases have foreign key constraints. A meeting requires a user. A task requires a meeting. A transcription requires a task. Creating test data means building the entire dependency chain.

### The Solution: Cascading Factories

```go
func NewTaskBuilder() *TaskBuilder {
    return &TaskBuilder{
        id:        uuid.New().String(),
        meetingID: "", // Will be auto-created if not set
        status:    "pending",
    }
}

func (b *TaskBuilder) Build(t *testing.T, db *sql.DB) Task {
    t.Helper()
    
    // Auto-create meeting dependency if not explicitly set
    if b.meetingID == "" {
        meeting := NewMeetingBuilder().Build(t, db)
        b.meetingID = meeting.ID
    }
    
    task := Task{ID: b.id, MeetingID: b.meetingID, Status: b.status}
    _, err := db.Exec(
        "INSERT INTO tasks (id, meeting_id, status) VALUES ($1, $2, $3)",
        task.ID, task.MeetingID, task.Status,
    )
    require.NoError(t, err)
    return task
}
```

**Usage:**
```go
// Auto-creates a meeting + user dependency chain
task := NewTaskBuilder().Build(t, db)

// Explicit meeting reference
meeting := NewMeetingBuilder().WithTitle("Specific Meeting").Build(t, db)
task := NewTaskBuilder().WithMeetingID(meeting.ID).Build(t, db)
```

### Dependency Graph

```
User (root)
  └── Meeting (requires User)
       └── Task (requires Meeting)
            └── Transcription (requires Task)

Factory chain: Transcription → auto-creates Task → auto-creates Meeting → auto-creates User
```

---

## Test Data Isolation

### Parallel Test Safety

When tests run in parallel, data isolation prevents interference:

| Strategy | Mechanism |
|----------|-----------|
| **Unique IDs** | Every factory generates a UUID. No fixed IDs. No collisions. |
| **Scoped data** | Each test creates its own user → meeting → task chain. No shared entities. |
| **Transaction isolation** | Each test runs in its own transaction (if using rollback strategy). |
| **Schema isolation** | For extreme parallelism, use PostgreSQL schemas per test: `CREATE SCHEMA test_abc123`. |

### What NOT to Do

```go
// BAD: Fixed IDs → will collide in parallel execution
meeting := Meeting{ID: "meeting_001", Title: "Test"}

// BAD: Depending on data from another test
func TestUpdateMeeting(t *testing.T) {
    // Assumes TestCreateMeeting ran first and created "meeting_001"
    resp := httpPut(t, "/api/v1/meetings/meeting_001", ...)
}

// BAD: Shared global seed data modified by multiple tests
var globalMeeting = seedMeeting("shared_meeting")
```

---

## Seeding Strategies by Test Type

| Test Type | Strategy | Rationale |
|-----------|----------|-----------|
| **Service Test** | Factory functions with builder pattern. Container recreation per suite. | Maximum isolation. Fast iteration. Type-safe. |
| **System Test** | Scenario-based seeding via API calls (not direct DB inserts). Truncation between scenarios. | Tests the full creation path. Ensures API and DB agree. |
| **Live Service Test** | Factory functions with explicit cleanup. Unique identifiers to avoid collision with production data. | Must not leave residual state in 3rd-party systems. |
| **Live System Test** | API-driven seeding + explicit teardown hooks. Dedicated test accounts. | Full journey validation requires realistic data created through real APIs. |

### System Test Seeding: API-Driven

In System Tests, seed data through the API, not by inserting directly into the database:

```go
func seedMeetingViaAPI(t *testing.T, client *http.Client) Meeting {
    t.Helper()
    body := `{"title": "Seeded Meeting", "duration": 30}`
    resp := client.Post("/api/v1/meetings", "application/json", strings.NewReader(body))
    require.Equal(t, 201, resp.StatusCode)
    
    var meeting Meeting
    json.NewDecoder(resp.Body).Decode(&meeting)
    return meeting
}
```

**Why?** Direct DB inserts bypass validation, event emission, and side effects. If the API changes how meetings are created (e.g., adds a required field), API-driven seeding catches it immediately. DB inserts silently produce invalid state.

---

## See Also

- `emulation-strategy.md` — Container lifecycle ties to reset strategies
- `validation-layers.md` — Seeding approach per test type
- `worked-examples.md` — Factory patterns in context
