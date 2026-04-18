# Data Architecture Decision Framework

Choosing the right data architecture is a design decision, not a doctrine. The correct pattern depends on the system's latency requirements, data volume, reprocessing needs, and operational budget. This reference provides selection criteria for five common patterns, ordered from simplest to most complex.

## Decision Flowchart

```
Does the feature need real-time push to clients?
├── No  → 1. Direct Request/Response
└── Yes
    ├── Is it primarily entity CRUD with multi-device / multi-tab sync?
    │   └── Yes → 2. Optimistic Mutation + Echo-Suppressed Streaming ★
    ├── Do multiple services need to react to the same event independently?
    │   └── Yes → 3. Event-Driven Pub/Sub
    ├── Is the event log the source of truth with full replay required?
    │   └── Yes → 4. Kappa Architecture
    └── Do you need both historical batch reprocessing AND real-time views?
        └── Yes → 5. Lambda Architecture
```

> **★ Default for Wordloop.** Pattern 2 is the canonical design for the Wordloop platform and most entity-level CRUD applications requiring low-latency, multi-device synchronization. See `optimistic-echo-suppression.md` for the complete edge-case reference.

## Pattern Selection Matrix

| Pattern | Latency | Complexity | Reprocessing | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **1. Direct Request/Response** | Per-request | Low | N/A | CRUD apps, form submissions, simple APIs |
| **2. Optimistic Mutation + Echo-Suppressed Streaming** | Perceived instant | Medium | N/A | Entity CRUD with multi-device sync |
| **3. Event-Driven Pub/Sub** | Near real-time | Medium | Limited | Decoupling services, async workflows, notifications |
| **4. Kappa (Unified Stream)** | Real-time | High | Full replay | Live dashboards, audit trails, event-sourced domains |
| **5. Lambda (Batch + Speed)** | Mixed | Very High | Native batch | ML training pipelines, historical analytics + live views |

---

## 1. Direct Request/Response

The simplest pattern. A client sends a request, the server processes it synchronously, and returns a response. No intermediate message broker, no log, no replay.

**When to use:**
- The interaction is inherently transactional (create, read, update, delete).
- There is no need to fan data out to multiple consumers.
- Latency tolerance is per-request (under 500ms is fine).
- The system does not need an audit trail or event history.
- Only a single client instance ever displays the data.

**When it breaks down:**
- Multiple downstream services need to react to the same state change.
- You need to decouple the producer from consumers for independent scaling.
- Real-time push to clients (dashboards, notifications) is required.
- Multiple tabs/devices need to stay synchronized.

Most applications start here. Do not introduce messaging infrastructure until the use case demands it.

---

## 2. Optimistic Mutation with Echo-Suppressed Streaming

**The canonical pattern for entity-level CRUD applications requiring low-latency, multi-device synchronization.** This splits the write path (REST) from the read path (WebSocket/SSE) and uses per-tab client IDs for echo suppression.

**When to use:**
- The application is entity CRUD (notes, tasks, documents, records).
- Users operate across multiple tabs or devices simultaneously.
- Perceived latency must be instant — users should not wait for server confirmation.
- The server is the authoritative source of truth, and eventual consistency is acceptable.
- The data model is entity-level (not collaborative text editing requiring CRDTs).

**When it breaks down:**
- Multiple users edit the same text field simultaneously (need CRDTs — see `state.md`).
- The application requires guaranteed strong consistency (financial transactions).
- There is no multi-device or multi-user requirement (use pattern 1 instead).
- Event volume is extreme (>100k events/sec) and replay is needed (use pattern 4).

**Implementation summary:**

```
Step 1: Client generates X-Client-Id (UUIDv4, per browser tab)

Step 2: User mutates → client applies optimistic update immediately
        → sends REST POST/PUT/DELETE with X-Client-Id header

Step 3: Server persists to database (authoritative write)
        → publishes event to WebSocket hub with sourceClientId

Step 4: WebSocket hub fans out to all connected clients for that user

Step 5: Each client checks event.sourceClientId vs own clientId:
        → Match (echo): suppress (already shown optimistically)
        → No match (other device): apply to UI as new data
```

**Critical edge cases:** idempotency keys, version-based out-of-order discard, rollback on server rejection, reconnection with full state hydration, rapid mutation coalescing, auth lifecycle on token expiry.

For the complete edge-case reference with implementation details, see `optimistic-echo-suppression.md`.

---

## 3. Event-Driven Pub/Sub

Services communicate by publishing events to a message broker (e.g., GCP Pub/Sub, RabbitMQ, Amazon SNS/SQS). Consumers subscribe to topics and react asynchronously. Events are typically transient — consumed and acknowledged, not replayed.

**When to use:**
- Services need to be decoupled — the publisher should not know or care about consumers.
- Multiple consumers need to react to the same event independently.
- Async processing is acceptable (email notifications, webhook dispatch, background jobs).
- You need at-least-once delivery guarantees but not full event replay.

**When it breaks down:**
- You need to replay historical events to rebuild state or recompute derived data.
- Ordering guarantees are critical across partitions.
- The event log itself is the source of truth (event sourcing).

**Implementation guidance:**
1. Use a managed broker (GCP Pub/Sub, Amazon SQS) to avoid operational overhead.
2. Make consumers idempotent — duplicates will happen.
3. Use dead-letter queues for poison messages.
4. Monitor consumer lag as a first-class SLI.

---

## 4. Kappa Architecture (Unified Streaming)

All data flows through a single, durable, append-only log (e.g., Apache Kafka, Redpanda). A unified stream processing engine computes real-time materialized views. If business logic changes, replay the immutable log through updated processors to derive new state.

**When to use:**
- The system needs real-time materialized views that update continuously.
- Event sourcing is a domain requirement — the log of events *is* the source of truth.
- Replayability is essential (auditing, compliance, schema evolution).
- The same data must serve both real-time consumers and historical queries.

**When it is overkill:**
- The application is primarily CRUD with no real-time push requirements.
- Event volume is low and the operational cost of Kafka/Redpanda is not justified.
- The team lacks experience operating distributed log infrastructure.
- Reprocessing is rare (happens quarterly, not on every deployment).

**Implementation guidance:**
1. **The immutable backbone.** All raw incoming data is written to a durable, append-only log (Kafka, Redpanda). This log is the single source of truth.
2. **Unified stream processing.** A single processing engine (Kafka Streams, Apache Flink) computes materialized views and triggers downstream events.
3. **Replayability.** When business logic changes, replay the historical log through updated processors to generate new state — eliminating the need for a separate batch layer.
4. **Compaction strategy.** Configure log compaction for entity-keyed topics to bound storage while preserving the latest state per key.
5. **Schema evolution.** Use a schema registry (Confluent, Redpanda) with backward-compatible Avro or Protobuf schemas to evolve payloads safely.

**Trade-offs to communicate:**
- Log retention and compaction require careful capacity planning.
- Replay latency grows with log size — cold replays of large topics can take hours.
- Schema evolution mistakes are expensive because the log is immutable.

---

## 5. Lambda Architecture (Batch + Speed)

Maintains two parallel processing paths: a **batch layer** for comprehensive, high-accuracy historical processing, and a **speed layer** for low-latency approximate results. A serving layer merges both views.

**When to use:**
- Historical reprocessing on massive datasets is a core requirement (ML model training, large-scale analytics).
- The batch layer needs different compute characteristics than real-time (e.g., Spark/BigQuery for batch, Flink for speed).
- The cost of replaying an entire event log through a single streaming engine is prohibitive.
- Regulatory requirements mandate periodic full-dataset reconciliation separate from real-time views.

**When to avoid:**
- The dual codebase burden is not justified — maintaining batch and speed logic in sync is the primary operational cost.
- A Kappa replay can meet the same reprocessing needs with a single engine.
- The team is small and cannot afford the operational complexity of two pipelines.

**Implementation guidance:**
1. **Batch layer.** Periodically process the full dataset (BigQuery, Spark) to produce highly accurate master views.
2. **Speed layer.** Stream recent events through a real-time engine (Flink, Dataflow) to produce low-latency approximate views.
3. **Serving layer.** Merge batch and speed views at query time. The speed layer compensates for batch layer latency.
4. **Reconciliation.** The batch layer periodically overwrites the speed layer's approximations with accurate results.

---

## Targeted Fan-Out Topology

Regardless of the data architecture pattern chosen, scaling from thousands to millions of concurrent socket connections changes how a system routes events to connected clients.

### The Anti-Pattern: Global Pub/Sub Broadcast
Legacy architectures often have every stateful API node subscribe to a global `messages` Redis channel. If User A triggers an event meant for User B, all 100 API nodes receive the message, even though 99 of them immediately discard it. This creates severe $O(N^2)$ broadcast storms.

### The Standard: Targeted Unicast via Relay Services
1. **The Stateless Relay.** API instances serving SSE/WebTransport should be decoupled "Relay Services". They hold the active TCP/QUIC sessions but process zero domain logic.
2. **The Routing Directory.** Upon client connection, the Relay Service writes to a high-speed KV store (e.g., Redis or DynamoDB): `UserId_123 -> Pod_IP_10.0.4.5`.
3. **Direct Routing.** When the backend yields an event for `UserId_123`, the internal event bus looks up the Routing Directory and pushes the message *exclusively* to `Pod_IP_10.0.4.5`.
4. **Delivery.** The Relay Service pushes the payload down the open connection to the client.

By decoupling persistent connection handling from heavy stream-processing logic, the connection tier can be scaled automatically on memory or connection-count metrics.

---

## Cross-References

- **Optimistic mutation implementation details** → `optimistic-echo-suppression.md`
- **CRDTs for collaborative editing** → `state.md`
- **Protocol selection (SSE, WebSocket, WebTransport)** → `protocols.md`
- **Client-side patterns (offline queues, reconnection)** → `client.md`
- **Contract-first design & CloudEvents** → `contracts.md`
