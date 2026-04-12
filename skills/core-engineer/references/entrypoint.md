# Entrypoint Layer

## Table of Contents
- [Entrypoint Design](#entrypoint-design)
- [HTTP Handler Pattern](#http-handler-pattern)
- [Request/Response Mapping](#requestresponse-mapping)
- [Request Validation](#request-validation)
- [Error Sanitization](#error-sanitization)
- [Inbound Resiliency](#inbound-resiliency)
  - [Rate Limiting](#rate-limiting)
  - [Per-Client Rate Limiting](#per-client-rate-limiting)
  - [Request Timeouts](#request-timeouts)
  - [Request Body Size Limits](#request-body-size-limits)
  - [Load Shedding](#load-shedding)
  - [Backpressure Signaling](#backpressure-signaling)
- [Idempotency](#idempotency)
- [Middleware Composition](#middleware-composition)

---

## Entrypoint Design

Entrypoints translate external requests into service calls and service responses into external responses. For HTTP, this means parsing requests, calling services, and writing responses. Entrypoints are driving (inbound) adapters — they sit on the outer edge of the hexagon alongside providers, but serve the opposite direction.

Key rules:
- Import Services and Domain — **never import Providers or Gateways directly**
- Map domain errors to HTTP status codes
- Sanitize errors before returning to the client — never expose raw internal errors
- Log errors **exactly once** here — services and providers only wrap and propagate
- Handle all request validation and response serialization
- Enforce all inbound resiliency (rate limits, timeouts, body size limits)

## HTTP Handler Pattern

Handlers are thin translation layers. They parse the request, call a service method, and translate the result:

```go
type MeetingHandler struct {
    meetings *service.MeetingService
}

func NewMeetingHandler(meetings *service.MeetingService) *MeetingHandler {
    return &MeetingHandler{meetings: meetings}
}

func (h *MeetingHandler) Create(w http.ResponseWriter, r *http.Request) {
    // 1. Parse and validate the HTTP request
    var req createMeetingRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, r, http.StatusBadRequest, "invalid request body")
        return
    }

    if problems := req.validate(); len(problems) > 0 {
        writeValidationError(w, r, problems)
        return
    }

    // 2. Translate to service-level request
    svcReq := service.ScheduleMeetingRequest{
        Title:    req.Title,
        HostID:   domain.UserID(req.HostID),
        StartsAt: req.StartsAt,
        Duration: req.Duration,
    }

    // 3. Call the service
    meeting, err := h.meetings.ScheduleMeeting(r.Context(), svcReq)
    if err != nil {
        handleServiceError(w, r, err)
        return
    }

    // 4. Translate to HTTP response
    writeJSON(w, http.StatusCreated, toMeetingResponse(meeting))
}
```

Handlers must never contain business logic. If you find yourself writing `if` conditions based on domain state inside a handler, that logic belongs in the service or domain layer.

## Request/Response Mapping

Define request and response types in the entrypoint layer with JSON tags. These are distinct from both domain entities and service request types:

```go
// Request type — JSON tags, validation tags
type createMeetingRequest struct {
    Title    string        `json:"title"`
    HostID   string        `json:"host_id"`
    StartsAt time.Time     `json:"starts_at"`
    Duration time.Duration `json:"duration"`
}

// Response type — shaped for the API consumer
type meetingResponse struct {
    ID        string    `json:"id"`
    Title     string    `json:"title"`
    HostID    string    `json:"host_id"`
    StartsAt  time.Time `json:"starts_at"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

// Mapper — translates domain to response
func toMeetingResponse(m *domain.Meeting) meetingResponse {
    return meetingResponse{
        ID:        m.ID().String(),
        Title:     m.Title(),
        HostID:    m.HostID().String(),
        StartsAt:  m.StartsAt(),
        Status:    string(m.Status()),
        CreatedAt: m.CreatedAt(),
    }
}
```

Domain entities never have JSON tags. The entrypoint owns the serialization format; the domain owns the business rules. This separation means API response shapes can evolve independently of the domain model.

## Request Validation

Validate request data at the entrypoint before it reaches the service layer. This keeps validation errors cheap (no DB round-trips) and produces clear client-facing messages:

```go
func (r *createMeetingRequest) validate() map[string]string {
    problems := make(map[string]string)

    if strings.TrimSpace(r.Title) == "" {
        problems["title"] = "title is required"
    }
    if r.HostID == "" {
        problems["host_id"] = "host_id is required"
    }
    if r.StartsAt.Before(time.Now()) {
        problems["starts_at"] = "starts_at must be in the future"
    }
    if r.Duration <= 0 {
        problems["duration"] = "duration must be positive"
    }

    return problems
}

func writeValidationError(w http.ResponseWriter, r *http.Request, problems map[string]string) {
    writeJSON(w, http.StatusUnprocessableEntity, map[string]any{
        "error":   "validation failed",
        "details": problems,
    })
}
```

Note the distinction: **entrypoint validation** checks request format (required fields, valid UUIDs, future dates). **Domain validation** checks business rules (meeting duration limits, host permissions). Both are necessary.

## Error Sanitization

Never return raw internal errors to HTTP clients. Providers and services wrap errors for internal diagnostics; the entrypoint sanitizes for the client. Leaking internal errors exposes infrastructure details (CWE-209) — database connection strings, file paths, SQL table names.

```go
func handleServiceError(w http.ResponseWriter, r *http.Request, err error) {
    // Generate a unique request ID for correlation
    requestID := middleware.RequestIDFrom(r.Context())

    // Log the full error chain internally — this is the ONE place errors are logged
    slog.ErrorContext(r.Context(), "request failed",
        slog.String("request_id", requestID),
        slog.String("method", r.Method),
        slog.String("path", r.URL.Path),
        slog.String("error", err.Error()),
    )

    // Map domain errors to HTTP status codes
    switch {
    case errors.Is(err, domain.ErrNotFound):
        writeError(w, r, http.StatusNotFound, "resource not found")
    case errors.Is(err, domain.ErrConflict):
        writeError(w, r, http.StatusConflict, "resource already exists")
    case errors.Is(err, domain.ErrValidation):
        writeError(w, r, http.StatusUnprocessableEntity, "validation failed")
    case errors.Is(err, domain.ErrUnauthorized):
        writeError(w, r, http.StatusForbidden, "access denied")
    default:
        // Generic 500 — never expose the raw error message
        writeError(w, r, http.StatusInternalServerError, "internal error")
    }
}

func writeError(w http.ResponseWriter, r *http.Request, status int, message string) {
    writeJSON(w, status, map[string]string{
        "error":      message,
        "request_id": middleware.RequestIDFrom(r.Context()),
    })
}
```

| Internal log (for developers) | External response (for clients) |
|-------------------------------|--------------------------------|
| `"save meeting: insert meeting: pq: duplicate key violates constraint \"meetings_pkey\""` | `{"error": "resource already exists", "request_id": "abc-123"}` |

## Inbound Resiliency

Provider resiliency defends the service from its dependencies failing. **Entrypoint resiliency defends the service from its clients overwhelming it.**

### Rate Limiting

Use `golang.org/x/time/rate` for global rate limiting. This protects against burst traffic and abuse:

```go
import "golang.org/x/time/rate"

func rateLimitMiddleware(limiter *rate.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// At server startup — inject the limiter
limiter := rate.NewLimiter(rate.Limit(1000), 100) // 1000 req/sec, burst of 100
mux.Use(rateLimitMiddleware(limiter))
```

### Per-Client Rate Limiting

Global limits protect the service overall, but per-client limits prevent a single client from consuming all capacity. Use a map of limiters keyed by client identity:

```go
type clientRateLimiter struct {
    mu       sync.Mutex
    clients  map[string]*rate.Limiter
    rate     rate.Limit
    burst    int
}

func newClientRateLimiter(r rate.Limit, burst int) *clientRateLimiter {
    return &clientRateLimiter{
        clients: make(map[string]*rate.Limiter),
        rate:    r,
        burst:   burst,
    }
}

func (crl *clientRateLimiter) getLimiter(clientID string) *rate.Limiter {
    crl.mu.Lock()
    defer crl.mu.Unlock()

    lim, exists := crl.clients[clientID]
    if !exists {
        lim = rate.NewLimiter(crl.rate, crl.burst)
        crl.clients[clientID] = lim
    }
    return lim
}

func perClientRateLimitMiddleware(crl *clientRateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            clientID := r.Header.Get("X-API-Key") // or extract from auth token
            if clientID == "" {
                clientID = r.RemoteAddr
            }

            if !crl.getLimiter(clientID).Allow() {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

### Request Timeouts

Enforce a maximum duration for every incoming request. This prevents slow clients or pathological queries from holding server resources indefinitely:

```go
func timeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.TimeoutHandler(next, timeout, `{"error": "request timeout"}`)
    }
}

// At server startup
mux.Use(timeoutMiddleware(30 * time.Second))
```

Additionally, configure read/write timeouts on the HTTP server itself to prevent slowloris attacks:

```go
srv := &http.Server{
    Addr:         cfg.Addr,
    Handler:      mux,
    ReadTimeout:  10 * time.Second,  // max time to read entire request (including body)
    WriteTimeout: 30 * time.Second,  // max time to write response
    IdleTimeout:  120 * time.Second, // max time for keep-alive connections
}
```

### Request Body Size Limits

Prevent clients from sending excessively large payloads that could exhaust memory:

```go
func maxBodySizeMiddleware(maxBytes int64) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
            next.ServeHTTP(w, r)
        })
    }
}

// At server startup — 1MB limit
mux.Use(maxBodySizeMiddleware(1 << 20))
```

When `MaxBytesReader` limit is exceeded, subsequent reads return an error and the server automatically responds with 413 Request Entity Too Large.

### Load Shedding

Under extreme load, it's better to reject requests quickly than to accept them and respond slowly. Use an in-flight request counter to shed load when the server approaches capacity:

```go
type loadShedder struct {
    inFlight atomic.Int64
    maxLoad  int64
}

func newLoadShedder(maxConcurrent int64) *loadShedder {
    return &loadShedder{maxLoad: maxConcurrent}
}

func (ls *loadShedder) middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        current := ls.inFlight.Add(1)
        defer ls.inFlight.Add(-1)

        if current > ls.maxLoad {
            w.Header().Set("Retry-After", "5")
            http.Error(w, "service overloaded", http.StatusServiceUnavailable)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// At server startup — shed load beyond 500 concurrent requests
shedder := newLoadShedder(500)
mux.Use(shedder.middleware)
```

### Backpressure Signaling

When rejecting requests (rate limit or load shed), always include the `Retry-After` header. This tells well-behaved clients when to retry instead of hammering the service in a tight loop:

```go
w.Header().Set("Retry-After", "5") // seconds until client should retry
```

| HTTP Status | Meaning | When to Use |
|-------------|---------|------------|
| `429 Too Many Requests` | Client is exceeding its rate limit | Rate limiting (global or per-client) |
| `503 Service Unavailable` | Server is at capacity | Load shedding, dependency failure |

## Idempotency

Design mutating endpoints to handle duplicate requests safely. This is essential for distributed systems where retries are inevitable — network failures, client timeouts, and agentic workflows all produce duplicate requests.

### Idempotency Key Pattern

Clients provide a unique key with each mutating request. The server stores the result of the first execution and returns the cached result for subsequent requests with the same key:

```go
func idempotencyMiddleware(store IdempotencyStore) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Only apply to mutating methods
            if r.Method == http.MethodGet || r.Method == http.MethodHead {
                next.ServeHTTP(w, r)
                return
            }

            key := r.Header.Get("Idempotency-Key")
            if key == "" {
                next.ServeHTTP(w, r) // No key — process normally
                return
            }

            // Check for a cached response
            if cached, found := store.Get(r.Context(), key); found {
                w.Header().Set("X-Idempotent-Replayed", "true")
                w.WriteHeader(cached.StatusCode)
                w.Write(cached.Body)
                return
            }

            // Capture the response
            rec := &responseRecorder{ResponseWriter: w, statusCode: http.StatusOK}
            next.ServeHTTP(rec, r)

            // Store the result for future replays (TTL: 24 hours)
            store.Set(r.Context(), key, &CachedResponse{
                StatusCode: rec.statusCode,
                Body:       rec.body.Bytes(),
            })
        })
    }
}
```

### Naturally Idempotent Operations

Some operations are idempotent by design and don't need an explicit key:

| Method | Idempotent? | Rationale |
|--------|------------|-----------|
| `GET` | Yes | Read-only, no side effects |
| `PUT` | Yes | Full replacement — same input produces same state |
| `DELETE` | Yes | Deleting an already-deleted resource is a no-op |
| `POST` | **No** | Creates new resources — duplicates cause conflicts |
| `PATCH` | **No** | Partial updates may have additive side effects |

For `POST` and `PATCH`, always accept and encourage `Idempotency-Key` headers.

Apply middleware in a deliberate order. Earlier middleware wraps later middleware, so execution flows outside-in on request and inside-out on response:

```go
func buildMiddlewareChain(handler http.Handler, cfg *config.Config) http.Handler {
    // Applied bottom-to-top (last wraps first)
    handler = routes.NewRouter(handler)       // 6. Routing
    handler = authMiddleware(handler)          // 5. Authentication
    handler = maxBodySizeMiddleware(1<<20)(handler)  // 4. Body size limit
    handler = timeoutMiddleware(30*time.Second)(handler) // 3. Request timeout
    handler = rateLimitMiddleware(limiter)(handler)   // 2. Rate limiting
    handler = requestIDMiddleware(handler)     // 1. Request ID (outermost)

    return handler
}
```

### Recommended Ordering

| Order | Middleware | Rationale |
|-------|-----------|-----------|
| 1 | Request ID | Assigns correlation ID before any processing |
| 2 | Recovery (panic) | Catches panics, logs stack trace, returns 500 |
| 3 | Rate limiting | Rejects excess traffic before any work is done |
| 4 | Request timeout | Bounds the total request lifecycle |
| 5 | Body size limit | Prevents memory exhaustion during parsing |
| 6 | CORS | Handles preflight requests before auth |
| 7 | Authentication | Validates identity |
| 8 | Logging | Records request/response details |
| 9 | Routing | Dispatches to specific handlers |
