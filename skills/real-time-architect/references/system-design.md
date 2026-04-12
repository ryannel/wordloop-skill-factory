# System Design & Data Pipelines

When architecting real-time streaming backends, you must look beyond the edge nodes handling the client connections. The backend data pipeline dictates how effectively real-time state can be derived, routed, and scaled.

## The Kappa Architecture Paradigm

Avoid the traditional **Lambda Architecture**, which requires maintaining two separate codebases (a slow, highly accurate Batch layer and a fast, approximated Speed layer) that must constantly be synchronized.

Instead, advocate for the **Kappa Architecture** ("Everything is a Stream"):
1. **The Immutable Backbone:** All raw incoming data is written to a highly durable, append-only log (e.g., Apache Kafka, Redpanda). 
2. **Unified Stream Processing:** A single unified stream processing engine (e.g., Kafka Streams, Apache Flink) computes real-time materialized views or triggers subsequent events.
3. **Replayability:** If business logic changes, you replay the historical immutable log through the updated stream processor to generate the new state—eliminating the need for a separate batch layer.

## Targeted Fan-Out Topography

Scaling from thousands to millions of concurrent socket connections changes how a system routes events.

### The Anti-Pattern: Global Pub/Sub Broadcast
Legacy architectures often have every stateful API node subscribe to a global `messages` Redis channel. If User A triggers an event meant for User B, all 100 API nodes receive the message, even though 99 of them immediately discard it because User B isn't connected to them. This creates severe $O(N^2)$ broadcast storms.

### The Standard: Targeted Unicast via Relay Services
1. **The Stateless Relay:** The API instances serving SSE/WebTransport should be entirely decoupled "Relay Services". They hold the active TCP/QUIC sessions but process zero domain logic.
2. **The Routing Directory:** Upon client connection, the Relay Service writes to a high-speed KV store (e.g., Redis or DynamoDB): `UserId_123 -> Pod_IP_10.0.4.5`.
3. **Direct Routing:** When the backend stream processor yields an event for `UserId_123`, the internal event bus looks up the Routing Directory and pushes the message *exclusively* to `Pod_IP_10.0.4.5`.
4. **Delivery:** The Relay Service pushes the payload down the open connection to the client.

By decoupling the persistent connection handling from the heavy stream-processing logic, the connection tier can be scaled automatically strictly on memory or connection-count metrics.
