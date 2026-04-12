# Concurrency

## Table of Contents
- [Managed Concurrency with errgroup](#managed-concurrency-with-errgroup)
- [Context Propagation](#context-propagation)
- [Worker Pools](#worker-pools)
- [Goroutine Leak Detection](#goroutine-leak-detection)
- [Sync Primitives](#sync-primitives)

---

## Managed Concurrency with errgroup

All concurrent operations in wordloop-core use `errgroup.WithContext` for lifecycle management. Raw goroutines without lifecycle control are a primary source of goroutine leaks, zombie processes, and resource exhaustion.

### Basic errgroup Pattern

```go
import "golang.org/x/sync/errgroup"

func (s *SynthesisService) ProcessSegments(ctx context.Context, meetingID domain.MeetingID) error {
    segments, err := s.segments.ListByMeeting(ctx, meetingID)
    if err != nil {
        return fmt.Errorf("list segments: %w", err)
    }

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // Bound concurrency to match capacity

    for _, seg := range segments {
        seg := seg // capture for goroutine
        g.Go(func() error {
            if err := s.analyzer.Analyze(ctx, seg); err != nil {
                return fmt.Errorf("analyze segment %s: %w", seg.ID(), err)
            }
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return fmt.Errorf("process segments for meeting %s: %w", meetingID, err)
    }
    return nil
}
```

### Why errgroup Over Raw Goroutines

errgroup provides three properties that raw goroutines lack:

1. **Error propagation** — Captures the first error and cancels the group context, stopping wasted work
2. **Resource bounding** — `SetLimit(n)` restricts concurrent goroutines to match system capacity (DB pool size, CPU cores)
3. **Completion tracking** — `Wait()` blocks until all goroutines complete, making it impossible to leak

### Checking ctx.Done() for Clean Cancellation

Every goroutine within an errgroup must periodically check `ctx.Done()` for clean cancellation:

```go
g.Go(func() error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err() // Clean exit on cancellation
        default:
        }

        if err := processItem(ctx, item); err != nil {
            return err
        }
    }
    return nil
})
```

## Context Propagation

`context.Context` is the "gold thread" of wordloop-core. It carries request-scoped values (trace IDs, deadlines, cancellation signals) through every layer.

### Rules

- `context.Context` is the **first parameter** of every function in the Service, Gateway, and Provider layers
- **Never store** `context.Context` in a struct
- **Always use** `context.WithTimeout` or `context.WithDeadline` for external operations
- Use `context.WithCancel` and defer the cancel immediately to prevent leaks

```go
// Correct — context flows through parameters
func (s *MeetingService) Get(ctx context.Context, id domain.MeetingID) (*domain.Meeting, error) {
    return s.meetings.GetByID(ctx, id)
}

// Correct — timeout for external calls
func (p *ExternalProvider) FetchData(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("build request: %w", err)
    }
    // ...
}
```

### Swallowed Context Antipattern

Never spawn work or make I/O calls without the request context:

```go
// WRONG — goroutine continues after client disconnects, causing zombie requests
go func() {
    db.Query("SELECT ...") // no context
}()

// CORRECT
g.Go(func() error {
    return db.QueryContext(ctx, "SELECT ...")
})
```

## Worker Pools

For long-lived background processing where errgroup's per-batch model doesn't fit, use a dedicated worker pool:

```go
type WorkerPool struct {
    tasks chan func()
    wg    sync.WaitGroup
}

func NewWorkerPool(workers int) *WorkerPool {
    wp := &WorkerPool{
        tasks: make(chan func(), workers*2),
    }
    for i := 0; i < workers; i++ {
        wp.wg.Add(1)
        go func() {
            defer wp.wg.Done()
            for task := range wp.tasks {
                task()
            }
        }()
    }
    return wp
}

func (wp *WorkerPool) Submit(task func()) { wp.tasks <- task }
func (wp *WorkerPool) Shutdown()          { close(wp.tasks); wp.wg.Wait() }
```

## Goroutine Leak Detection

Use the `goroutineleak` profile in `runtime/pprof` to identify permanently blocked goroutines in staging environments. This diagnostic tool leverages the garbage collector to find goroutines waiting on synchronization primitives that have become unreachable.

```go
import (
    "net/http"
    _ "net/http/pprof" // exposes /debug/pprof/ endpoints
)

// In cmd/api — expose pprof for staging/development
go func() {
    slog.Info("pprof server starting", slog.String("addr", ":6060"))
    http.ListenAndServe(":6060", nil)
}()
```

Query goroutine state at runtime:

```bash
# List all goroutines and their current state (blocking call, channel wait, etc.)
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Profile goroutine creation rate over 30 seconds
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

Common leak patterns to watch for:

| Pattern | Symptom | Fix |
|---------|---------|-----|
| Unbuffered channel with no reader | Goroutine blocks on send forever | Use buffered channels or `select` with `ctx.Done()` |
| Missing `context.WithCancel` defer | Child goroutines survive parent cancellation | Always `defer cancel()` |
| Listener without shutdown | Accept loop runs after server stops | Use `signal.NotifyContext` + `Shutdown()` |
| `time.After` in loop | New channel allocated each iteration, old one leaks | Use `time.NewTimer` and `timer.Reset()` |

Integrate goroutine count monitoring into health checks — a steadily climbing goroutine count is the first sign of a leak.

## Sync Primitives

| Primitive | Use Case | Key Property |
|-----------|----------|-------------|
| `errgroup` | Parallel tasks with error handling | First error cancels all |
| `sync.WaitGroup` | Wait for goroutines without error tracking | Simple completion signal |
| `sync.Mutex` | Protect shared mutable state | Exclusive access |
| `sync.RWMutex` | Read-heavy concurrent access | Multiple readers or one writer |
| `sync.Once` | One-time initialization | Thread-safe singleton setup |
| `sync.Pool` | Reuse temporary objects | Reduce GC pressure |

