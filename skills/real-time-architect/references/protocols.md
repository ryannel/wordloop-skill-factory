# Protocol Selection & Transports

Deep reference for selecting, implementing, and migrating between real-time transport protocols. Use the tiered requirement model to match protocol capabilities to system needs.

For internal service-to-service streaming (gRPC) and client-facing subscriptions (GraphQL), see `grpc-graphql.md`. For serialization formats and compression, see `serialization.md`.

## Table of Contents

- [SSE Implementation](#sse-implementation)
- [WebSocket Protocol](#websocket-protocol)
- [WebSocket vs Socket.IO](#websocket-vs-socketio)
- [WebTransport Architecture](#webtransport-architecture)
- [Migration Paths](#migration-paths)
- [Performance Benchmarks](#performance-benchmarks)
- [Industry Patterns](#industry-patterns)

---

## SSE Implementation

### When to Use

SSE is the correct default for server-to-client streaming. It provides automatic reconnection, native event typing, and zero infrastructure overhead compared to WebSockets. Over HTTP/3, SSE eliminates head-of-line blocking entirely.

- Live feeds and notification streams
- Financial tickers and price updates
- AI token delivery (LLM streaming responses)
- Sensor data and IoT telemetry
- Any case where the client only reads

### Server Implementation

```javascript
import express from "express";

const app = express();

app.get("/events", (req, res) => {
  // SSE headers
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.setHeader("X-Accel-Buffering", "no"); // Disable nginx buffering

  // Send initial connection event with CloudEvents envelope
  const connectEvent = {
    metadata: {
      specversion: "1.0",
      id: crypto.randomUUID(),
      type: "com.app.connection.established",
      source: "/events/v1",
      time: new Date().toISOString(),
    },
    data: { status: "connected" },
  };

  res.write(`event: connection\n`);
  res.write(`id: ${connectEvent.metadata.id}\n`);
  res.write(`data: ${JSON.stringify(connectEvent)}\n\n`);

  // Heartbeat to keep connection alive through proxies
  const heartbeat = setInterval(() => {
    res.write(`: heartbeat\n\n`);
  }, 15000);

  req.on("close", () => {
    clearInterval(heartbeat);
    res.end();
  });
});
```

### Client with Resume

```javascript
const eventSource = new EventSource("/events");

// The browser automatically sends Last-Event-ID on reconnection
eventSource.addEventListener("update", (event) => {
  const payload = JSON.parse(event.data);
  // event.lastEventId contains the resume cursor
  processUpdate(payload);
});

eventSource.addEventListener("error", () => {
  // EventSource auto-reconnects with exponential backoff
  // The retry directive from the server controls the base interval
  console.warn("SSE connection lost, auto-reconnecting...");
});
```

### SSE Text Format

```text
event: sensor_update
id: 10567
retry: 5000
data: {
data: "metadata": {
data: "specversion": "1.0",
data: "id": "evt_10567",
data: "type": "com.iot.sensor.reading",
data: "source": "/sensors/north-01",
data: "time": "2025-03-27T10:15:35Z"
data: },
data: "data": {
data: "temperature": 22.45,
data: "humidity": 45.2,
data: "battery": 88
data: }
data: }
```

Key SSE directives:
- `event:` — Named event type for `addEventListener` routing
- `id:` — Sets `Last-Event-ID` for automatic resume on reconnection
- `retry:` — Server-controlled reconnection interval (ms)
- `data:` — Payload (multi-line supported; each line must be prefixed)

---

## WebSocket Protocol

### Handshake

```
Client → Server: HTTP Upgrade Request
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13

Server → Client: HTTP 101 Switching Protocols
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
```

### Opcodes

```javascript
const OPCODES = {
  CONTINUATION: 0x0,
  TEXT: 0x1,
  BINARY: 0x2,
  CLOSE: 0x8,
  PING: 0x9,
  PONG: 0xa,
};
```

### Close Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 1000 | Normal Closure | Clean disconnect |
| 1001 | Going Away | Server shutting down (graceful drain) |
| 1002 | Protocol Error | Malformed frame received |
| 1008 | Policy Violation | Auth expired, session invalidated |
| 1009 | Message Too Big | Payload exceeds `maxPayload` |
| 1011 | Internal Server Error | Unexpected server failure |

### Ping/Pong Heartbeat

```javascript
import WebSocket from "ws";

const wss = new WebSocket.Server({ port: 8080 });

wss.on("connection", (ws) => {
  ws.isAlive = true;

  ws.on("pong", () => {
    ws.isAlive = true;
  });
});

// 30-second heartbeat cycle to detect half-open connections
const heartbeat = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

wss.on("close", () => clearInterval(heartbeat));
```

### Server Setup (Socket.IO)

```javascript
import { createServer } from "http";
import { Server } from "socket.io";

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: process.env.ALLOWED_ORIGIN, credentials: true },
  pingTimeout: 20000,
  pingInterval: 25000,
  maxHttpBufferSize: 1e6, // 1MB max message
});

io.on("connection", (socket) => {
  const { userId } = socket.data.user;

  socket.on("join-room", (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit("user-joined", { userId });
  });

  socket.on("message", ({ roomId, text }) => {
    io.to(roomId).emit("message", { userId, text, ts: Date.now() });
  });
});

httpServer.listen(3000);
```

---

## WebSocket vs Socket.IO

| Feature | Native WebSocket (`ws`) | Socket.IO |
|---------|------------------------|-----------| 
| **Protocol** | RFC 6455 WebSocket | Custom protocol over Engine.IO |
| **Handshake** | HTTP Upgrade | Engine.IO handshake + optional upgrade |
| **Reconnection** | Manual implementation | Automatic with configurable backoff |
| **Rooms** | Manual implementation | Built-in (`socket.join()`, `io.to()`) |
| **Broadcasting** | Manual iteration | Built-in scoped emit |
| **Acknowledgments** | Manual implementation | Built-in with timeout |
| **Binary** | Native | Auto-detected and converted |
| **Overhead** | 2-6 bytes/message | ~20-40 bytes/message |
| **Fallback** | None | HTTP long-polling, then upgrade |
| **Horizontal scaling** | Manual pub/sub | Redis adapter built-in |
| **Performance** | Maximum throughput | Good throughput, more features |

**Choose `ws`** when: you need maximum performance, have a small team comfortable managing infrastructure, and don't need rooms/acks/broadcasting.

**Choose Socket.IO** when: you need built-in reconnection, rooms/namespaces, acknowledgments, and horizontal scaling with minimal infrastructure code.

For high-throughput scenarios, consider `uWebSockets.js`:

```javascript
import uWS from "uWebSockets.js";

const app = uWS.App()
  .ws("/*", {
    compression: uWS.SHARED_COMPRESSOR,
    maxPayloadLength: 64 * 1024,
    idleTimeout: 60,
    open: (ws) => console.log("Client connected"),
    message: (ws, message, isBinary) => ws.send(message, isBinary),
    close: (ws, code, message) => console.log(`Disconnected: ${code}`),
  })
  .listen(9001, (token) => {
    if (token) console.log("Listening on port 9001");
  });
```

---

## WebTransport Architecture

### Why WebTransport

WebSocket runs over TCP, which enforces ordered delivery for all data on a single connection. If one message is lost, all subsequent messages stall until the retransmission is complete — this is head-of-line (HOL) blocking. WebTransport solves this by running over HTTP/3 (QUIC), which provides:

1. **Multiple independent streams** — Each stream has its own ordering. A stalled stream does not block others.
2. **Unreliable datagrams** — Fire-and-forget delivery for loss-tolerant data (video frames, cursor positions). Dropping a lost frame is better than stalling playback.
3. **0-RTT connection resumption** — QUIC's session ticket mechanism allows instant reconnection without a full handshake.
4. **Transparent connection migration** — QUIC connections survive IP/network changes (wifi → cellular) because connections are identified by Connection IDs, not IP:port tuples.

### Server Implementation

```javascript
import { WebTransportServer } from "@aspect/webtransport";
import { readFileSync } from "fs";

const server = new WebTransportServer({
  port: 4433,
  host: "0.0.0.0",
  cert: readFileSync("cert.pem"),
  key: readFileSync("key.pem"),
});

server.ready.then(() => {
  console.log("WebTransport server ready on port 4433");
});

server.on("session", async (session) => {
  // Reliable bidirectional stream
  const stream = await session.createBidirectionalStream();
  const reader = stream.readable.getReader();
  const writer = stream.writable.getWriter();

  // Handle incoming data
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    await processMessage(value);
  }

  // Unreliable datagrams for loss-tolerant data
  const dgReader = session.datagrams.readable.getReader();
  while (true) {
    const { value, done } = await dgReader.read();
    if (done) break;
    processDatagram(value); // OK to lose some
  }
});
```

### Client Implementation

```javascript
const transport = new WebTransport("https://edge.example.com:4433/stream");
await transport.ready;

// Reliable stream for critical data
const stream = await transport.createBidirectionalStream();
const writer = stream.writable.getWriter();
await writer.write(encoder.encode(JSON.stringify(event)));

// Unreliable datagrams for loss-tolerant data
const datagramWriter = transport.datagrams.writable.getWriter();
await datagramWriter.write(new Uint8Array([0x01, 0x02, 0x03]));

// Handle connection closure
transport.closed.then(() => {
  console.log("Transport closed cleanly");
}).catch((err) => {
  console.error("Transport closed with error:", err);
});
```

### Sequence Coordination

For protocols handling interleaved parallel streams (WebTransport), out-of-order messages are common. Implement producer-side sequencing:

| Header | Purpose |
|--------|---------|
| `Producer-Id` | Stable client identifier (survives reconnection) |
| `Producer-Epoch` | Integer incremented on client restart (invalidates old delayed packets) |
| `Producer-Seq` | Monotonically increasing sequence number per epoch |

---

## Migration Paths

### SSE → WebSocket

When unidirectional push evolves to require client-initiated writes:

1. Keep SSE for the read stream (`GET /events`)
2. Add WebSocket for writes (`wss://api.example.com/ws`)
3. Both share the same backplane and routing directory
4. Eventually consolidate to WebSocket if bidirectional volume justifies the infrastructure complexity

### WebSocket → WebTransport

When HOL blocking or connection migration become requirements:

1. Implement feature detection on the client
2. Prefer WebTransport where available, fall back to WebSocket
3. Abstract the transport layer behind a common interface
4. The server must support both simultaneous protocols during migration

```javascript
async function connectTransport(url) {
  if ("WebTransport" in window) {
    try {
      const wt = new WebTransport(url.replace("wss://", "https://") + "/webtransport");
      await wt.ready;
      return new WebTransportAdapter(wt);
    } catch {
      // Fall through to WebSocket
    }
  }
  return new WebSocketAdapter(new WebSocket(url));
}
```

---

## Performance Benchmarks

### Latency (p99)

| Protocol | Startup | Per-Message |
|----------|---------|-------------|
| SSE (HTTP/3) | 0-1 RTT | 10-50ms |
| WebSocket | 3-4 RTTs | 5-20ms |
| WebTransport | 0-1 RTT | 3-15ms |
| Long Polling | 1 RTT/poll | 100-500ms |

### Max Connections Per Server (Node.js, 4GB RAM)

| Protocol | Connections |
|----------|------------|
| SSE | 100,000+ |
| WebSocket (ws) | 50,000-100,000 |
| WebSocket (Socket.IO) | 20,000-50,000 |
| WebTransport | 50,000+ |
| Long Polling | 10,000-20,000 |

### Per-Message Overhead

| Protocol | Bytes |
|----------|-------|
| WebSocket | 2-6 |
| SSE | ~20 |
| Long Polling | 500-2000 (HTTP headers) |
| WebTransport | 2-6 (stream), 0 (datagram) |

---

## Industry Patterns

### High-Frequency Trading (HFT)

For standard market data feeds and order execution:

- **Protocol:** WebTransport is the clear choice. 0-RTT connection resumption provides a critical latency edge — reconnecting after a network blip without a full QUIC handshake means missed ticks are measured in single-digit milliseconds rather than seconds.
- **Stream isolation:** Independent streams for different symbol feeds prevent a stalled order acknowledgment from blocking price updates.
- **Serialization:** Protobuf or FlatBuffers mandatory — JSON overhead is unacceptable at tick frequencies above 10,000/sec.
- **Hardware edge:** Leading firms bypass the OS entirely with FPGA-based network processing for nanosecond-level execution. For software-based systems, WebTransport over QUIC is the nearest equivalent at the application layer.

### Cloud Gaming

WebTransport's dual-mode capability maps directly to gaming requirements:

- **Button inputs → Reliable streams:** Player actions (jump, shoot, menu selection) must arrive exactly once and in order. Use bidirectional reliable streams.
- **Video frames → Unreliable datagrams:** Dropping a lost video frame is better than stalling the game while waiting for TCP retransmission. Use the datagram API for all media frames.
- **Cursor/pointer position → Unreliable datagrams:** High-frequency, low-value positional data. Loss-tolerant.

```javascript
// Cloud gaming transport setup
const transport = new WebTransport("https://edge.gaming.com/session");
await transport.ready;

// Reliable stream for player inputs (must arrive, must be ordered)
const inputStream = await transport.createBidirectionalStream();
const inputWriter = inputStream.writable.getWriter();

await inputWriter.write(encoder.encode(JSON.stringify({
  type: "input",
  action: "jump",
  frame: 14523,
  timestamp: performance.now(),
})));

// Unreliable datagrams for video frame acknowledgments (loss-tolerant)
const dgWriter = transport.datagrams.writable.getWriter();
await dgWriter.write(new Uint8Array([0x01, /* frame ack */ 0x38, 0xCB]));
```

### Node.js Clustering

For maximum throughput on multi-core machines:

```javascript
import cluster from "cluster";
import os from "os";

if (cluster.isPrimary) {
  const numWorkers = os.cpus().length;
  console.log(`Primary ${process.pid} starting ${numWorkers} workers`);

  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} died, respawning`);
    cluster.fork();
  });
} else {
  // Each worker runs the server
  import("./socket-server.js");
}
```
