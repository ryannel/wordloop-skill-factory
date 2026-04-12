# Asynchronous & Event-Driven Patterns

High-throughput systems decouple execution through queues, brokers, and streams. We enforce strict machine-readable contracts here exactly as we do for REST.

## AsyncAPI 3.0 & Serialization

Do not simply drop messages into queues globally. Utilize **AsyncAPI 3.0** definitions to document channels, topics, payload schemas, and servers. This allows automated generation of consumer stubs exactly like OpenAPI does for clients.

**Serialization Matrix:**
For high-scale internal communications (e.g. gRPC or Kafka streams), prefer binary serializers like **Protocol Buffers (Protobuf)** or **Apache Avro**. They drastically reduce computational parsing load and network payload bulk compared to JSON. Save JSON strictly for public or browser-facing API communication boundaries.

## The Async Request-Reply (202 Accepted)
When a client triggers a synchronous HTTP call that requires 3 minutes of background ML computation, blocking the HTTP connection will result in a 504 Timeout.

1. **Request:** Client calls `POST /heavy-jobs`.
2. **Accept:** API enqueues the job onto Kafka/RabbitMQ and immediately answers HTTP `202 Accepted`.
3. **Monitor:** Response headers include `Location: /jobs/{id}/status`.
4. **Conclusion:** Client gracefully polls the Location URL (or waits for a webhook/SSE) until processing concludes.

## Dual-Writes & The Transactional Outbox

Updating a database and publishing an event to Kafka in overlapping steps creates a "dual write" risk. If the DB commits, but the network to Kafka drops, the system state is fatally misaligned.

**The Transactional Outbox Solutions:**
1. During the primary database transaction, write your core payload to its table, AND immediately write an event payload to an `outbox_events` table utilizing the same internal DB transaction log mechanism. 
2. A separate daemon (like Debezium for Change Data Capture) parses the `outbox_events` table transaction trail and pushes it asynchronously to Kafka. If it crashes, it just restarts parsing from its last marker. It guarantees at-least-once delivery.

Because "at-least-once" delivery can duplicate payloads, always ensure your Event Consumers evaluate an `Idempotency-Key` via a centralized datastore (like Redis) before running side effects.

## Navigating Kafka Partitions

Topic partitions are the primary dimension of Kafka scale, acting as parallel lanes.

* **Key Hashing:** Always include well-distributed entity IDs (`userId`, `orderId`) as partition keys to guarantee temporal ordering per entity instead of random round-robin.
* **Hot Partitions:** Do not use boolean statuses or low-cardinality keys, as they will funnel traffic heavily into singular partitions, crushing isolated broker nodes.
