# Webhooks & Event Delivery

## Table of Contents

- [When to Use Webhooks](#when-to-use-webhooks)
- [Event Design](#event-design)
- [Registration API](#registration-api)
- [Delivery Architecture](#delivery-architecture)
- [Security: HMAC Signatures](#security-hmac-signatures)
- [Idempotency and At-Least-Once Delivery](#idempotency-and-at-least-once-delivery)
- [Retry Policy](#retry-policy)
- [Consumer Guidelines](#consumer-guidelines)
- [Reconciliation API](#reconciliation-api)
- [Observability](#observability)
- [OpenAPI Integration](#openapi-integration)
- [Antipatterns](#antipatterns)

---

## When to Use Webhooks

Webhooks are server-to-server HTTP callbacks that push event notifications to consumer-registered URLs. Use them when:

- The consumer needs real-time awareness of state changes (order completed, payment failed, user deactivated)
- Polling would create excessive load on both sides
- Multiple independent systems need to react to the same event
- The API is a platform with third-party integrations

Webhooks complement the request-response API — they are not a replacement. The REST API remains the source of truth; webhooks notify consumers that something changed, and the consumer can optionally fetch the full resource representation via a GET request.

---

## Event Design

### Event Envelope

Every webhook delivery wraps the event data in a consistent envelope:

```json
{
  "id": "evt_abc123def456",
  "type": "order.completed",
  "created_at": "2024-06-15T14:30:00Z",
  "api_version": "2024-06-15",
  "data": {
    "id": "ord_xyz789",
    "status": "completed",
    "total": 149.99,
    "currency": "USD",
    "customer_id": "usr_456"
  }
}
```

### Envelope Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique, stable event ID. Must remain identical across all retries. |
| `type` | string | Yes | Dot-delimited event type: `{resource}.{action}` |
| `created_at` | string (ISO 8601) | Yes | When the event occurred |
| `api_version` | string | No | API version the payload conforms to (for date-based versioning) |
| `data` | object | Yes | The event payload — typically the resource representation at the time of the event |

### Event Type Naming

Use a `{resource}.{action}` pattern with dot-delimited namespacing:

```
user.created
user.updated
user.deleted
order.completed
order.cancelled
payment.succeeded
payment.failed
invoice.finalized
subscription.renewed
subscription.expired
```

This structure enables consumers to subscribe to specific event types or wildcard patterns (`order.*`).

### Thin vs. Fat Payloads

| Strategy | Payload | Trade-off |
|----------|---------|-----------|
| **Fat** (recommended) | Full resource representation in `data` | Larger payload but consumer can process without additional API calls |
| **Thin** | Only resource ID and event type | Smaller payload but consumer must GET the resource for details |

Fat payloads reduce N+1 API calls from consumers. Include the resource state at the moment the event occurred — not the current state, which may have changed between event creation and delivery.

---

## Registration API

Provide a REST API for consumers to manage their webhook subscriptions:

```yaml
paths:
  /webhooks:
    post:
      summary: Create webhook subscription
      operationId: create_webhook_subscription
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [url, events]
              properties:
                url:
                  type: string
                  format: uri
                  description: HTTPS endpoint to deliver events to
                events:
                  type: array
                  items: { type: string }
                  description: Event types to subscribe to
                  example: ["order.completed", "payment.failed"]
                secret:
                  type: string
                  description: Shared secret for HMAC signature verification (auto-generated if omitted)
      responses:
        "201":
          description: Subscription created
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:     { type: string, example: "wh_sub_123" }
                  url:    { type: string, format: uri }
                  events: { type: array, items: { type: string } }
                  secret: { type: string, description: "Shown once at creation time" }
                  status: { type: string, enum: [active, disabled] }

    get:
      summary: List webhook subscriptions
      operationId: list_webhook_subscriptions

  /webhooks/{id}:
    patch:
      summary: Update webhook subscription
      operationId: update_webhook_subscription

    delete:
      summary: Delete webhook subscription
      operationId: delete_webhook_subscription
```

### URL Validation

When a consumer registers a webhook URL:
1. Require HTTPS — never deliver to HTTP endpoints
2. Send a verification request (challenge) to confirm the consumer controls the URL
3. Reject private/reserved IP ranges to prevent SSRF attacks

---

## Delivery Architecture

Never deliver webhooks synchronously within the main request-response cycle. Use the **Outbox Pattern** for reliable delivery:

### Flow

```
API Request → Business Logic + Event Record (same DB transaction)
                        ↓
              Event Queue (Kafka, SQS, RabbitMQ)
                        ↓
              Delivery Workers (independent, scalable)
                        ↓
              HTTP POST to consumer URL
                        ↓
              Record delivery result (success/failure)
```

### Why Outbox Pattern

Writing the event record and the business state change in the same database transaction guarantees that events are never lost or created without the corresponding state change. The delivery worker reads from the queue and handles retries independently.

This decoupling ensures:
- A slow or failing consumer endpoint does not block API responses
- Delivery workers scale independently from the API
- Failed deliveries are retried without impacting other consumers
- Dead-letter queues capture permanently failed events for investigation

---

## Security: HMAC Signatures

Every webhook delivery must be cryptographically signed so consumers can verify authenticity and integrity.

### Signing Process (Producer)

1. Compute the HMAC-SHA256 of the raw request body using the subscription's shared secret
2. Include the signature, timestamp, and signing algorithm in headers:

```http
POST /webhook-receiver HTTP/1.1
Content-Type: application/json
X-Webhook-ID: evt_abc123def456
X-Webhook-Timestamp: 1718456200
X-Webhook-Signature: v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd

{
  "id": "evt_abc123def456",
  "type": "order.completed",
  ...
}
```

### Verification Process (Consumer)

1. Extract the timestamp from `X-Webhook-Timestamp`
2. Reject if the timestamp is older than 5 minutes (prevents replay attacks)
3. Construct the signing payload: `{timestamp}.{raw_body}`
4. Compute HMAC-SHA256 using the shared secret
5. Compare the computed signature with `X-Webhook-Signature` using **constant-time comparison** (prevents timing attacks)

### Signing Payload Format

```
signing_input = "{timestamp}.{raw_request_body}"
signature = HMAC-SHA256(secret, signing_input)
header_value = "v1={hex(signature)}"
```

The `v1=` prefix enables future algorithm migration without breaking existing consumers.

### Key Rotation

Support multiple active secrets during rotation:
1. Generate a new secret and add it to the subscription (both old and new are active)
2. Deliver events signed with both secrets (two signature values in the header)
3. Consumer verifies against either secret
4. After the consumer confirms the new secret works, revoke the old one

---

## Idempotency and At-Least-Once Delivery

Network failures make exactly-once delivery impossible in distributed systems. Webhooks use **at-least-once delivery** — the same event may be delivered multiple times. This makes consumer-side idempotency mandatory.

### Producer Responsibilities

- Include a stable, unique `id` field in every event
- Use the **same** event ID across all retry attempts
- Never regenerate the event payload on retry — send the exact same bytes

### Consumer Responsibilities

- Store processed event IDs (in Redis with TTL, or a database with unique constraint)
- Before processing, check if the event ID has already been handled
- If already processed, return `200 OK` immediately without re-executing business logic
- Make the idempotency check and business logic atomic (same transaction)

```
1. Receive webhook POST
2. Verify HMAC signature
3. Check: Has event ID been processed?
   → Yes: Return 200 OK (no-op)
   → No: Process event, store event ID, return 200 OK
```

---

## Retry Policy

### Exponential Backoff with Jitter

When delivery fails (non-2xx response or timeout), retry with exponential backoff:

| Attempt | Delay | Cumulative |
|---------|-------|-----------|
| 1 | Immediate | 0 |
| 2 | 1 min ± jitter | ~1 min |
| 3 | 5 min ± jitter | ~6 min |
| 4 | 30 min ± jitter | ~36 min |
| 5 | 2 hours ± jitter | ~2.5 hours |
| 6 | 8 hours ± jitter | ~10.5 hours |
| 7 (final) | 24 hours ± jitter | ~34.5 hours |

Jitter prevents thundering herd problems when many subscriptions fail simultaneously (e.g., during a consumer outage).

### Failure Handling

- After all retry attempts are exhausted, mark the event as permanently failed
- Move it to a **dead-letter queue** for manual review or automated reconciliation
- If a subscription accumulates repeated failures (e.g., 5 consecutive events fail), automatically **disable** the subscription and notify the consumer via email or dashboard

### Timeout

Set a delivery timeout of 5-10 seconds. If the consumer does not respond within this window, treat it as a failure and retry. Consumers should acknowledge quickly and process asynchronously.

---

## Consumer Guidelines

Document these guidelines for webhook consumers:

1. **Respond quickly** — Return `200 OK` within 5 seconds. Process the event asynchronously after acknowledging receipt.
2. **Verify signatures** — Always validate the HMAC signature before processing.
3. **Handle duplicates** — Implement idempotency using the event `id`.
4. **Reject stale events** — Check the timestamp and reject events older than 5 minutes.
5. **Use HTTPS** — Only register HTTPS endpoints.
6. **Expect retries** — A 200 response means "received and understood." Any non-2xx triggers a retry.
7. **Monitor your endpoint** — Track response times and error rates. Set up alerts for failures.

---

## Reconciliation API

Webhooks are not 100% reliable — network partitions, consumer downtime, and expired retries can cause missed events. Provide a reconciliation endpoint as the source of truth:

```http
GET /events?type=order.completed&created_after=2024-06-15T00:00:00Z&limit=100
```

This allows consumers to:
- Backfill events missed during downtime
- Verify their local state matches the API's state
- Reprocess events after a bug fix

The reconciliation API follows the same pagination patterns as other collection endpoints (cursor-based, with `has_more`).

---

## Observability

### Producer Dashboard

Provide consumers with a webhook delivery dashboard showing:
- Delivery history per subscription (timestamp, status code, latency)
- Retry status for pending events
- Manual retry trigger for failed deliveries
- Subscription health metrics (success rate, average latency)

### Producer-Side Monitoring

- Delivery success rate per consumer
- Average delivery latency
- Queue depth and backlog growth
- Dead-letter queue size
- Subscription disable rate (indicating persistent consumer issues)

---

## OpenAPI Integration

### Webhook Declaration (OpenAPI 3.1)

OpenAPI 3.1 supports a top-level `webhooks` object for documenting outbound events:

```yaml
webhooks:
  orderCompleted:
    post:
      summary: Order completed event
      operationId: onOrderCompleted
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/WebhookEvent"
            example:
              id: "evt_abc123"
              type: "order.completed"
              created_at: "2024-06-15T14:30:00Z"
              data:
                id: "ord_xyz789"
                status: "completed"
                total: 149.99
      responses:
        "200":
          description: Event acknowledged

components:
  schemas:
    WebhookEvent:
      type: object
      required: [id, type, created_at, data]
      properties:
        id:         { type: string, description: "Unique event ID, stable across retries" }
        type:       { type: string, description: "Event type (resource.action)" }
        created_at: { type: string, format: date-time }
        api_version: { type: string }
        data:       { type: object, description: "Event payload" }
```

---

## Antipatterns

| Antipattern | Problem | Fix |
|------------|---------|-----|
| Synchronous delivery in the request handler | Blocks API response, causes timeouts | Use outbox pattern with async delivery workers |
| No HMAC signatures | Consumers can't verify authenticity | Sign every delivery with HMAC-SHA256 |
| Regenerating payload on retry | Consumer idempotency breaks (different body = different hash) | Send identical bytes on every retry |
| Delivering to HTTP (not HTTPS) | Payload exposed in transit | Require HTTPS for all webhook URLs |
| No event ID | Consumers can't deduplicate | Include a stable, unique `id` in every event |
| No retry or backoff | Single failure = permanent data loss | Exponential backoff with dead-letter queue |
| No reconciliation API | No recovery path for missed events | Provide a queryable event history endpoint |
| Fat payloads with current state | Consumer processes stale data | Include resource state at event time, not current state |
