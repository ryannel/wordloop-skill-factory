# Observability & Tracing

Deep reference for instrumenting persistent connections with distributed tracing, operational monitoring, and compliance logging. Traditional HTTP metrics are useless for long-lived streams — this reference defines the correct observability strategy.

## Table of Contents

- [W3C Trace Context in Streams](#w3c-trace-context-in-streams)
- [Redefined Golden Signals](#redefined-golden-signals)
- [OpenTelemetry Integration](#opentelemetry-integration)
- [Connection Metrics](#connection-metrics)
- [Alert Thresholds](#alert-thresholds)
- [Audit Logging](#audit-logging)

---

## W3C Trace Context in Streams

HTTP requests naturally carry trace headers (`traceparent`, `tracestate`) in HTTP headers. Persistent connections carry a single initial handshake, but then exchange thousands of messages — each needing its own trace context.

### Message-Level Propagation

Embed `traceparent` and `tracestate` inside the CloudEvents envelope, not in transport headers:

```javascript
import { trace, context, SpanKind } from "@opentelemetry/api";

const tracer = trace.getTracer("realtime-server");

io.on("connection", (socket) => {
  socket.on("message", (event) => {
    // Extract trace context from the CloudEvents envelope
    const { traceparent, tracestate } = event.metadata.tracing;

    // Restore the parent context
    const parentContext = propagation.extract(context.active(), {
      traceparent,
      tracestate,
    });

    // Create a child span for processing this message
    const span = tracer.startSpan(
      `process ${event.metadata.type}`,
      {
        kind: SpanKind.CONSUMER,
        attributes: {
          "messaging.system": "websocket",
          "messaging.operation": "process",
          "messaging.message.id": event.metadata.id,
          "cloudevents.type": event.metadata.type,
          "cloudevents.source": event.metadata.source,
        },
      },
      parentContext
    );

    try {
      processEvent(event);
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      span.recordException(err);
    } finally {
      span.end();
    }
  });
});
```

### Client-Side Context Generation

```javascript
import { trace, context } from "@opentelemetry/api";

const tracer = trace.getTracer("realtime-client");

function sendTracedMessage(socket, type, data) {
  const span = tracer.startSpan(`send ${type}`, {
    kind: SpanKind.PRODUCER,
  });

  // Inject trace context into the CloudEvents envelope
  const carrier = {};
  propagation.inject(context.active(), carrier);

  const event = {
    metadata: {
      specversion: "1.0",
      id: crypto.randomUUID(),
      type,
      source: "/client/v1",
      time: new Date().toISOString(),
      tracing: {
        traceparent: carrier.traceparent,
        tracestate: carrier.tracestate,
      },
    },
    data,
  };

  socket.emit("message", event);
  span.end();
}
```

### Cross-Service Propagation

When a WebSocket message triggers downstream microservice calls, extract the trace context from the envelope and inject it into outgoing HTTP/gRPC requests:

```javascript
async function handleOrderEvent(event) {
  // Restore context from the WebSocket message
  const parentContext = propagation.extract(context.active(), event.metadata.tracing);

  // Downstream HTTP call inherits the trace
  await context.with(parentContext, async () => {
    await fetch("https://inventory.internal/reserve", {
      method: "POST",
      headers: {
        ...propagation.inject({}, context.active()), // Inject into HTTP headers
        "Content-Type": "application/json",
      },
      body: JSON.stringify(event.data),
    });
  });
}
```

---

## Redefined Golden Signals

Traditional HTTP golden signals (request rate, error rate, duration, saturation) do not apply to persistent connections. Use these stream-specific signals:

### Latency

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| **Handshake TTFB** | Time to First Byte at connection establishment | < 200ms |
| **Message Processing Latency** | Time from message receipt to processing completion | < 50ms (p95) |
| **End-to-End Latency** | Time from client send to server acknowledgment | < 100ms (p99) |
| **Backplane Latency** | Time for a message to traverse the backplane (Pod A → Pod B) | < 10ms (p95) |

### Errors

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| **Connection Drop Rate** | Percentage of connections closed abnormally | > 1% per minute |
| **Schema Validation Failures** | Messages rejected due to malformed payloads | > 0.1% of messages |
| **Authentication Failures** | Handshake rejections due to invalid/expired tokens | > 5% of attempts |
| **Backpressure Events** | Times `bufferedAmount` exceeded threshold | Any occurrence |

### Saturation

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| **Memory Per Pod** | RSS memory of the server process | > 80% of container limit |
| **Event Loop Lag** | Runtime loop blocked duration | > 50ms |
| **Connection Count Per Pod** | Active WebSocket/SSE/WT connections | > 80% of configured max |
| **Backplane Queue Depth** | Pending messages in the broker | > 10,000 messages |

---

## OpenTelemetry Integration

### Instrumentation Setup

```javascript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + "/v1/traces",
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + "/v1/metrics",
    }),
    exportIntervalMillis: 10000,
  }),
  serviceName: "realtime-server",
});

sdk.start();
```

### Custom Metrics

```javascript
import { metrics } from "@opentelemetry/api";

const meter = metrics.getMeter("realtime-server");

// Connection gauge
const connectionGauge = meter.createUpDownCounter("ws.connections.active", {
  description: "Active WebSocket connections",
  unit: "connections",
});

// Message throughput
const messageCounter = meter.createCounter("ws.messages.processed", {
  description: "Total messages processed",
  unit: "messages",
});

// Processing latency histogram
const latencyHistogram = meter.createHistogram("ws.message.processing_duration", {
  description: "Message processing duration",
  unit: "ms",
  advice: {
    explicitBucketBoundaries: [1, 5, 10, 25, 50, 100, 250, 500, 1000],
  },
});

// Event loop lag
const eventLoopLag = meter.createHistogram("ws.event_loop.lag", {
  description: "Event loop lag",
  unit: "ms",
});

// Monitor event loop
let lastCheck = process.hrtime.bigint();
setInterval(() => {
  const now = process.hrtime.bigint();
  const expected = 1000; // 1 second interval
  const actual = Number(now - lastCheck) / 1e6;
  const lag = Math.max(0, actual - expected);
  eventLoopLag.record(lag);
  lastCheck = now;
}, 1000);

// Usage in connection handler
io.on("connection", (socket) => {
  connectionGauge.add(1);

  socket.on("message", (event) => {
    const start = performance.now();

    processEvent(event);

    const duration = performance.now() - start;
    messageCounter.add(1, { "event.type": event.metadata.type });
    latencyHistogram.record(duration, { "event.type": event.metadata.type });
  });

  socket.on("disconnect", () => {
    connectionGauge.add(-1);
  });
});
```

---

## Connection Metrics

### Prometheus-Compatible Exposition

```javascript
import { register, Gauge, Counter, Histogram } from "prom-client";

// Expose metrics endpoint for scraping
app.get("/metrics", async (req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
});

const wsConnections = new Gauge({
  name: "ws_active_connections",
  help: "Number of active WebSocket connections",
  labelNames: ["transport", "namespace"],
});

const wsMessagesTotal = new Counter({
  name: "ws_messages_total",
  help: "Total WebSocket messages processed",
  labelNames: ["direction", "event_type"],
});

const wsMessageDuration = new Histogram({
  name: "ws_message_processing_seconds",
  help: "WebSocket message processing duration",
  labelNames: ["event_type"],
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1],
});
```

---

## Alert Thresholds

### Recommended Alerting Rules

```yaml
# Prometheus alerting rules for real-time systems
groups:
  - name: realtime-server
    rules:
      # Connection drop rate
      - alert: HighConnectionDropRate
        expr: rate(ws_disconnections_total{reason!="normal"}[5m]) / rate(ws_connections_total[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Connection drop rate exceeds 1%"

      # Event loop lag
      - alert: EventLoopBlocked
        expr: ws_event_loop_lag_seconds > 0.05
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Event loop blocked for >50ms"

      # Memory pressure
      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod memory usage above 85%"

      # Connection count approaching limit
      - alert: ConnectionCapacityHigh
        expr: ws_active_connections / ws_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection count at 80% of pod capacity"

      # Backplane latency
      - alert: BackplaneLatencyHigh
        expr: histogram_quantile(0.95, rate(backplane_message_duration_seconds_bucket[5m])) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Backplane p95 latency exceeds 10ms"
```

---

## Audit Logging

Log connection lifecycle events for security analysis and compliance. This data feeds security incident investigations and is often required for regulatory compliance.

### Connection Events

```javascript
import { createLogger, format, transports } from "winston";

const auditLog = createLogger({
  level: "info",
  format: format.combine(format.timestamp(), format.json()),
  transports: [
    new transports.File({ filename: "audit/websocket.log" }),
  ],
});

io.on("connection", (socket) => {
  auditLog.info("connection.established", {
    socketId: socket.id,
    userId: socket.data.user?.id,
    ip: socket.handshake.address,
    origin: socket.handshake.headers.origin,
    transport: socket.conn.transport.name,
  });

  socket.on("disconnect", (reason) => {
    auditLog.info("connection.closed", {
      socketId: socket.id,
      userId: socket.data.user?.id,
      reason,
      duration: Date.now() - socket.handshake.issued,
    });
  });
});
```

### What to Log

| Event | Fields | Purpose |
|-------|--------|---------|
| `connection.established` | userId, IP, origin, transport | Track who connects and from where |
| `connection.closed` | userId, reason, duration | Identify abnormal disconnections |
| `auth.failed` | IP, token hash, error type | Detect brute force / credential stuffing |
| `room.joined` / `room.left` | userId, roomId | Track access patterns for sensitive resources |
| `message.rejected` | userId, reason, event type | Track rate limiting and policy violations |
