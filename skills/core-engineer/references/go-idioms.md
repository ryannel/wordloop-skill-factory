# Go Idioms

## Table of Contents
- [Modern Pointer Initialization](#modern-pointer-initialization)
- [Functional Data Pipelines with iter](#functional-data-pipelines-with-iter)
- [Error Handling](#error-handling)
- [Codebase Modernization with go fix](#codebase-modernization-with-go-fix)
- [Performance Optimization](#performance-optimization)
- [Runtime Security & Performance](#runtime-security--performance)

---

## Modern Pointer Initialization

The `new(expr)` built-in creates and initializes pointers to literals or expressions in a single operation. This eliminates temporary variables and `ptr()` helper functions.

```go
// Modern pointer creation — clean, single-expression
p := new(30)                    // *int pointing to 30
name := new("alice")            // *string pointing to "alice"
deadline := new(time.Now().Add(24 * time.Hour)) // *time.Time

// Particularly useful in struct initialization with optional pointer fields
type CreateMeetingRequest struct {
    Title       string
    Description *string
    MaxGuests   *int
}

req := CreateMeetingRequest{
    Title:       "Sprint Planning",
    Description: new("Weekly sync with the team"),
    MaxGuests:   new(50),
}
```

Remove any legacy `ptr()` helper functions from the codebase in favor of the built-in `new(expr)`.

## Functional Data Pipelines with iter

The `iter` package provides lazy-evaluation sequences for memory-efficient data processing. Use `iter.Seq` and `iter.Seq2` instead of building intermediate slices for large datasets.

### Lazy Sequences

```go
// Generate a sequence without allocating a full slice
func ActiveMeetings(meetings []*domain.Meeting) iter.Seq[*domain.Meeting] {
    return func(yield func(*domain.Meeting) bool) {
        for _, m := range meetings {
            if m.Status() == domain.MeetingStatusInProgress {
                if !yield(m) {
                    return // Consumer called break
                }
            }
        }
    }
}

// Consume lazily — only processes until we find what we need
for m := range ActiveMeetings(allMeetings) {
    if m.HostID() == currentUser {
        fmt.Println("Found active meeting:", m.Title())
        break // Iterator stops generating
    }
}
```

### Key-Value Sequences

```go
// Yield key-value pairs for map-like iteration
func MeetingsByHost(meetings []*domain.Meeting) iter.Seq2[domain.UserID, *domain.Meeting] {
    return func(yield func(domain.UserID, *domain.Meeting) bool) {
        for _, m := range meetings {
            if !yield(m.HostID(), m) {
                return
            }
        }
    }
}
```

### When to Use iter

| Scenario | Traditional Approach | iter Approach |
|----------|---------------------|---------------|
| Large datasets | High memory (intermediate copies) | Near-zero allocation (lazy) |
| Early exit needed | Process full collection | Stops on break |
| Chained transforms | Multiple slice allocations | Single pass |
| Simple small sets | Fine as-is | Unnecessary overhead |

Use `iter` for processing pipelines over large collections. For small, simple collections, plain for loops remain more readable.

## Error Handling

### Wrapping with Context

Every error return should add context with `%w`:

```go
if err := s.repo.Create(ctx, entity); err != nil {
    return fmt.Errorf("create meeting for host %s: %w", hostID, err)
}
```

### Type-Safe Error Extraction

Use `errors.AsType` for compile-time safe extraction of custom error types — it avoids reflection overhead:

```go
// Instead of errors.As with a target variable:
if ve, ok := errors.AsType[*domain.ValidationError](err); ok {
    // ve is already *domain.ValidationError — no reflection, no allocation
    handleValidation(ve)
}

// Compare with errors.Is for sentinel errors:
if errors.Is(err, domain.ErrNotFound) {
    return http.StatusNotFound
}
```

### Error Sanitization

Never return raw internal errors to HTTP clients. Providers and services wrap errors for internal context; entrypoints sanitize for the client:

```go
// Internal: full error chain visible in logs
// "create meeting for host user-123: insert meeting: pq: duplicate key value violates constraint"

// External: sanitized response
// {"error": "meeting already exists", "request_id": "abc-123"}
```

## Codebase Modernization with go fix

Use `go fix` to automatically modernize the codebase as new language features become available. Integrate this into the CI pipeline to ensure idiom consistency:

| Modernizer | Replaces | With |
|-----------|---------|------|
| `any` | `interface{}` | `any` |
| `minmax` | Manual comparison logic | Built-in `min()` / `max()` |
| `rangeint` | 3-clause for-loops | Range-over-int syntax |
| `newexpr` | `v := 42; p := &v` | `p := new(42)` |

```bash
# Run all modernizers
go fix ./...

# Verify in CI
golangci-lint run ./...
```

## Performance Optimization

### Profile-Guided Optimization (PGO)

Collect CPU profiles from production and feed them into the compiler for more aggressive inlining and devirtualization:

```bash
# 1. Collect profile from production
curl -o cpu.pprof http://service:6060/debug/pprof/profile?seconds=30

# 2. Build with PGO
go build -pgo=cpu.pprof -o service ./cmd/api
```

PGO requires no code changes and typically yields a 2-10% performance improvement in CPU-bound paths.

### Resource Pooling

Use `sync.Pool` for frequently allocated temporary objects to reduce garbage collector pressure:

```go
var bufferPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func processWithPool(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    buf.Write(data)
    // Process...
}
```

## Runtime Security & Performance

### Green Tea Garbage Collector

Go 1.26 enables the "Green Tea" garbage collector by default. It is optimized for modern multi-core CPUs where memory locality matters, providing a 10–40% reduction in GC overhead for real-world programs. On x86-64 and ARM64 platforms, it leverages SIMD vector instructions for faster memory scanning.

Practical impact: **manual `GOGC` tuning is rarely needed**. The Green Tea collector automatically adapts to workload characteristics. Only adjust `GOGC` if profiling shows GC as a measurable bottleneck in your specific service.

```bash
# Check GC behavior in production
GODEBUG=gctrace=1 ./service

# Only tune if profiling justifies it (default is usually optimal)
GOGC=200 ./service  # less frequent GC, higher memory usage
```

### Heap Randomization

On 64-bit platforms, Go 1.26 randomizes the heap base address at startup. This makes memory-based exploits significantly harder to execute — no code changes required.

### Sensitive Data Handling

Use the experimental `runtime/secret` package for cryptographic operations. It automatically wipes sensitive temporary data from registers and memory when the operation completes, protecting against memory-dump attacks:

```go
import "runtime/secret"

func processAPIKey(key string) {
    s := secret.New([]byte(key))
    defer s.Destroy() // Zeroes memory on scope exit

    // Use s.Bytes() for cryptographic operations
    hash := hmac.New(sha256.New, s.Bytes())
    // ...
}
```

