# Resiliency & Operations

Deep reference for ensuring real-time systems degrade gracefully under failure. Persistent connections fail in silent, cascading ways that traditional REST monitoring will not catch.

For client-side resilience (offline queue, optimistic UI, reconnection), see `client.md`.

## Table of Contents

- [Backpressure & Flow Control](#backpressure--flow-control)
- [Circuit Breakers](#circuit-breakers)
- [Dead Letter Queues](#dead-letter-queues)
- [Graceful Degradation](#graceful-degradation)
- [Idempotency](#idempotency)
- [Distributed Rate Limiting](#distributed-rate-limiting)
- [Thundering Herd Mitigation](#thundering-herd-mitigation)
- [Graceful Shutdown](#graceful-shutdown)
- [Chaos Engineering](#chaos-engineering)
- [Connection Lifecycle](#connection-lifecycle)

---

## Backpressure & Flow Control

Backpressure prevents a fast producer from overwhelming a slow consumer. Without it, buffers grow unbounded until the process runs out of memory and crashes.

### WebTransport (Native QUIC Flow Control)

QUIC provides built-in flow control at the stream level. If the receiver's buffer is full, the QUIC layer automatically pauses transport-level data transmission:

```javascript
const stream = await transport.createBidirectionalStream();
const writer = stream.writable.getWriter();

// The write() call returns a promise that resolves when the data
// is accepted by the QUIC flow control window. If the receiver
// is slow, this naturally applies backpressure.
await writer.write(encodedData);
```

### WebSocketStream (Modern Pull Model)

Unlike the legacy WebSocket API which buffers indefinitely until a crash, `WebSocketStream` uses a pull model:

```javascript
const wss = new WebSocketStream("wss://api.example.com");
const { readable, writable } = await wss.opened;

const reader = readable.getReader();

// Pull-based: only reads when the app is ready
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  await processMessage(value); // Slow processing naturally throttles the sender
}
```

### Legacy WebSocket (Manual Monitoring)

For older clients, manually monitor `bufferedAmount`:

```javascript
const BUFFER_THRESHOLD = 1024 * 1024; // 1MB safety limit
let paused = false;

function safeSend(ws, data) {
  if (ws.bufferedAmount > BUFFER_THRESHOLD) {
    if (!paused) {
      paused = true;
      console.warn(`Backpressure: buffer at ${ws.bufferedAmount} bytes, pausing`);
    }
    return new Promise((resolve) => {
      const check = setInterval(() => {
        if (ws.bufferedAmount < BUFFER_THRESHOLD / 2) {
          clearInterval(check);
          paused = false;
          ws.send(data);
          resolve();
        }
      }, 100);
    });
  }

  ws.send(data);
  return Promise.resolve();
}
```

### Server-Side Backpressure (Socket.IO)

```javascript
io.on("connection", (socket) => {
  const MAX_BUFFER = 10000;
  let pending = 0;

  const originalEmit = socket.emit.bind(socket);

  socket.emit = function (event, ...args) {
    pending++;

    if (pending > MAX_BUFFER) {
      console.warn("Buffer overflow — dropping message");
      return false;
    }

    const result = originalEmit(event, ...args);

    socket.once("drain", () => {
      pending = 0;
    });

    return result;
  };
});
```

---

## Circuit Breakers

When a real-time event handler depends on a downstream service (database, payment API, notification service), wrap those calls in a circuit breaker to prevent cascading failure.

### States

```
┌────────┐  errors exceed threshold  ┌────────┐
│ CLOSED │ ─────────────────────────>│  OPEN  │
│(normal)│                           │ (fail  │
│        │<──────────────────────────│  fast) │
└────────┘  success in half-open     └───┬────┘
                                         │ after timeout
                                    ┌────▼─────┐
                                    │HALF-OPEN  │
                                    │(test 1    │
                                    │ request)  │
                                    └───────────┘
```

### Implementation (Opossum)

```javascript
const CircuitBreaker = require("opossum");

// Wrap each downstream dependency with its own breaker
const dbBreaker = new CircuitBreaker(saveToDatabase, {
  timeout: 3000,                    // If DB takes > 3s, trigger failure
  errorThresholdPercentage: 50,     // Trip if 50% of calls fail
  resetTimeout: 30000,              // Wait 30s before trying again (Half-Open)
  volumeThreshold: 10,              // Min calls before tripping
});

// Fallback: push to DLQ when circuit is open
dbBreaker.fallback(async (event) => {
  console.error("Circuit open — routing to DLQ");
  await moveToDLQ(event, "circuit_open");
});

// Use in WebSocket handler
io.on("connection", (socket) => {
  socket.on("action", async (event) => {
    try {
      const result = await dbBreaker.fire(event);
      socket.emit("ack", { idempotencyKey: event.idempotencyKey });
    } catch (err) {
      // Already handled by fallback
    }
  });
});

// Monitor circuit state
dbBreaker.on("open", () => metrics.circuitState.set("open"));
dbBreaker.on("halfOpen", () => metrics.circuitState.set("half-open"));
dbBreaker.on("close", () => metrics.circuitState.set("closed"));
```

### Granular Breakers

Use separate breakers for each downstream dependency. One failing service should not block access to healthy services:

```javascript
const breakers = {
  userDb: new CircuitBreaker(queryUserDb, { timeout: 2000 }),
  paymentApi: new CircuitBreaker(callPaymentApi, { timeout: 5000 }),
  notificationSvc: new CircuitBreaker(sendNotification, { timeout: 3000 }),
};
```

---

## Dead Letter Queues

When messages fail processing after bounded retries, isolate them in a DLQ rather than dropping silently or retrying infinitely.

### Implementation

```javascript
const MAX_RETRIES = 3;

async function processWithRetry(event) {
  let attempt = 0;

  while (attempt < MAX_RETRIES) {
    try {
      await processEvent(event);
      return; // Success
    } catch (err) {
      attempt++;

      if (isTerminalError(err)) {
        // Terminal errors (validation, auth) go straight to DLQ
        await moveToDLQ(event, err.message);
        return;
      }

      if (attempt < MAX_RETRIES) {
        // Transient errors get exponential backoff
        await sleep(Math.pow(2, attempt) * 100);
      }
    }
  }

  // Max retries exhausted — move to DLQ
  await moveToDLQ(event, "max_retries_exhausted");
}

function isTerminalError(err) {
  return err.code === "VALIDATION_ERROR" ||
         err.code === "AUTH_FAILED" ||
         err.code === "SCHEMA_MISMATCH";
}

async function moveToDLQ(event, reason) {
  await redis.lpush("dlq:events", JSON.stringify({
    originalEvent: event,
    failedAt: new Date().toISOString(),
    reason,
    metadata: {
      originalChannel: event.metadata?.type,
      producerId: event.metadata?.source,
      retryCount: MAX_RETRIES,
    },
  }));

  metrics.dlqDepth.add(1);
}
```

### DLQ Monitoring

A DLQ is useless if it grows silently. Monitor depth and velocity:

```yaml
# Prometheus alerting
- alert: DLQGrowing
  expr: rate(dlq_events_total[5m]) > 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "DLQ receiving events at > 6/min — investigate failing consumer"
```

### Redrive Strategy

After fixing the root cause, replay DLQ messages:

```javascript
async function redriveDLQ(batchSize = 100) {
  const events = await redis.lrange("dlq:events", 0, batchSize - 1);

  for (const raw of events) {
    const { originalEvent } = JSON.parse(raw);
    try {
      await processEvent(originalEvent);
      await redis.lrem("dlq:events", 1, raw);
    } catch (err) {
      console.error(`Redrive failed for ${originalEvent.metadata?.id}:`, err);
      // Leave in DLQ for manual inspection
    }
  }
}
```

---

## Graceful Degradation

When downstream dependencies fail, maintain core functionality by falling back to lower-fidelity alternatives.

### Fallback Strategies

| Strategy | When to Use | Example |
|----------|-------------|---------|
| **Cached data** | Database or API unavailable | Serve last-known user profile from Redis |
| **Simplified logic** | Expensive feature overloading | Disable real-time suggestions, serve static |
| **Partial processing** | Non-critical enrichment failing | Skip analytics tagging, deliver raw event |
| **Default responses** | Personalization service down | Show generic recommendations |

### Feature Flag Integration

Use feature flags to toggle degraded modes without redeployment:

```javascript
import { getFlag } from "./feature-flags"; // LaunchDarkly, Unleash, etc.

io.on("connection", (socket) => {
  socket.on("message", async (event) => {
    // Core delivery always works
    io.to(event.roomId).emit("message", event);

    // Non-critical features are flag-gated
    if (getFlag("enable-typing-indicators")) {
      await broadcastTypingIndicator(event);
    }

    if (getFlag("enable-read-receipts")) {
      await trackReadReceipt(event);
    }

    // When the analytics service is failing, the flag is toggled off
    // rather than rolling back the entire deployment
    if (getFlag("enable-analytics")) {
      await analytics.track(event); // Non-critical
    }
  });
});
```

---

## Idempotency

Network jitter, retries, and reconnections cause duplicate message delivery. Every mutation event must include an idempotency key in its CloudEvents envelope.

### The "Act-If-Not-Exists" Pattern

```javascript
async function processWithIdempotency(event) {
  const { key: idempotencyKey } = event.metadata.idempotency;

  try {
    // 1. Attempt atomic insert of the idempotency key
    await db.query(
      `INSERT INTO idempotency_keys (key, created_at, expires_at)
       VALUES ($1, NOW(), $2)`,
      [idempotencyKey, event.metadata.idempotency.expiresAt]
    );
  } catch (err) {
    if (err.constraint === "idempotency_keys_pkey") {
      // 2. Duplicate — return cached response
      const cached = await db.query(
        "SELECT response FROM idempotency_keys WHERE key = $1",
        [idempotencyKey]
      );
      return cached.rows[0].response;
    }
    throw err;
  }

  // 3. First time — execute business logic
  const result = await executeBusinessLogic(event.data);

  // 4. Cache the response for future duplicate checks
  await db.query(
    "UPDATE idempotency_keys SET response = $1 WHERE key = $2",
    [JSON.stringify(result), idempotencyKey]
  );

  return result;
}
```

### Sequence Coordination

For protocols handling interleaved parallel streams (WebTransport), out-of-order messages are common. Use producer-side sequencing:

| Header | Purpose |
|--------|---------|
| `Producer-Id` | Stable client identifier (survives reconnection) |
| `Producer-Epoch` | Integer incremented on client restart (invalidates old delayed packets) |
| `Producer-Seq` | Monotonically increasing sequence number per epoch |

```javascript
const producerState = new Map();

function validateSequence(msg) {
  const { producerId, producerEpoch, producerSeq } = msg.metadata;
  const state = producerState.get(producerId);

  // New producer or new epoch — accept and initialize
  if (!state || producerEpoch > state.epoch) {
    producerState.set(producerId, { epoch: producerEpoch, seq: producerSeq });
    return { valid: true, action: "accept" };
  }

  // Same epoch, higher sequence — accept
  if (producerEpoch === state.epoch && producerSeq > state.seq) {
    state.seq = producerSeq;
    return { valid: true, action: "accept" };
  }

  // Old epoch or duplicate/lower sequence — reject
  return { valid: false, action: "reject", reason: "stale_or_duplicate" };
}
```

---

## Distributed Rate Limiting

In-memory rate limiting fails at scale because requests get distributed across pods. Use a centralized approach.

### GCRA (Generic Cell Rate Algorithm)

GCRA handles bursts more gracefully than token bucket. It models time allowances rather than counting tokens:

```javascript
import Redis from "ioredis";
const redis = new Redis(process.env.REDIS_URL);

// GCRA via Lua script — atomic, < 2-3ms latency
const GCRA_SCRIPT = `
  local key = KEYS[1]
  local emission_interval = tonumber(ARGV[1])
  local burst = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])

  local tat = tonumber(redis.call('GET', key) or now)

  local new_tat = math.max(tat, now) + emission_interval
  local allow_at = new_tat - burst - emission_interval

  if now < allow_at then
    return {0, math.ceil(allow_at - now)}  -- rejected, retry-after ms
  end

  redis.call('SET', key, new_tat)
  redis.call('PEXPIRE', key, burst + emission_interval)

  return {1, 0}  -- allowed
`;

async function checkRate(userId, maxPerMinute = 100) {
  const emissionInterval = 60000 / maxPerMinute;
  const burst = emissionInterval * 10;

  const [allowed, retryAfter] = await redis.eval(
    GCRA_SCRIPT, 1, `rate:${userId}`, emissionInterval, burst, Date.now()
  );

  return { allowed: allowed === 1, retryAfterMs: retryAfter };
}

// Enforce at the gateway layer
io.use(async (socket, next) => {
  const { allowed, retryAfterMs } = await checkRate(socket.data.user.id);
  if (!allowed) {
    return next(new Error(`Rate limited. Retry after ${retryAfterMs}ms`));
  }
  next();
});
```

---

## Thundering Herd Mitigation

When a server restarts or a network partition recovers, every disconnected client attempts to reconnect simultaneously. See `client.md` for the client-side backoff implementation.

### Load Testing Requirements

Before production, simulate mass disconnection:

- **Target:** 100,000 concurrent client drops with simultaneous reconnection
- **Measure:** Peak CPU at ingress, P99 reconnection latency, failed connection rate
- **Pass criteria:** Backoff + jitter prevents CPU spikes above 80% at the ingress layer
- **Tool:** Use k6 or Artillery with WebSocket protocol support (see `testing.md`)

---

## Graceful Shutdown

Prevent connection drops during deployments:

```javascript
function gracefulShutdown() {
  console.log("Received SIGTERM — entering drain state");

  // 1. Stop accepting new connections
  httpServer.close();

  // 2. Notify all connected clients
  io.sockets.sockets.forEach((socket) => {
    socket.emit("server-draining", {
      code: 1001,
      message: "Server restarting — please reconnect",
      retryAfterMs: 5000,
    });
  });

  // 3. Allow time for in-flight operations
  setTimeout(() => {
    // 4. Close all remaining connections
    io.close(() => {
      console.log("All connections closed cleanly");
      process.exit(0);
    });
  }, 10000); // 10-second drain window

  // 5. Force exit if drain exceeds timeout
  setTimeout(() => {
    console.error("Forcing shutdown after timeout");
    process.exit(1);
  }, 30000);
}

process.on("SIGTERM", gracefulShutdown);
process.on("SIGINT", gracefulShutdown);
```

### Kubernetes preStop Hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]  # Wait for LB deregistration
terminationGracePeriodSeconds: 45       # Total shutdown budget
```

The `preStop` delay ensures the load balancer removes the pod from its rotation before the application starts draining. Without this, new connections may arrive at a draining pod.

---

## Chaos Engineering

Persistent systems fail in ways that REST services do not. Proactive fault injection is mandatory.

### Fault Injection Scenarios

| Scenario | What to Test | Expected Behavior |
|----------|-------------|-------------------|
| **Network partition** | Inject via service mesh (Istio fault injection) | Heartbeat detects half-open connections within 30s; clients reconnect with backoff |
| **Pod termination** | Kill random pods during peak load | Graceful drain sends `1001`, clients reconnect to healthy pods |
| **Backplane failure** | Disable Redis/NATS | Local delivery continues; cross-pod messages queue until backplane recovers |
| **Certificate expiry** | Rotate TLS certs mid-session | Active sessions remain on old cert; new connections use the new cert |
| **DNS failure** | Block DNS resolution | Clients use cached connections; reconnection fails gracefully with backoff |
| **Circuit breaker trip** | Kill a downstream dependency | Circuit opens, events route to DLQ, core delivery unaffected |

### Half-Open Connection Detection

Application-level heartbeat with a 30-second ping/pong cycle:

```javascript
const HEARTBEAT_INTERVAL = 30000;
const HEARTBEAT_TIMEOUT = 10000;

io.on("connection", (socket) => {
  let heartbeatTimer;
  let timeoutTimer;

  function scheduleHeartbeat() {
    heartbeatTimer = setTimeout(() => {
      socket.emit("ping");
      timeoutTimer = setTimeout(() => {
        console.warn(`Half-open connection detected: ${socket.id}`);
        socket.disconnect(true);
      }, HEARTBEAT_TIMEOUT);
    }, HEARTBEAT_INTERVAL);
  }

  socket.on("pong", () => {
    clearTimeout(timeoutTimer);
    scheduleHeartbeat();
  });

  scheduleHeartbeat();

  socket.on("disconnect", () => {
    clearTimeout(heartbeatTimer);
    clearTimeout(timeoutTimer);
  });
});
```

---

## Connection Lifecycle

### Connection Limits

```javascript
const MAX_CONNECTIONS = 10000;

io.engine.on("connection", (rawSocket) => {
  if (io.engine.clientsCount > MAX_CONNECTIONS) {
    rawSocket.close(1008, "Server at capacity");
    return;
  }
});
```

### Connection Cleanup

Every connection must clean up on disconnect:

1. **Presence records** — Remove from the routing directory
2. **Room membership** — Leave all rooms (Socket.IO handles this automatically)
3. **In-flight timers** — Clear heartbeat intervals and timeout callbacks
4. **Rate limit state** — Free rate limiter entries
5. **Backplane subscriptions** — Unsubscribe from per-user channels
6. **Circuit breaker metrics** — Decrement active connection counts
