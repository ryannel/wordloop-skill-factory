# HTTP Method Semantics, Idempotency & Async Patterns

## Table of Contents

- [Method Semantics](#method-semantics)
- [Safety and Idempotency](#safety-and-idempotency)
- [Idempotency Keys](#idempotency-keys)
- [Status Code Usage](#status-code-usage)
- [Asynchronous Operations](#asynchronous-operations)
- [Cache Control](#cache-control)
- [Conditional Requests](#conditional-requests)

---

## Method Semantics

HTTP methods carry precise semantics. Using them correctly enables network intermediaries, caches, and client libraries to make automatic decisions about retries, prefetching, and safety.

### GET — Retrieve Resources

GET is **safe** (never modifies state) and **idempotent** (multiple identical requests produce the same result). Caches, crawlers, and prefetch mechanisms rely on this guarantee.

```http
GET /users/123
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}
```

Never modify server state in response to a GET. If a "read" operation has side effects (e.g., generating a report), use POST.

### POST — Create Resources

POST is **unsafe** and **non-idempotent** by default. Each identical POST may create a new resource. The server returns `201 Created` with a `Location` header pointing to the new resource.

```http
POST /users
Content-Type: application/json

{
  "name": "Jane Smith",
  "email": "jane@example.com"
}

HTTP/1.1 201 Created
Location: /users/usr_abc456
Content-Type: application/json

{
  "id": "usr_abc456",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "created_at": "2024-01-16T14:20:00Z"
}
```

### PUT — Replace Entire Resource

PUT is **idempotent**. Sending the same PUT request multiple times results in the same resource state. Use PUT when the client sends the complete representation.

```http
PUT /users/123
Content-Type: application/json

{
  "name": "John Doe Updated",
  "email": "john.new@example.com"
}

HTTP/1.1 200 OK
```

If the client sends a PUT with missing fields, those fields are set to null/default — PUT replaces the entire resource, it does not merge.

### PATCH — Partial Update

PATCH sends only the fields that changed. While technically non-idempotent per the HTTP specification, design PATCH operations to behave idempotently in practice for reliable distributed systems.

```http
PATCH /users/123
Content-Type: application/json

{
  "email": "john.updated@example.com"
}

HTTP/1.1 200 OK
```

### DELETE — Remove Resource

DELETE is **idempotent**. The first DELETE returns `204 No Content`; subsequent DELETEs to the same URI return `404 Not Found` (or `204` — both are acceptable), but the server state remains unchanged after the first call.

```http
DELETE /users/123

HTTP/1.1 204 No Content
```

### HEAD — Metadata Only

HEAD is identical to GET but returns only headers, no body. Use for existence checks, cache validation, and metadata retrieval without transferring the payload.

```http
HEAD /users/123

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 234
ETag: "33a64df5"
```

### OPTIONS — Allowed Methods

OPTIONS returns the HTTP methods supported for a given URI. Primarily used in CORS preflight requests.

```http
OPTIONS /users/123

HTTP/1.1 204 No Content
Allow: GET, PUT, PATCH, DELETE, HEAD, OPTIONS
```

---

## Safety and Idempotency

| Method | Safe | Idempotent | Request Body | Response Body |
|--------|------|------------|-------------|--------------|
| GET | ✓ | ✓ | No | Yes |
| HEAD | ✓ | ✓ | No | No |
| OPTIONS | ✓ | ✓ | No | Optional |
| POST | ✗ | ✗ | Yes | Yes |
| PUT | ✗ | ✓ | Yes | Optional |
| PATCH | ✗ | ✗* | Yes | Yes |
| DELETE | ✗ | ✓ | No | Optional |

*Design PATCH to be idempotent in practice.

**Safe** means the method does not modify server state. Safe methods can be prefetched, cached, and retried freely.

**Idempotent** means repeated identical requests produce the same server state. Idempotent methods can be retried on network failure without risk of duplication.

---

## Idempotency Keys

For operations that are inherently non-idempotent but dangerous to duplicate (payments, order submissions, account creation), use an `Idempotency-Key` header.

### How It Works

1. The client generates a unique key (UUID v4) and sends it with the request
2. The server checks if a result for this key already exists
3. If yes, the server returns the cached result without re-executing the operation
4. If no, the server processes the request and caches the result keyed by the idempotency key

### Example

```http
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "amount": 100.00,
  "currency": "USD",
  "recipient": "usr_xyz789"
}

HTTP/1.1 201 Created
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

### Server-Side Implementation

- Cache idempotency key → response mapping for a defined TTL (typically 24-48 hours)
- Use a database-level unique constraint or distributed lock on the idempotency key
- Return the exact same response (status code, headers, body) for duplicate keys
- If the original request is still processing, return `409 Conflict` with a retry hint

### OpenAPI Schema for Idempotency Key

```yaml
parameters:
  IdempotencyKey:
    name: Idempotency-Key
    in: header
    required: true
    description: Unique key for idempotent request processing. Use UUID v4.
    schema:
      type: string
      format: uuid
      example: "550e8400-e29b-41d4-a716-446655440000"
```

---

## Status Code Usage

### Success (2xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| `200 OK` | Request succeeded | GET, PUT, PATCH responses with a body |
| `201 Created` | Resource created | POST; include `Location` header |
| `202 Accepted` | Request accepted for async processing | Long-running operations |
| `204 No Content` | Success, no body | DELETE; PUT/PATCH when no body is needed |

### Redirection (3xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| `301 Moved Permanently` | Resource permanently relocated | Endpoint migration |
| `304 Not Modified` | Cache is still valid | Conditional GET with ETag/If-None-Match |

### Client Errors (4xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| `400 Bad Request` | Malformed syntax | Unparseable JSON, missing required headers |
| `401 Unauthorized` | Authentication required or failed | Missing/invalid/expired token |
| `403 Forbidden` | Authenticated but not authorized | Insufficient permissions |
| `404 Not Found` | Resource does not exist | Unknown resource ID |
| `405 Method Not Allowed` | HTTP method not supported | PUT on a read-only resource |
| `409 Conflict` | State conflict | Duplicate resource, concurrent modification |
| `410 Gone` | Resource permanently removed | Sunset endpoints |
| `422 Unprocessable Entity` | Semantic validation failure | Valid JSON but business rule violation |
| `429 Too Many Requests` | Rate limit exceeded | Include `Retry-After` header |

### Server Errors (5xx)

| Code | Meaning | When to Use |
|------|---------|-------------|
| `500 Internal Server Error` | Unexpected failure | Unhandled exception (never expose stack traces) |
| `502 Bad Gateway` | Upstream service failure | Invalid response from dependency |
| `503 Service Unavailable` | Temporary unavailability | Maintenance, overload; include `Retry-After` |
| `504 Gateway Timeout` | Upstream timeout | Dependency did not respond in time |

---

## Asynchronous Operations

For tasks that take longer than a few seconds (report generation, batch processing, data migrations), use the **Asynchronous Request-Reply** pattern.

### Flow

```
Client                          Server
  |                               |
  |  POST /reports                |
  |------------------------------>|
  |                               | (starts processing)
  |  202 Accepted                 |
  |  Location: /jobs/job_abc123   |
  |<------------------------------|
  |                               |
  |  GET /jobs/job_abc123         |
  |------------------------------>|
  |  200 OK                       |
  |  { "status": "processing",   |
  |    "progress": 45 }          |
  |<------------------------------|
  |                               |
  |  GET /jobs/job_abc123         |
  |------------------------------>|
  |  200 OK                       |
  |  { "status": "completed",    |
  |    "result": "/reports/rpt_1"}|
  |<------------------------------|
  |                               |
  |  GET /reports/rpt_1           |
  |------------------------------>|
  |  200 OK { ... report data }   |
  |<------------------------------|
```

### Status Resource Schema

```yaml
JobStatus:
  type: object
  required: [id, status, created_at]
  properties:
    id:
      type: string
      example: "job_abc123"
    status:
      type: string
      enum: [pending, processing, completed, failed]
    progress:
      type: integer
      minimum: 0
      maximum: 100
      description: Completion percentage
    result:
      type: string
      format: uri
      description: URI to the completed resource (present when status is completed)
    error:
      $ref: "#/components/schemas/ProblemDetail"
      description: Error details (present when status is failed)
    created_at:
      type: string
      format: date-time
    updated_at:
      type: string
      format: date-time
```

---

## Cache Control

### Response Headers

```http
Cache-Control: public, max-age=3600                      # Cacheable for 1 hour
Cache-Control: private, no-cache                          # Must revalidate
Cache-Control: no-store                                   # Never cache (sensitive data)
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"         # Entity tag for validation
Last-Modified: Wed, 15 Jan 2024 10:30:00 GMT             # Last modification time
Vary: Accept, Authorization                               # Cache varies by these headers
```

### Cache Strategy by Resource Type

| Resource Type | Cache-Control | ETag | Rationale |
|--------------|---------------|------|-----------|
| Static reference data | `public, max-age=86400` | ✓ | Changes rarely |
| User profile | `private, max-age=300` | ✓ | User-specific, changes occasionally |
| Real-time feed | `no-store` | ✗ | Always fresh |
| Collection listing | `public, max-age=60` | ✓ | Changes moderately |

---

## Conditional Requests

### Read: If-None-Match (ETag)

```http
GET /users/123
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

HTTP/1.1 304 Not Modified
```

Saves bandwidth by returning no body when the resource hasn't changed.

### Write: If-Match (Optimistic Concurrency)

```http
PUT /users/123
If-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Content-Type: application/json

{
  "name": "Updated Name"
}

HTTP/1.1 200 OK                          # ETag matched, update applied
```

```http
PUT /users/123
If-Match: "stale-etag-value"
Content-Type: application/json

{
  "name": "Updated Name"
}

HTTP/1.1 412 Precondition Failed         # Someone else updated the resource
```

Use `If-Match` for concurrent update protection. This prevents the "lost update" problem in multi-client environments.
