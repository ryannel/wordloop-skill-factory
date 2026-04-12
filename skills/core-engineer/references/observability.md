# Observability

## Table of Contents
- [Structured Logging with slog](#structured-logging-with-slog)
- [OpenTelemetry Integration](#opentelemetry-integration)
- [Graceful Shutdown](#graceful-shutdown)
- [Health Checks](#health-checks)

---

## Structured Logging with slog

wordloop-core uses the `log/slog` package for all logging. Plain-text logs are not used in production — all output is structured JSON.

### Logger Configuration

The global logger is configured at application startup in `cmd/`:

```go
func configureLogger(env string) {
    var level slog.Level
    switch env {
    case "production":
        level = slog.LevelInfo
    case "development":
        level = slog.LevelDebug
    default:
        level = slog.LevelInfo
    }

    opts := &slog.HandlerOptions{
        Level:     level,
        AddSource: true, // Enables log-to-code mapping in observability platforms
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Redact sensitive fields
            switch a.Key {
            case "password", "token", "authorization", "api_key":
                return slog.String(a.Key, "[REDACTED]")
            }
            return a
        },
    }

    handler := slog.NewJSONHandler(os.Stdout, opts)

    logger := slog.New(handler).With(
        slog.String("service.name", "wordloop-core"),
        slog.String("service.version", version.Version),
        slog.String("deployment.env", env),
    )

    slog.SetDefault(logger)
}
```

### Context-Aware Logging

Always use `slog.InfoContext()`, `slog.ErrorContext()`, etc. By passing the context, the logger automatically injects trace_id and span_id when the OpenTelemetry bridge is active:

```go
// Correct — trace-correlated
slog.InfoContext(ctx, "meeting scheduled",
    slog.String("meeting.id", meeting.ID().String()),
    slog.String("host.id", meeting.HostID().String()),
    slog.Duration("duration", meeting.Duration()),
)

// Correct — type-safe attributes prevent broken log entries
slog.ErrorContext(ctx, "failed to persist meeting",
    slog.String("meeting.id", id.String()),
    slog.String("error", err.Error()),
)
```

### Log Level Discipline

| Level | Use | Production Default |
|-------|-----|-------------------|
| **Debug** | Detailed troubleshooting, variable dumps | Disabled (toggle dynamically) |
| **Info** | High-level operational events: "meeting created", "sync completed" | Enabled |
| **Warn** | Non-critical anomalies: notification delivery failure, rate limit approach | Enabled |
| **Error** | Actionable failures requiring investigation: database errors, panic recovery | Enabled |

### Dynamic Log Levels

Use `slog.LevelVar` to change log verbosity at runtime without restarts — essential for investigating transient production issues:

```go
var logLevel = new(slog.LevelVar)

// At startup
handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: logLevel})

// Toggle via admin endpoint
http.HandleFunc("/admin/log-level", func(w http.ResponseWriter, r *http.Request) {
    level := r.URL.Query().Get("level")
    switch level {
    case "debug":
        logLevel.Set(slog.LevelDebug)
    case "info":
        logLevel.Set(slog.LevelInfo)
    }
})
```

### The "Log and Return" Antipattern

Log errors **exactly once** at the entrypoint (HTTP handler). In service and provider layers, only wrap errors with context using `%w` and return them:

```go
// WRONG — error logged at multiple layers
func (s *Service) Do(ctx context.Context) error {
    err := s.repo.Fetch(ctx)
    if err != nil {
        slog.Error("fetch failed", "error", err) // logged here...
        return err                                 // ...and again by caller
    }
    return nil
}

// CORRECT — wrap and return, log at the top
func (s *Service) Do(ctx context.Context) error {
    err := s.repo.Fetch(ctx)
    if err != nil {
        return fmt.Errorf("service fetch: %w", err) // context added, no log
    }
    return nil
}
```

## OpenTelemetry Integration

wordloop-core uses OpenTelemetry (OTel) as the universal standard for distributed tracing and metrics, providing vendor-neutral telemetry.

### Trace Initialization

```go
func initTracer(ctx context.Context) (func(context.Context) error, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
    )
    if err != nil {
        return nil, fmt.Errorf("create trace exporter: %w", err)
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("wordloop-core"),
            semconv.ServiceVersion(version.Version),
        ),
        resource.WithProcess(),
        resource.WithContainer(),
    )
    if err != nil {
        return nil, fmt.Errorf("create resource: %w", err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.AlwaysOn())),
    )
    otel.SetTracerProvider(tp)

    return tp.Shutdown, nil
}
```

### Log-Trace Correlation

Use the `otelslog` bridge to connect structured logs with distributed traces:

```go
import "go.opentelemetry.io/contrib/bridges/otelslog"

func registerOTelBridge() {
    logger := otelslog.NewLogger("wordloop-core")
    slog.SetDefault(logger)
}
```

With this bridge active, every `slog.InfoContext(ctx, ...)` call automatically includes `trace_id` and `span_id`, enabling teams to pivot from a metric alert → trace visualization → specific log entries for a request.

### Instrumented HTTP Client

Wrap outbound HTTP clients with OTel transport for automatic trace propagation:

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

func newInstrumentedClient() *http.Client {
    return &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
}
```

### Business Metrics

Track application metrics using the OTel Metrics API. Focus on the RED pattern — Rate, Errors, Duration:

```go
var meter = otel.Meter("wordloop-core")

func recordMeetingDuration(ctx context.Context, duration time.Duration, status string) {
    histogram, _ := meter.Float64Histogram(
        "meeting.processing.duration",
        metric.WithDescription("Time to process meeting synthesis"),
        metric.WithUnit("ms"),
    )

    histogram.Record(ctx, float64(duration.Milliseconds()), metric.WithAttributes(
        attribute.String("meeting.status", status), // low-cardinality only
    ))
}
```

Avoid high-cardinality metric labels (user IDs, meeting IDs). Use traces for high-cardinality investigation.

## Graceful Shutdown

Production Go services implement a multi-stage shutdown sequence to prevent dropped connections and data loss:

```go
func runServer(ctx context.Context, cfg *config.Config) error {
    srv := &http.Server{Addr: cfg.Addr, Handler: router}

    // Stage 1: Trap termination signals
    sigCtx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("server failed", "error", err)
        }
    }()

    <-sigCtx.Done()
    slog.Info("shutdown signal received")

    // Stage 2: Flip readiness probe and wait for LB propagation
    isReady.Store(false)
    time.Sleep(15 * time.Second) // allow load balancers to stop sending traffic

    // Stage 3: Drain in-flight requests
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown timeout exceeded, forcing close", "error", err)
        return srv.Close()
    }

    // Stage 4: Close infrastructure (DB, cache) last
    return db.Close()
}
```

### Why the Stages Matter

1. **Readiness flip + delay** — The load balancer removes the pod from its pool before the server stops
2. **Graceful drain** — In-flight requests get time to complete
3. **Resource cleanup** — Database connections close after all requests have drained, so in-flight queries complete

## Health Checks

Divide health checks into three distinct endpoints for orchestrator coordination:

```go
// /livez — is the process alive?
// Static 200 OK. If this fails, Kubernetes restarts the pod.
func livenessHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}

// /readyz — is the service ready to accept traffic?
// Checks database connectivity, cache availability, upstream health.
func readinessHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()

        if err := db.PingContext(ctx); err != nil {
            http.Error(w, "database unavailable", http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    }
}

// /startupz — has initial bootstrapping completed?
// Blocks liveness/readiness until caches are warm, migrations are done.
func startupHandler(ready *atomic.Bool) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if !ready.Load() {
            http.Error(w, "starting up", http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    }
}
```

| Endpoint | Checks | Orchestrator Action |
|----------|--------|-------------------|
| `/livez` | Static 200 | Restarts pod if deadlocked |
| `/readyz` | DB, cache, upstreams | Removes from load balancer |
| `/startupz` | Boot completion | Defers liveness/readiness probes |
