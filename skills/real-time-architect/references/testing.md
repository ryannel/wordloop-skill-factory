# Testing & Validation

Reference for validating real-time APIs at every stage — from contract compliance to production load. Persistent connections require specialized testing strategies that go beyond traditional request-response patterns.

## Table of Contents

- [Contract Testing](#contract-testing)
- [Connection Lifecycle Testing](#connection-lifecycle-testing)
- [Load Testing](#load-testing)
- [Consumer-Driven Contracts](#consumer-driven-contracts)
- [CI/CD Integration](#cicd-integration)

---

## Contract Testing

### AsyncAPI Contract Validation

The AsyncAPI spec is not just documentation — it is an executable contract. Validate it in CI to catch breaking changes before deployment:

```bash
# Validate spec syntax and references
npx @asyncapi/cli validate asyncapi.yaml

# Generate documentation
npx @asyncapi/cli generate fromTemplate asyncapi.yaml @asyncapi/html-template -o docs/

# Generate typed server stubs
npx @asyncapi/cli generate fromTemplate asyncapi.yaml @asyncapi/nodejs-template -o src/
```

### Specmatic Contract Testing

Specmatic treats the AsyncAPI spec as the source of truth and automatically generates tests from it:

```bash
# Install Specmatic
npm install -D specmatic

# Run contract tests against a running server
npx specmatic test --spec asyncapi.yaml --host localhost --port 3000
```

### Runtime Schema Validation

Every incoming message must be validated against the schema defined in the AsyncAPI contract. Failures are a signal of drift between producer and consumer:

```javascript
import Ajv from "ajv";
import addFormats from "ajv-formats";

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

// Load schemas from AsyncAPI spec at startup
const messageSchemas = loadSchemasFromAsyncAPI("./asyncapi.yaml");

function validateMessage(channelName, message) {
  const validate = ajv.compile(messageSchemas[channelName]);
  const valid = validate(message);

  if (!valid) {
    return {
      valid: false,
      errors: validate.errors.map((e) => ({
        path: e.instancePath,
        message: e.message,
        params: e.params,
      })),
    };
  }

  return { valid: true, errors: [] };
}

// In the message handler
socket.on("message", (event) => {
  const result = validateMessage("document-updates", event);

  if (!result.valid) {
    socket.emit("error", {
      type: "https://api.example.com/errors/validation",
      title: "Schema Validation Error",
      status: 422,
      detail: result.errors,
    });
    // Track as a golden signal (schema validation failures)
    metrics.schemaValidationFailures.add(1);
    return;
  }

  processEvent(event);
});
```

---

## Connection Lifecycle Testing

Real-time APIs have a connection lifecycle that must be tested explicitly. Unlike REST, a test must verify behavior across connect, message exchange, disconnection, and reconnection.

### Handshake & Authentication

```javascript
import { io } from "socket.io-client";

describe("Connection Lifecycle", () => {
  test("rejects unauthenticated connections", (done) => {
    const socket = io("ws://localhost:3000", {
      auth: {}, // No token
    });

    socket.on("connect_error", (err) => {
      expect(err.message).toBe("Authentication required");
      socket.close();
      done();
    });
  });

  test("accepts valid JWT and establishes session", (done) => {
    const socket = io("ws://localhost:3000", {
      auth: { token: validJWT },
    });

    socket.on("connect", () => {
      expect(socket.connected).toBe(true);
      socket.close();
      done();
    });
  });

  test("disconnects on expired token during session", (done) => {
    const socket = io("ws://localhost:3000", {
      auth: { token: soonExpiringJWT },
    });

    socket.on("auth-expired", ({ reason }) => {
      expect(reason).toBe("Session invalidated");
      socket.close();
      done();
    });
  });
});
```

### Reconnection Behavior

```javascript
describe("Reconnection", () => {
  test("reconnects with exponential backoff after server drop", async () => {
    const socket = io("ws://localhost:3000", {
      reconnectionDelay: 100,
      reconnectionDelayMax: 2000,
      randomizationFactor: 0.5,
    });

    await waitForConnect(socket);

    // Simulate server-side disconnect
    const reconnectTimes = [];
    socket.on("reconnect_attempt", () => {
      reconnectTimes.push(Date.now());
    });

    await simulateServerDrop();
    await waitForReconnect(socket);

    // Verify backoff: each interval should be longer than the previous
    for (let i = 1; i < reconnectTimes.length; i++) {
      const gap = reconnectTimes[i] - reconnectTimes[i - 1];
      expect(gap).toBeGreaterThan(50); // Not instant
    }

    socket.close();
  });

  test("drains offline queue on reconnection", async () => {
    const socket = io("ws://localhost:3000");
    await waitForConnect(socket);

    // Queue messages while disconnected
    socket.disconnect();
    socket.emit("action", { type: "save", id: "1" });
    socket.emit("action", { type: "save", id: "2" });

    // Reconnect and verify queued messages were delivered
    socket.connect();
    await waitForConnect(socket);

    const serverReceived = await getServerReceivedActions();
    expect(serverReceived).toContainEqual({ type: "save", id: "1" });
    expect(serverReceived).toContainEqual({ type: "save", id: "2" });

    socket.close();
  });
});
```

### Graceful Shutdown

```javascript
describe("Graceful Shutdown", () => {
  test("server sends 1001 close code during drain", async () => {
    const socket = io("ws://localhost:3000");
    await waitForConnect(socket);

    const drainPromise = new Promise((resolve) => {
      socket.on("server-draining", ({ code, retryAfterMs }) => {
        expect(code).toBe(1001);
        expect(retryAfterMs).toBeGreaterThan(0);
        resolve();
      });
    });

    // Send SIGTERM to the server
    await sendSigterm(serverPid);
    await drainPromise;

    socket.close();
  });
});
```

---

## Load Testing

### k6 WebSocket Load Test

```javascript
import ws from "k6/ws";
import { check, sleep } from "k6";
import { Counter, Trend } from "k6/metrics";

// Custom metrics
const messageLatency = new Trend("ws_message_latency", true);
const messagesReceived = new Counter("ws_messages_received");

export const options = {
  scenarios: {
    sustained_load: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [
        { duration: "30s", target: 100 },   // Ramp to 100 connections
        { duration: "2m",  target: 100 },   // Sustain
        { duration: "30s", target: 500 },   // Spike to 500
        { duration: "1m",  target: 500 },   // Sustain spike
        { duration: "30s", target: 0 },     // Ramp down
      ],
    },
  },
  thresholds: {
    ws_message_latency: ["p(95)<100"],       // 95th percentile < 100ms
    ws_messages_received: ["count>1000"],     // At least 1000 messages received
    ws_connecting: ["avg<500"],               // Connection time < 500ms avg
  },
};

export default function () {
  const url = "wss://staging.example.com/socket.io/?EIO=4&transport=websocket";
  const params = {
    headers: { Authorization: `Bearer ${__ENV.TEST_TOKEN}` },
  };

  const response = ws.connect(url, params, function (socket) {
    socket.on("open", () => {
      // Join a test room
      socket.send(JSON.stringify({
        type: "join",
        room: `load-test-room-${__VU % 10}`,
      }));

      // Send messages at realistic intervals
      socket.setInterval(() => {
        const sendTime = Date.now();
        socket.send(JSON.stringify({
          type: "message",
          text: `Load test message from VU ${__VU}`,
          timestamp: sendTime,
        }));
      }, 2000); // 1 message every 2 seconds per VU
    });

    socket.on("message", (data) => {
      const msg = JSON.parse(data);
      messagesReceived.add(1);

      if (msg.timestamp) {
        messageLatency.add(Date.now() - msg.timestamp);
      }
    });

    socket.on("close", () => {
      // Connection closed
    });

    // Keep connection alive for the test duration
    socket.setTimeout(() => {
      socket.close();
    }, 120000); // 2 minutes
  });

  check(response, {
    "WebSocket handshake successful": (r) => r && r.status === 101,
  });
}
```

### Key Metrics to Track

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| `ws_connecting` | Time to establish WebSocket connection | < 500ms avg |
| `ws_session_duration` | Total time the connection stayed open | Matches test duration |
| `ws_msgs_sent` / `ws_msgs_received` | Message throughput | Matches expected rate |
| `ws_message_latency` (custom) | Round-trip message latency | < 100ms p95 |
| Server CPU/Memory | Resource consumption under load | CPU < 80%, Memory < 85% |

### Load Testing Best Practices

1. **Start with smoke tests** — 5-10 VUs to verify connectivity before scaling
2. **Use realistic load profiles** — Simulate actual user behavior (idle periods, message bursts)
3. **Test thundering herd** — Drop all connections simultaneously, verify backoff prevents cascade
4. **Test at 2x expected peak** — Your system should survive double the peak with degraded performance
5. **Monitor server-side** — k6 metrics only show client perspective; correlate with server Prometheus metrics

---

## Consumer-Driven Contracts

In microservice architectures, multiple consumers subscribe to the same event channels. Consumer-driven contracts (CDC) ensure that producer changes don't silently break downstream consumers.

### Workflow

```
┌──────────────┐   publishes contract   ┌──────────────┐
│   Consumer A │ ─────────────────────> │  Pact Broker │
│   (Frontend) │                        │              │
└──────────────┘                        │              │
                                        │              │
┌──────────────┐   publishes contract   │              │
│   Consumer B │ ─────────────────────> │              │
│   (Mobile)   │                        └──────┬───────┘
└──────────────┘                               │
                                               │ verifies against
                                               │
                                        ┌──────▼───────┐
                                        │   Provider   │
                                        │  (RT Server) │
                                        └──────────────┘
```

### Pact Consumer Test

```javascript
import { PactV4 } from "@pact-foundation/pact";

describe("Document Update Consumer Contract", () => {
  const pact = new PactV4({
    consumer: "frontend-app",
    provider: "realtime-server",
  });

  test("expects document updates with version and operations", async () => {
    await pact
      .addInteraction()
      .given("document doc_123 exists")
      .uponReceiving("a document update event")
      .withRequestLike({
        channel: "document-updates",
        message: {
          documentId: "doc_123",
          version: 42,
          operations: [
            { type: "INSERT", path: "/content", value: "Hello" },
          ],
          updatedBy: { id: "user_456", name: "Alice" },
        },
      })
      .toPactFile();
  });
});
```

### Provider Verification

```javascript
// In the provider's CI pipeline
import { Verifier } from "@pact-foundation/pact";

const verifier = new Verifier({
  providerBaseUrl: "http://localhost:3000",
  pactBrokerUrl: process.env.PACT_BROKER_URL,
  provider: "realtime-server",
  publishVerificationResult: true,
  providerVersion: process.env.GIT_SHA,
});

await verifier.verifyProvider();
```

### Can-I-Deploy Gate

```bash
# In CI before deployment — blocks if any consumer contract would break
npx pact-broker can-i-deploy \
  --pacticipant realtime-server \
  --version $(git rev-parse HEAD) \
  --to-environment production
```

---

## CI/CD Integration

### Breaking Change Detection Pipeline

```yaml
# .github/workflows/schema-validation.yml
name: Schema Validation

on:
  pull_request:
    paths:
      - "asyncapi.yaml"
      - "schemas/**"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate AsyncAPI spec
        run: npx @asyncapi/cli validate asyncapi.yaml

      - name: Check backward compatibility
        run: |
          # Compare current spec against main branch
          git fetch origin main
          git show origin/main:asyncapi.yaml > /tmp/asyncapi-main.yaml

          # Run compatibility check
          npx @asyncapi/diff /tmp/asyncapi-main.yaml asyncapi.yaml \
            --fail-on breaking

      - name: Run contract tests
        run: npx specmatic test --spec asyncapi.yaml

      - name: Verify consumer contracts
        run: |
          npx pact-broker can-i-deploy \
            --pacticipant realtime-server \
            --version ${{ github.sha }} \
            --to-environment staging
```

### Schema Registry Validation

For Kafka/event-sourced systems, validate schemas at the broker level:

```bash
# Register schema with compatibility check
curl -X POST \
  http://schema-registry:8081/subjects/document-updates-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schemaType": "PROTOBUF",
    "schema": "syntax = \"proto3\"; message DocumentUpdate { ... }"
  }'

# The registry rejects schemas that violate the configured compatibility mode
# (BACKWARD, FORWARD, FULL, NONE)
```
