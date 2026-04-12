# Serialization & Wire Formats

Reference for choosing and implementing wire-level serialization strategies. The format you use between producer and consumer directly impacts bandwidth, latency, and CPU cost.

## Table of Contents

- [Format Selection](#format-selection)
- [Protobuf](#protobuf)
- [FlatBuffers](#flatbuffers)
- [Compression](#compression)
- [Binary Data Handling](#binary-data-handling)
- [AI Pipeline Economics](#ai-pipeline-economics)

---

## Format Selection

| Format | Size vs. JSON | Deserialization Cost | Schema Enforcement | Best For |
|--------|--------------|---------------------|-------------------|----------|
| **JSON** | Baseline | Medium | Runtime (manual) | Human-readable APIs, debugging, GraphQL |
| **Protobuf** | ~33% smaller | Low | Strict (.proto) | Service-to-service, CloudEvents, gRPC |
| **FlatBuffers** | ~40% smaller | Near-zero (zero-copy) | Strict (.fbs) | Read-heavy feeds, financial data, gaming |
| **MessagePack** | ~20% smaller | Low | Runtime (manual) | Drop-in JSON replacement, Redis caching |

**Default choice:** Protobuf for all binary payloads. Use JSON at boundaries where human readability matters (GraphQL, debugging, LLM injection points).

---

## Protobuf

### CloudEvents Envelope in Protobuf

Protobuf provides ~33% payload reduction compared to JSON while maintaining strict schema enforcement:

```protobuf
syntax = "proto3";

package events.v1;

message CloudEventEnvelope {
  string specversion = 1;
  string id = 2;
  string type = 3;
  string source = 4;
  string time = 5;
  string datacontenttype = 6;
  TracingContext tracing = 7;
  IdempotencyMeta idempotency = 8;
  bytes data = 9;  // Serialized domain payload
}

message TracingContext {
  string traceparent = 1;
  string tracestate = 2;
}

message IdempotencyMeta {
  string key = 1;
  string expires_at = 2;
}
```

### Schema Evolution Rules

Protobuf enforces strict rules for backward/forward compatibility:

1. **Safe changes** (additive): Add new optional fields with new field numbers
2. **Unsafe changes** (breaking): Renaming fields, changing field numbers, changing types
3. **Deprecation**: Mark fields as `reserved` to prevent reuse of old field numbers

```protobuf
message OrderEvent {
  string order_id = 1;
  double amount = 2;
  string currency = 3;

  // Added in v2 — backward compatible (old consumers ignore it)
  optional string customer_note = 4;

  // Prevent future reuse of removed field numbers
  reserved 5, 6;
  reserved "deprecated_field_name";
}
```

---

## FlatBuffers

Use FlatBuffers for read-heavy workloads where deserialization cost matters (financial data feeds, high-frequency telemetry). FlatBuffers allow accessing fields without unpacking the entire buffer — zero-copy reads.

```flatbuffers
// price_update.fbs
namespace Trading;

table PriceUpdate {
  symbol: string;
  bid: double;
  ask: double;
  volume: long;
  timestamp: long;
  exchange: string;
}

root_type PriceUpdate;
```

**When to use FlatBuffers over Protobuf:**
- Read frequency vastly exceeds write frequency (e.g., streaming price updates read by thousands of consumers)
- Deserialization appears in profiling hot paths
- Message sizes exceed 1KB and you only need to access a few fields

---

## Compression

### WebSocket `permessage-deflate`

Extension-level compression for WebSocket messages. Configured at the server:

```javascript
const wss = new WebSocket.Server({
  port: 8080,
  perMessageDeflate: {
    zlibDeflateOptions: {
      chunkSize: 1024,
      memLevel: 7,
      level: 3,          // 1 = fastest, 9 = best compression
    },
    zlibInflateOptions: {
      chunkSize: 10 * 1024,
    },
    clientNoContextTakeover: true,   // Free memory per message (lower RAM, higher CPU)
    serverNoContextTakeover: true,
    serverMaxWindowBits: 10,
    concurrencyLimit: 10,            // Limit concurrent zlib operations
    threshold: 1024,                 // Only compress messages > 1KB
  },
});
```

**Trade-offs:**
- `NoContextTakeover: true` — Reduces memory per connection but increases CPU because each message is compressed independently (no shared dictionary). Use for high connection counts with moderate message sizes.
- `NoContextTakeover: false` — Maintains a compression dictionary per connection for better ratios but increases memory. Use for fewer connections with large, repetitive payloads.

For Socket.IO, compression is configured at the engine level:

```javascript
const io = new Server(httpServer, {
  perMessageDeflate: {
    threshold: 2048,  // Only compress messages > 2KB
  },
});
```

### Compression Decision

| Scenario | Compress? | Notes |
|----------|-----------|-------|
| JSON messages > 1KB | Yes | Significant savings on repetitive JSON |
| Protobuf messages | Rarely | Already compact; compression adds CPU with minimal savings |
| Binary (images, video) | No | Already compressed; double-compression wastes CPU |
| High-frequency small messages | No | Overhead exceeds savings below ~200 bytes |

---

## Binary Data Handling

WebSockets natively support binary frames alongside text frames. Use binary for Protobuf, images, or any non-text payload.

### Server-Side (Node.js)

```javascript
// Send binary data
const buffer = Buffer.from([0x00, 0x01, 0x02, 0x03]);
ws.send(buffer, { binary: true });

// Detect data type on receive
ws.on("message", (data, isBinary) => {
  if (isBinary) {
    // Protobuf, images, etc.
    const decoded = MyProtobufMessage.decode(data);
    processProtobuf(decoded);
  } else {
    // JSON text
    const event = JSON.parse(data.toString());
    processEvent(event);
  }
});
```

### Browser Client

```javascript
const ws = new WebSocket("wss://api.example.com");
ws.binaryType = "arraybuffer"; // or "blob"

ws.onmessage = (event) => {
  if (event.data instanceof ArrayBuffer) {
    const view = new Uint8Array(event.data);
    processProtobuf(view);
  } else {
    const payload = JSON.parse(event.data);
    processEvent(payload);
  }
};
```

### WebTransport Binary Streams

WebTransport uses the Streams API for binary data — no text/binary frame distinction:

```javascript
const transport = new WebTransport("https://edge.example.com/stream");
await transport.ready;

const stream = await transport.createBidirectionalStream();
const writer = stream.writable.getWriter();

// All data is binary (Uint8Array)
const encoded = new TextEncoder().encode(JSON.stringify(event));
await writer.write(encoded);

// Or send raw protobuf bytes
const protobufBytes = MyMessage.encode(payload).finish();
await writer.write(protobufBytes);
```

---

## AI Pipeline Economics

When streaming data feeds to LLM inference endpoints, the wire format strategy significantly impacts API costs:

```
┌─────────────┐  Protobuf   ┌──────────────┐  Protobuf   ┌──────────────┐
│   Producer   │ ──────────> │ Intermediate │ ──────────> │   Decode     │
│   Service    │   (binary)  │  Services    │   (binary)  │   Boundary   │
└─────────────┘             └──────────────┘             └──────┬───────┘
                                                                │
                                                    Minified JSON (text)
                                                                │
                                                         ┌──────▼───────┐
                                                         │  LLM / AI    │
                                                         │  Endpoint    │
                                                         └──────────────┘
```

### Strategy

1. **Wire format:** Binary (Protobuf) for minimal bandwidth across all internal services
2. **Decode boundary:** Convert to minified JSON only at the point of LLM prompt injection
3. **Impact:** Significant reduction in AI API token costs by eliminating verbose JSON transmission across internal services
4. **Field selection:** At the decode boundary, serialize only the fields the LLM needs — strip metadata, tracing, and infrastructure fields before injection
