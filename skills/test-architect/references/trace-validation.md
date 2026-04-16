# Trace-Driven Testing & Continuous Profiling

This module defines how to validate OpenTelemetry (OTel) traces as first-class test assertions and how to integrate continuous profiling into the testing lifecycle. The boundary between "test" and "monitor" is dissolved — observability is a design-time concern.

---

## Table of Contents

1. [Core Principle: Observability-Driven Development](#core-principle-observability-driven-development)
2. [Shifting Observability Left](#shifting-observability-left)
3. [Trace-Driven Test Architecture](#trace-driven-test-architecture)
4. [Implementing Trace Assertions](#implementing-trace-assertions)
5. [Context Propagation Validation](#context-propagation-validation)
6. [Continuous Profiling with eBPF](#continuous-profiling-with-ebpf)
7. [Tooling Reference](#tooling-reference)

---

## Core Principle: Observability-Driven Development

Observability is not bolted on after development. It is a design-time architectural concern:

- **Developers instrument code with TraceIDs and metrics during the design phase**, not as a post-deployment afterthought.
- **Failing traces are test failures.** A missing span or lost TraceID in a System Test is not an instrumentation TODO — it is a functional defect.
- **Traces replace grep-and-pray debugging.** A developer can click from a failing log directly into a distributed trace that shows exactly which service, function, and line caused the issue.

---

## Shifting Observability Left

### What This Means in Practice

1. **Design Phase:** When designing a new feature, identify which spans, attributes, and metrics the feature must emit. Document these in the design specification alongside the API contract.
2. **Implementation Phase:** Instrument code with OpenTelemetry SDKs as the feature is built, not after. Ensure `context.Context` (Go) or context managers (Python) propagate trace context through every layer.
3. **Test Phase:** Write test assertions that validate trace completeness, span attributes, and parent-child relationships. These assertions are as important as response body assertions.
4. **Production Phase:** The same instrumentation that powers test assertions provides production observability. No "test-only" instrumentation.

### Instrumentation Standards

**Go:**
```go
func (s *MeetingService) Create(ctx context.Context, req CreateMeetingRequest) (*Meeting, error) {
    ctx, span := otel.Tracer("meeting-service").Start(ctx, "MeetingService.Create")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("meeting.title", req.Title),
        attribute.String("user.id", req.UserID),
    )
    
    // Business logic — context flows through all internal calls
    meeting, err := s.repo.Save(ctx, req)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }
    
    return meeting, nil
}
```

**Python:**
```python
from opentelemetry import trace

tracer = trace.get_tracer("transcription-service")

async def transcribe(self, ctx: Context, audio: AudioInput) -> TranscriptionResult:
    with tracer.start_as_current_span("TranscriptionService.transcribe") as span:
        span.set_attribute("audio.duration_ms", audio.duration_ms)
        span.set_attribute("audio.format", audio.format)
        
        result = await self.codec.convert(audio)
        # Span automatically captures timing and propagates context
        return result
```

---

## Trace-Driven Test Architecture

### System Test Trace Validation

System Tests must validate the completeness and correctness of distributed traces. The architecture looks like this:

```
┌──────────────────────────────────────────────────────┐
│ Test Runner                                          │
│  1. Triggers action (HTTP request to Service A)      │
│  2. Captures TraceID from response headers           │
│  3. Waits for trace propagation                      │
│  4. Queries trace backend                            │
│  5. Asserts trace completeness                       │
├──────────────────────────────────────────────────────┤
│ Service Cluster                                      │
│  Service A → Service B → Service C                   │
│  (each emits spans with traceparent propagation)     │
├──────────────────────────────────────────────────────┤
│ OTel Collector → Trace Backend                       │
│  (Jaeger, Tempo, or in-memory exporter)              │
└──────────────────────────────────────────────────────┘
```

### What to Assert

| Assertion Type | Description | Example |
|----------------|-------------|---------|
| **Span Count** | Expected number of spans in the trace | "Meeting creation should produce exactly 5 spans" |
| **Span Names** | Specific operations were executed | "Trace must include `MeetingService.Create` and `UserRepository.FindByID`" |
| **Parent-Child Relationships** | Correct calling hierarchy | "`Repository.Save` must be a child of `Service.Create`" |
| **Cross-Service Propagation** | TraceID survives service boundaries | "TraceID in Service A's response must appear in Service B's spans" |
| **Error Propagation** | Failures are correctly recorded | "If Service B errors, the span must have `status=ERROR` and `error.message` attribute" |
| **Attribute Correctness** | Business data is attached to spans | "`meeting.id` attribute must match the created meeting's ID" |
| **Duration Thresholds** | Performance budgets are met | "Database span duration must be < 500ms" |

---

## Implementing Trace Assertions

### Option 1: Direct Trace Backend Query

Query the trace backend (Jaeger, Tempo) via its API after test execution:

```go
func assertTraceComplete(t *testing.T, traceID string) {
    t.Helper()
    
    // Poll with timeout — traces are async
    var trace *JaegerTrace
    require.Eventually(t, func() bool {
        resp, err := http.Get(fmt.Sprintf("http://jaeger:16686/api/traces/%s", traceID))
        if err != nil || resp.StatusCode != 200 {
            return false
        }
        trace, _ = parseJaegerResponse(resp.Body)
        return trace != nil && len(trace.Spans) >= 5
    }, 15*time.Second, 500*time.Millisecond, "trace did not complete within timeout")
    
    // Assert span hierarchy
    spans := trace.SpansByName()
    assert.Contains(t, spans, "MeetingService.Create")
    assert.Contains(t, spans, "MeetingRepository.Save")
    
    // Assert parent-child
    saveSpan := spans["MeetingRepository.Save"]
    createSpan := spans["MeetingService.Create"]
    assert.Equal(t, createSpan.SpanID, saveSpan.ParentSpanID)
    
    // Assert no errors
    for _, span := range trace.Spans {
        assert.NotEqual(t, "ERROR", span.Status, 
            "span %s has error status: %s", span.OperationName, span.ErrorMessage)
    }
}
```

### Option 2: In-Memory Exporter (Unit/Service Tests)

For Service Tests, use an in-memory span exporter to capture and inspect spans without an external backend:

```go
exporter := tracetest.NewInMemoryExporter()
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSyncer(exporter),
)
otel.SetTracerProvider(tp)

// Run the code under test
service.Create(ctx, request)

// Assert spans
spans := exporter.GetSpans()
require.Len(t, spans, 3)
assert.Equal(t, "MeetingService.Create", spans[0].Name)
```

### Option 3: Tracetest (Dedicated Framework)

Tracetest is a purpose-built framework for trace-based testing. It integrates with existing test runners and allows visual or YAML-based trace assertions:

```yaml
# tracetest.yaml
type: Test
spec:
  name: Meeting Creation Trace Validation
  trigger:
    type: http
    httpRequest:
      url: http://localhost:8080/api/meetings
      method: POST
      body: '{"title": "Test Meeting"}'
  specs:
    - selector: span[name="MeetingService.Create"]
      assertions:
        - attr:tracetest.span.duration < 1000ms
    - selector: span[name="MeetingRepository.Save"]
      assertions:
        - attr:db.system = "postgresql"
    - selector: span[name="WebSocket.Broadcast"]
      assertions:
        - attr:messaging.destination = "meetings"
```

---

## Context Propagation Validation

### The Critical Path

Trace context (`traceparent` header) must propagate correctly through every boundary:

```
HTTP Request (traceparent header)
  → API Handler (extracts context)
    → Service Layer (passes context.Context)
      → Database Call (context flows to driver)
      → HTTP Client Call to Service B (injects traceparent)
        → Service B Handler (extracts context)
          → Pub/Sub Publish (embeds trace context in message metadata)
            → Consumer (extracts context from message)
```

### Common Propagation Failures

| Failure Mode | Symptom | Root Cause |
|-------------|---------|------------|
| Broken trace | New TraceID appears mid-chain | Context was not passed to an outgoing HTTP call |
| Orphan spans | Spans exist but have no parent | `context.Background()` used instead of the request context |
| Missing spans | Expected span is absent | Function was not instrumented or errored silently |
| Wrong parent | Span appears under wrong service | Message metadata did not carry trace context |

### Validation Checklist

- [ ] Every HTTP handler extracts `traceparent` from incoming request headers
- [ ] Every outgoing HTTP call injects `traceparent` into request headers
- [ ] Every Pub/Sub publish embeds trace context in message metadata
- [ ] Every Pub/Sub consumer extracts trace context from message metadata
- [ ] `context.Context` (Go) or equivalent is the first parameter of every function call
- [ ] No function uses `context.Background()` when a request context is available
- [ ] WebSocket messages carry trace context within the CloudEvents envelope

---

## Continuous Profiling with eBPF

### What It Is

eBPF (extended Berkeley Packet Filter) enables kernel-level instrumentation with near-zero overhead. Continuous profiling agents run in the kernel, sampling stack traces and correlating performance data with specific lines of code — without any application code changes.

### How It Works

1. **eBPF programs** hook into kernel events (CPU scheduling, syscalls, memory allocation)
2. **Stack traces** are sampled at regular intervals (e.g., 99 Hz)
3. **In-kernel aggregation** via eBPF maps prevents overhead from per-event streaming
4. **User-space agents** (Parca, Pyroscope) read aggregated data and produce flame graphs

### Integration with Testing

| Context | Practice |
|---------|----------|
| **System Tests** | Run eBPF profiling agents alongside the test cluster. After test completion, compare flame graphs against baseline to detect performance regressions. |
| **CI/CD Pipeline** | Capture profiling snapshots during integration tests. Fail the pipeline if CPU or memory usage exceeds defined budgets for critical code paths. |
| **Production Monitoring** | Continuous profiling in production provides a persistent signal for slow regression detection — performance drift that accumulates over weeks. |

### Recommended Tools

| Tool | Description |
|------|-------------|
| **Parca** | Open-source continuous profiling with eBPF agent. Stores profiles as time series for historical comparison. |
| **Grafana Pyroscope** | Continuous profiling integrated with the Grafana observability stack. |
| **Elastic Universal Profiling** | eBPF-based profiler integrated with the Elastic stack. |

### Performance Budget Assertion Example

```go
func TestCreateMeeting_PerformanceBudget(t *testing.T) {
    // Start profiling capture
    profile := startCPUProfile(t)
    
    // Run the operation under load
    for i := 0; i < 1000; i++ {
        service.Create(ctx, validRequest)
    }
    
    // Assert performance budget
    report := profile.Stop()
    assert.Less(t, report.P99Latency, 50*time.Millisecond,
        "P99 latency exceeds budget — check flame graph at %s", report.FlameGraphURL)
    assert.Less(t, report.PeakMemoryMB, 256,
        "Peak memory exceeds budget")
}
```

---

## See Also

- `emulation-strategy.md` — Configuring OTel Collector containers
- `validation-layers.md` — Which test types require trace validation
- `shift-right-testing.md` — Trace-based production assertions
