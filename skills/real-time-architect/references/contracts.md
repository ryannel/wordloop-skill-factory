# Contracts & Governance

Deep reference for contract-first design with AsyncAPI 3.0 and standardized CloudEvents payload envelopes. Every real-time API must have a machine-readable contract that serves as the single source of truth.

For serialization formats (Protobuf, FlatBuffers, compression), see `serialization.md`. For runtime validation and SDK generation, see `client.md`. For contract testing and CI/CD validation, see `testing.md`.

## Table of Contents

- [AsyncAPI 3.0 Specification](#asyncapi-30-specification)
- [CloudEvents Envelope](#cloudevents-envelope)
- [Schema Evolution](#schema-evolution)
- [Schema Registry](#schema-registry)

---

## AsyncAPI 3.0 Specification

### Structure

AsyncAPI 3.0 separates concerns into independent building blocks:

- **Channels** — Technical addresses where messages flow (e.g., `/v1/orders/updates`)
- **Operations** — Actions bound to channels (`send`, `receive`)
- **Messages** — Structured payloads with schemas, headers, and metadata
- **Message Traits** — Reusable message semantics (correlation IDs, auth headers, CloudEvents envelope)
- **Server Bindings** — Protocol-specific configurations (WebSocket, MQTT, Kafka, WebTransport)

### Complete Example: Collaborative Editing API

```yaml
asyncapi: 3.0.0
info:
  title: Collaborative Editing API
  version: 2.1.0
  description: Real-time document collaboration with CRDT-based conflict resolution

servers:
  production:
    host: ws.example.com
    protocol: wss
    description: Production WebSocket endpoint
    security:
      - $ref: '#/components/securitySchemes/bearerAuth'
    bindings:
      ws:
        headers:
          type: object
          properties:
            Authorization:
              type: string
              description: Bearer JWT token

  edge-transport:
    host: edge.example.com
    protocol: https
    description: WebTransport endpoint for low-latency editing
    bindings:
      http:
        version: "1.1"

channels:
  document-updates:
    address: /v1/documents/{documentId}/updates
    parameters:
      documentId:
        description: Unique document identifier
        schema:
          type: string
          format: uuid
    messages:
      documentChanged:
        $ref: '#/components/messages/DocumentChanged'
      cursorMoved:
        $ref: '#/components/messages/CursorMoved'

  presence:
    address: /v1/documents/{documentId}/presence
    parameters:
      documentId:
        schema:
          type: string
          format: uuid
    messages:
      userPresence:
        $ref: '#/components/messages/UserPresence'

operations:
  onDocumentChanged:
    action: receive
    channel:
      $ref: '#/channels/document-updates'
    messages:
      - $ref: '#/channels/document-updates/messages/documentChanged'
    traits:
      - $ref: '#/components/operationTraits/cloudEventsEnvelope'

  sendEdit:
    action: send
    channel:
      $ref: '#/channels/document-updates'
    messages:
      - $ref: '#/channels/document-updates/messages/documentChanged'
    reply:
      channel:
        $ref: '#/channels/document-updates'
      address:
        location: '$message.header#/replyTo'

  broadcastPresence:
    action: receive
    channel:
      $ref: '#/channels/presence'
    messages:
      - $ref: '#/channels/presence/messages/userPresence'

components:
  messages:
    DocumentChanged:
      name: documentChanged
      title: Document Changed Event
      contentType: application/json
      traits:
        - $ref: '#/components/messageTraits/cloudEventsMeta'
        - $ref: '#/components/messageTraits/correlationId'
      payload:
        type: object
        required: [documentId, operations, version]
        properties:
          documentId:
            type: string
            format: uuid
          operations:
            type: array
            items:
              $ref: '#/components/schemas/CRDTOperation'
          version:
            type: integer
            description: Document version after applying operations

    CursorMoved:
      name: cursorMoved
      title: Cursor Position Update
      contentType: application/json
      payload:
        type: object
        properties:
          userId:
            type: string
          position:
            type: object
            properties:
              line: { type: integer }
              column: { type: integer }

    UserPresence:
      name: userPresence
      title: User Presence Event
      contentType: application/json
      payload:
        type: object
        properties:
          userId: { type: string }
          status:
            type: string
            enum: [online, idle, offline]
          lastSeen:
            type: string
            format: date-time

  messageTraits:
    cloudEventsMeta:
      headers:
        type: object
        required: [ce-specversion, ce-id, ce-type, ce-source, ce-time]
        properties:
          ce-specversion: { type: string, const: "1.0" }
          ce-id: { type: string, format: uuid }
          ce-type: { type: string }
          ce-source: { type: string, format: uri }
          ce-time: { type: string, format: date-time }

    correlationId:
      correlationId:
        location: '$message.header#/ce-id'

  operationTraits:
    cloudEventsEnvelope:
      description: All operations use CloudEvents structured content mode

  schemas:
    CRDTOperation:
      type: object
      required: [type, path, value, timestamp, authorId]
      properties:
        type:
          type: string
          enum: [insert, delete, update]
        path:
          type: string
          description: JSON Pointer to the modified field
        value: {}
        timestamp:
          type: string
          format: date-time
        authorId:
          type: string

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

### Dynamic Request-Reply

AsyncAPI 3.0 supports runtime expressions for dynamic response channels:

```yaml
operations:
  queryDocument:
    action: send
    channel:
      $ref: '#/channels/document-queries'
    reply:
      channel:
        $ref: '#/channels/query-responses'
      address:
        location: '$message.header#/replyTo'
```

The client sets a `replyTo` header with a unique address, and the server publishes the response to that address. This enables request-reply semantics over inherently pub/sub protocols.

### WebTransport Operation Bindings

When defining operations over WebTransport, use operation-level bindings to specify stream type and reliability mode:

```yaml
operations:
  sendInput:
    action: send
    channel:
      $ref: '#/channels/input-stream'
    bindings:
      webtransport:
        streamType: bidirectional
        reliability: reliable        # Player inputs must arrive in order

  sendFrameAck:
    action: send
    channel:
      $ref: '#/channels/frame-acks'
    bindings:
      webtransport:
        streamType: unidirectional
        reliability: unreliable      # Video frame acks are loss-tolerant
```

Key binding fields:
- `streamType`: `bidirectional` | `unidirectional` — Maps to WebTransport's stream creation API
- `reliability`: `reliable` (stream-backed) | `unreliable` (datagram-backed) — Determines whether the data uses QUIC reliable streams or the datagram API

---

## CloudEvents Envelope

Every message, regardless of protocol, uses the CloudEvents specification for metadata. This decouples infrastructure routing from domain logic — the gateway can read the envelope without deserializing the domain payload.

### Structured Content Mode (JSON)

```json
{
  "metadata": {
    "specversion": "1.0",
    "id": "evt_abc123_xyz456",
    "type": "com.exchange.order.filled",
    "source": "/trading/engine/v1",
    "time": "2025-03-27T10:15:30.005Z",
    "datacontenttype": "application/json",
    "subject": "order_99887766",
    "version": "2.1.0",
    "tracing": {
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
      "tracestate": "roster=active,p=1"
    },
    "idempotency": {
      "key": "idem_99887766_fill_1",
      "expiresAt": "2025-03-28T10:15:30.000Z"
    }
  },
  "data": {
    "orderId": "order_99887766",
    "symbol": "BTC-USD",
    "side": "buy",
    "filledQuantity": "0.05",
    "price": "64500.50",
    "fee": "0.0001",
    "status": "fully_filled"
  }
}
```

### SSE Content Mode

```text
event: order_filled
id: evt_abc123_xyz456
retry: 5000
data: {
data: "metadata": {
data: "specversion": "1.0",
data: "id": "evt_abc123_xyz456",
data: "type": "com.exchange.order.filled",
data: "source": "/trading/engine/v1",
data: "time": "2025-03-27T10:15:30.005Z"
data: },
data: "data": {
data: "orderId": "order_99887766",
data: "status": "fully_filled"
data: }
data: }
```

### Envelope Field Reference

| Field | Required | Purpose |
|-------|----------|---------|
| `specversion` | Yes | CloudEvents version (`1.0`) |
| `id` | Yes | UUID for deduplication |
| `type` | Yes | Reverse-DNS event type for semantic routing |
| `source` | Yes | Origin URI producing the event |
| `time` | Yes | ISO 8601 timestamp of event creation |
| `datacontenttype` | Recommended | MIME type of the data field |
| `subject` | Optional | Additional context (resource ID affected) |
| `version` | Extension | Schema version for safe evolution |
| `traceparent` | Extension | W3C Trace Context for distributed tracing |
| `tracestate` | Extension | Additional tracing vendor state |
| `idempotencyKey` | Extension | Prevents duplicate side effects (see `resiliency.md`) |

---

## Schema Evolution

Event schemas evolve over time. Design for backward compatibility from day one.

### Compatibility Modes

| Mode | Rule | When to Use |
|------|------|-------------|
| **Backward** | New code can read old events | Mandatory for event sourcing / replay |
| **Forward** | Old code can read/ignore new events | Crucial for rolling deployments |
| **Full** | Both backward and forward satisfied | Gold standard for production |

### Versioning Strategy

1. **Additive changes** (new optional fields) → No version bump needed. Old consumers ignore the new field.
2. **Breaking changes** (removed fields, type changes) → Increment the `version` field in the CloudEvents envelope. Never remove fields — mark them as deprecated.
3. **Multi-version support** → Consumers must handle the current version and one prior version simultaneously.
4. **Event upcasting** → When replaying old events from an event store, transform them to the latest schema on-the-fly:

```javascript
// Event upcasting: transform old event versions during replay
const upcasters = {
  "com.app.order.created": {
    "1.0": (event) => ({
      ...event,
      data: {
        ...event.data,
        // v2 added 'currency' field — default for old events
        currency: event.data.currency || "USD",
      },
      metadata: { ...event.metadata, version: "2.0" },
    }),
  },
};

function upcast(event) {
  const { type, version } = event.metadata;
  const upcaster = upcasters[type]?.[version];
  return upcaster ? upcast(upcaster(event)) : event; // Recursive until latest
}
```

---

## Schema Registry

For event-driven systems at scale, enforce schema governance at the broker level — not just at the application.

### Purpose

A schema registry prevents "poison messages" — events that fail deserialization because the producer shipped a schema change without coordinating with consumers. The registry rejects incompatible schemas before they reach any consumer.

### Broker-Level Enforcement

```bash
# Register a new schema version with compatibility check
curl -X POST \
  http://schema-registry:8081/subjects/document-updates-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schemaType": "PROTOBUF",
    "schema": "syntax = \"proto3\"; message DocumentUpdate { string document_id = 1; int32 version = 2; }"
  }'

# Check compatibility before registering
curl -X POST \
  http://schema-registry:8081/compatibility/subjects/document-updates-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schemaType": "PROTOBUF", "schema": "..."}'
```

### Application-Level Registry

For systems without a broker-level registry, implement validation at the application layer:

```javascript
const schemas = {
  "com.app.order.created": {
    "1.0": orderCreatedV1Schema,
    "2.0": orderCreatedV2Schema,
  },
};

function validateMessage(event) {
  const { type, version } = event.metadata;
  const schema = schemas[type]?.[version];

  if (!schema) {
    throw new Error(`Unknown schema: ${type}@${version}`);
  }

  return schema.validate(event.data);
}
```

### CI/CD Integration

Automate schema validation in your deployment pipeline. See `testing.md` for the full CI/CD workflow including breaking change detection and consumer-driven contract verification.
