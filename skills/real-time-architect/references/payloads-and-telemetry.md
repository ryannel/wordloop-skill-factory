# Payload Standardization & Continuous Telemetry

Traditional HTTP APIs trace requests using HTTP Headers (`traceparent`, `tracestate`, `Authorization`). However, once a WebSocket or SSE connection is successfully upgraded or established, subsequent real-time messages are completely stripped of HTTP metadata.

## The Message Envelope Pattern

If you serialize arbitrary JSON across persistent sockets, you lose the ability to track the message path across distributed microservices.

**Actionable Rule:**
Enforce the **CloudEvents** specification to construct a "Message Envelope" that acts as a metadata carrier for every payload pushed across the stream.

### Structural Requirements
The payload must decouple routing variables from domain variables:
*   `eventId`: UUID for idempotency.
*   `eventType`: Inverse domain notation (e.g. `com.billing.invoice.paid`).
*   `version`: Guaranteeing schema compatibility over time.

### Trace Context Propagation
To preserve OpenTelemetry distributed tracing:
1. Extract the `traceparent` and `tracestate` variables natively supported by the CloudEvents Distributed Tracing Extension.
2. Inject these variables into the JSON or Binary envelope before dispatch.
3. Upon receiving the frame, the subscriber opens the Envelope, extracts the `traceparent`, and initiates a child span in the APM—ensuring visibility remains unbroken across the asynchronous boundaries.

## Message-Level Authorization
Do not assume that because the initial HTTP handshake was authorized, the connection remains permanently trusted. Permissions expire.

Architect the payload processing layer to perform Continuous Message-Level Authorization. Extract workload identities or BOLA controls encoded deeply within the envelope on every received event rather than trusting the open socket blind.
