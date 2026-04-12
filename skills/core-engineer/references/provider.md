# Provider Layer

## Table of Contents
- [Provider Design](#provider-design)
- [PostgreSQL Provider](#postgresql-provider)
- [Compile-Time Interface Verification](#compile-time-interface-verification)
- [Error Translation](#error-translation)
- [Resiliency Patterns](#resiliency-patterns)
  - [Timeouts](#timeouts-on-every-external-call)
  - [Circuit Breakers](#circuit-breakers)
  - [Retry with Backoff and Jitter](#retry-with-exponential-backoff-and-jitter)
  - [Retry + Circuit Breaker Ordering](#retry--circuit-breaker-ordering)
  - [Retry Budgets](#retry-budgets)
  - [eBPF Infrastructure Monitoring](#ebpf-infrastructure-monitoring)
  - [Resiliency Antipatterns](#resiliency-antipatterns)
- [Resource Pooling](#resource-pooling)

---

## Provider Design

Providers are the technology edge of the hexagon. They implement gateway interfaces using real dependencies (Postgres, Redis, S3, external APIs). Providers are driven (outbound) adapters — they are called by services through gateway contracts.

Key rules:
- Implement gateway interfaces defined in `internal/core/gateway`
- Depend on Domain and Gateway — **never import Service or Entrypoint**
- Wrap all driver errors with `fmt.Errorf("context: %w", err)`, mapping to domain sentinel errors where appropriate
- Include compile-time interface verification
- Configure connection pools and resource limits here
- Place integration tests alongside provider code using testcontainers

## PostgreSQL Provider

```go
package postgres

import (
    "context"
    "database/sql"
    "fmt"

    "wordloop-core/internal/core/domain"
)

// PostgresMeetingRepo implements gateway.MeetingRepository using PostgreSQL.
type PostgresMeetingRepo struct {
    db *sql.DB
}

// NewPostgresMeetingRepo constructs a PostgresMeetingRepo.
func NewPostgresMeetingRepo(db *sql.DB) *PostgresMeetingRepo {
    return &PostgresMeetingRepo{db: db}
}

// Compile-time interface verification
var _ gateway.MeetingRepository = (*PostgresMeetingRepo)(nil)

func (r *PostgresMeetingRepo) Create(ctx context.Context, m *domain.Meeting) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO meetings (id, title, host_id, starts_at, duration, status, created_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        m.ID(), m.Title(), m.HostID(), m.StartsAt(), m.Duration(), m.Status(), m.CreatedAt(),
    )
    if err != nil {
        return fmt.Errorf("insert meeting %s: %w", m.ID(), err)
    }
    return nil
}

func (r *PostgresMeetingRepo) GetByID(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error) {
    row := r.db.QueryRowContext(ctx,
        `SELECT id, title, host_id, starts_at, duration, status, created_at
         FROM meetings WHERE id = $1`, id,
    )

    var m meetingRow
    if err := row.Scan(&m.id, &m.title, &m.hostID, &m.startsAt, &m.duration, &m.status, &m.createdAt); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("meeting %s: %w", id, domain.ErrNotFound)
        }
        return nil, fmt.Errorf("scan meeting %s: %w", id, err)
    }

    return m.toDomain(), nil
}
```

## Compile-Time Interface Verification

Always include a compile-time check that your provider satisfies the gateway interface:

```go
var _ gateway.MeetingRepository = (*PostgresMeetingRepo)(nil)
```

This catches interface drift at compile time rather than runtime. Place this declaration immediately after the struct definition.

## Error Translation

Providers are responsible for translating technology-specific errors into domain-level sentinel errors. This ensures that the service layer never needs to import `database/sql` or any other driver package.

### Mapping Rules

| Driver Error | Domain Error | Example |
|-------------|-------------|---------|
| `sql.ErrNoRows` | `domain.ErrNotFound` | Row not found in SELECT |
| Unique constraint violation | `domain.ErrConflict` | Duplicate key on INSERT |
| Connection refused / timeout | Wrap with context, let it propagate | Infrastructure failure |

```go
func (r *PostgresMeetingRepo) Create(ctx context.Context, m *domain.Meeting) error {
    _, err := r.db.ExecContext(ctx, insertSQL, m.ID(), m.Title())
    if err != nil {
        // Check for unique constraint violation
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23505" {
            return fmt.Errorf("meeting %s: %w", m.ID(), domain.ErrConflict)
        }
        return fmt.Errorf("insert meeting: %w", err)
    }
    return nil
}
```

## Resiliency Patterns

The provider layer is where resiliency patterns live, because providers are the boundary between the application and fallible external systems. The guiding principle: **failure is normal** — systems must be designed to fail fast and recover gracefully.

### Timeouts on Every External Call

Every outbound call from a provider must have an explicit timeout via `context.WithTimeout`. Never rely on default timeouts (which are often infinite):

```go
func (p *ExternalProvider) FetchData(ctx context.Context, id string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, p.baseURL+"/data/"+id, nil)
    if err != nil {
        return nil, fmt.Errorf("build request: %w", err)
    }

    resp, err := p.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("fetch data %s: %w", id, err)
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}
```

### Circuit Breakers

Circuit breakers prevent cascading failures by "tripping" when a downstream dependency is unhealthy. They cycle through three states:

| State | Behavior |
|-------|----------|
| **Closed** | Requests pass through normally. Failures are counted. |
| **Open** | All requests fail immediately with `ErrOpenState`. No traffic reaches the dependency. |
| **Half-Open** | A limited number of probe requests are allowed. If they succeed, the breaker closes. If they fail, it re-opens. |

```go
import "github.com/sony/gobreaker"

type ResilientServiceClient struct {
    client *http.Client
    cb     *gobreaker.CircuitBreaker
}

func NewResilientServiceClient(client *http.Client) *ResilientServiceClient {
    return &ResilientServiceClient{
        client: client,
        cb: gobreaker.NewCircuitBreaker(gobreaker.Settings{
            Name:        "external-service",
            MaxRequests: 3,                       // probe requests allowed in half-open
            Interval:    10 * time.Second,         // reset interval for failure counts in closed state
            Timeout:     30 * time.Second,         // time spent in open before transitioning to half-open
            ReadyToTrip: func(counts gobreaker.Counts) bool {
                // Trip at 50% failure rate after a minimum of 10 requests
                failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
                return counts.Requests >= 10 && failureRatio > 0.5
            },
            OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
                slog.Warn("circuit breaker state change",
                    slog.String("breaker", name),
                    slog.String("from", from.String()),
                    slog.String("to", to.String()),
                )
            },
        }),
    }
}

func (c *ResilientServiceClient) Fetch(ctx context.Context, url string) ([]byte, error) {
    result, err := c.cb.Execute(func() (any, error) {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
        if err != nil {
            return nil, err
        }
        resp, err := c.client.Do(req)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()
        return io.ReadAll(resp.Body)
    })
    if err != nil {
        return nil, fmt.Errorf("fetch %s: %w", url, err)
    }
    return result.([]byte), nil
}
```

### Retry with Exponential Backoff and Jitter

Simple retry loops without jitter cause "thundering herd" failures — when a downstream recovers, all waiting clients retry simultaneously and overwhelm it again. Retries must use exponential backoff with random jitter:

```
Expected wait = min(base × 2^attempt, cap) × random(0.5, 1.0)
```

Where `base` is the initial delay (e.g., 100ms), `attempt` is the retry number, and `cap` is the maximum wait (e.g., 10s). The random factor prevents synchronized retry storms.

```go
import "github.com/hashicorp/go-retryablehttp"

func NewResilientHTTPClient() *http.Client {
    retryClient := retryablehttp.NewClient()
    retryClient.RetryMax = 5
    retryClient.RetryWaitMin = 100 * time.Millisecond
    retryClient.RetryWaitMax = 10 * time.Second
    // Library handles exponential backoff with jitter internally
    return retryClient.StandardClient()
}
```

### Retry + Circuit Breaker Ordering

Place retry logic **outside** the circuit breaker. This ensures the breaker counts the final failure of all retry attempts as a single signal. If retries are placed *inside* the breaker, a few transient network blips generate multiple failure counts and prematurely trip the circuit for all users.

```
Request → Retry (up to N attempts) → Circuit Breaker → Downstream Service
                                      │
                                      ├─ Closed: passes through
                                      ├─ Open: fails immediately (no downstream call)
                                      └─ Half-Open: allows probe
```

### Retry Budgets

Never allow retries to increase total traffic by more than 10%. Implement a token bucket: for every 10 successful requests, one retry token is earned. When retry tokens are exhausted, requests fail immediately rather than adding load to an already struggling dependency.

```go
type RetryBudget struct {
    mu     sync.Mutex
    tokens float64
    max    float64
    ratio  float64 // tokens earned per success (e.g., 0.1 = 1 token per 10 successes)
}

func NewRetryBudget(maxTokens float64) *RetryBudget {
    return &RetryBudget{tokens: maxTokens, max: maxTokens, ratio: 0.1}
}

func (b *RetryBudget) RecordSuccess() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.tokens = min(b.tokens+b.ratio, b.max)
}

func (b *RetryBudget) AllowRetry() bool {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.tokens >= 1 {
        b.tokens--
        return true
    }
    return false
}
```

### eBPF Infrastructure Monitoring

For infrastructure-level visibility without modifying application code, use eBPF-based monitoring. This provides kernel-level data that is invisible to standard application metrics.

| eBPF Probe Type | Data Collected | Use Case |
|-----------------|---------------|----------|
| **Kprobes** (Kernel) | Syscall execution time | Identifies kernel-level I/O bottlenecks |
| **Uprobes** (User) | Go runtime function entry | Traces application behavior without code changes |
| **XDP** (Network) | Per-packet latency | Detects network jitter at the deepest layer |
| **Tracepoints** | Disk block I/O | Monitors storage performance for databases |

Using `bpftrace`, profile CPU usage for a specific Go process:

```bash
# Profile CPU at 99Hz for a specific process
bpftrace -e 'profile:hz:99 /pid == 1234/ { @[ustack] = count(); }'
```

This generates a "zero-overhead" flame graph that identifies hot functions — expensive GC cycles, excessive goroutine context switching — that are invisible to OTel metrics.

### Resiliency Antipatterns

| Antipattern | Consequence | Correct Approach |
|-------------|-------------|-----------------|
| No timeout on external calls | Requests hang indefinitely, goroutine leak | `context.WithTimeout` on every outbound call |
| Retry without jitter | Thundering herd overwhelms recovering service | Exponential backoff with random jitter |
| Retry inside circuit breaker | Transient blips prematurely trip the circuit | Retry outside, breaker inside |
| Unbounded retries | Cascading load amplification | Retry budget with token bucket |
| Logging internal errors to client | CWE-209: exposure of infrastructure details | Log detailed error internally, return sanitized message with request_id |

## Resource Pooling

Configure database connection pools to match the concurrency characteristics of the service:

```go
func NewDatabasePool(connStr string) (*sql.DB, error) {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("open database: %w", err)
    }

    // Match pool size to worker count to prevent connection starvation
    db.SetMaxOpenConns(25)
    // Maintain idle connections to avoid TLS handshake latency on load surges
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(1 * time.Minute)

    return db, nil
}
```

For high-throughput paths, use `sync.Pool` to reuse allocated objects like byte buffers:

```go
var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.Write(data)
    // Process...
}
```
