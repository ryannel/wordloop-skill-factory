# Advanced Patterns: Expansion, Bulk Operations, File Uploads & Operational Endpoints

## Table of Contents

- [Resource Expansion](#resource-expansion)
- [Bulk and Batch Operations](#bulk-and-batch-operations)
- [File Upload Patterns](#file-upload-patterns)
- [Health Check Endpoints](#health-check-endpoints)
- [CORS Configuration](#cors-configuration)
- [Response Compression](#response-compression)
- [Collection-Level Caching](#collection-level-caching)

---

## Resource Expansion

The `expand` pattern allows clients to request that the server inline related resources within a single response, eliminating the N+1 API call problem. By default, relationships return only an identifier; with expansion, the full object is embedded.

### Default Response (No Expansion)

```json
{
  "id": "ord_abc123",
  "status": "completed",
  "total": 149.99,
  "customer": "usr_xyz789",
  "items": ["itm_001", "itm_002"]
}
```

### Expanded Response

```http
GET /orders/ord_abc123?expand=customer,items
```

```json
{
  "id": "ord_abc123",
  "status": "completed",
  "total": 149.99,
  "customer": {
    "id": "usr_xyz789",
    "name": "Jane Smith",
    "email": "jane@example.com"
  },
  "items": [
    { "id": "itm_001", "name": "Widget", "quantity": 2, "price": 49.99 },
    { "id": "itm_002", "name": "Gadget", "quantity": 1, "price": 50.01 }
  ]
}
```

### Syntax Variants

| Syntax | Example | Use Case |
|--------|---------|----------|
| Comma-separated | `?expand=customer,items` | Simple, flat expansion |
| Array notation | `?expand[]=customer&expand[]=items` | Explicit per-field expansion |
| Dot notation | `?expand=items.product` | Nested/recursive expansion |
| Collection prefix | `?expand=data.customer` | Expanding fields within list items |

### Implementation Rules

1. **Opt-in only** — Default responses return IDs only. Expansion is always explicit.
2. **Batch backend queries** — Collect all IDs needing expansion and fetch them in a single `WHERE id IN (...)` query. Never execute one query per expanded object (the DataLoader pattern).
3. **Enforce depth limits** — Cap recursive expansion at 2-4 levels to prevent server overload. Return `400 Bad Request` if the limit is exceeded.
4. **Document expandable fields** — Use an OpenAPI vendor extension (`x-expandable-fields`) or document in the schema description which fields support expansion.
5. **Rate expansion on collections** — Expanding fields on list endpoints multiplies backend queries. Consider limiting expansion depth on collection endpoints or requiring cursor pagination when expansion is used.

### OpenAPI Schema

```yaml
parameters:
  ExpandParam:
    name: expand
    in: query
    description: >
      Comma-separated list of fields to expand. Expandable fields:
      customer, items, items.product.
    schema:
      type: array
      items: { type: string }
    style: form
    explode: true
```

### Expand vs. Sparse Fieldsets

These patterns are complementary:
- **`expand`** adds data — inlines related resources that are normally IDs
- **`fields`** removes data — returns only the requested properties

Combined: `GET /orders/123?expand=customer&fields=id,total,customer.name,customer.email`

---

## Bulk and Batch Operations

Bulk operations process multiple resources in a single API call. They reduce HTTP overhead, improve throughput, and enable transactional semantics when needed.

### Endpoint Design

Use a dedicated sub-path to keep bulk operations separate from standard CRUD:

```
POST /users/bulk           # Bulk create
PATCH /users/bulk          # Bulk update
DELETE /users/bulk          # Bulk delete
POST /users/bulk/import    # Async import from file
```

Do not overload the standard `POST /users` endpoint with array payloads — this breaks client expectations and complicates error handling.

### Request Format

```http
POST /users/bulk
Content-Type: application/json

{
  "items": [
    { "email": "alice@example.com", "name": "Alice" },
    { "email": "bob@example.com", "name": "Bob" },
    { "email": "invalid-email", "name": "Charlie" }
  ]
}
```

### Partial Success: 207 Multi-Status

When individual items within a batch can succeed or fail independently, return `207 Multi-Status` with per-item results:

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "results": [
    {
      "index": 0,
      "status": 201,
      "data": { "id": "usr_001", "email": "alice@example.com", "name": "Alice" }
    },
    {
      "index": 1,
      "status": 201,
      "data": { "id": "usr_002", "email": "bob@example.com", "name": "Bob" }
    },
    {
      "index": 2,
      "status": 422,
      "error": {
        "type": "https://api.example.com/errors/validation-error",
        "title": "Validation Error",
        "status": 422,
        "detail": "Invalid email format.",
        "errors": [
          { "pointer": "/email", "code": "INVALID_FORMAT", "detail": "Must be a valid email address." }
        ]
      }
    }
  ],
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  }
}
```

### Atomic vs. Non-Atomic

| Mode | Behavior | Status Code | Use Case |
|------|----------|-------------|----------|
| **Non-atomic** (default) | Each item processed independently; partial success allowed | `207 Multi-Status` | Bulk imports, data migration |
| **Atomic** | All items succeed or all fail (transaction) | `200 OK` or `422 Unprocessable Entity` | Financial transactions, linked records |

Allow clients to choose via a query parameter:

```http
POST /users/bulk?atomic=true
```

### Async Bulk Operations

For large batches (hundreds or thousands of items), return `202 Accepted` and process asynchronously:

```http
POST /users/bulk/import
Content-Type: application/json

{ "items": [ ... 500 items ... ] }

HTTP/1.1 202 Accepted
Location: /jobs/job_abc123

{
  "job_id": "job_abc123",
  "status": "processing",
  "progress": 0
}
```

The client polls the job status endpoint until completion (see `http-semantics.md` for the async pattern).

### Batch Size Limits

Always enforce maximum batch sizes to prevent memory exhaustion and timeout issues:

```yaml
x-batch-limits:
  max_items: 100
  max_payload_bytes: 10485760  # 10MB
```

Return `400 Bad Request` if the limit is exceeded:

```json
{
  "type": "https://api.example.com/errors/batch-too-large",
  "title": "Batch Too Large",
  "status": 400,
  "detail": "Batch contains 250 items. Maximum allowed is 100."
}
```

---

## File Upload Patterns

### Strategy Selection

| File Size | Pattern | Mechanism |
|-----------|---------|-----------|
| Small (< 10 MB) | Direct upload | `POST` with `multipart/form-data` or `application/octet-stream` |
| Large (10 MB – 5 GB) | Presigned URL | Server generates a time-limited upload URL to cloud storage |
| Very large (> 5 GB) or unstable network | Resumable multipart | Server coordinates chunked upload with cloud storage |

### Direct Upload (Small Files)

```yaml
paths:
  /users/{id}/avatar:
    put:
      summary: Upload user avatar
      operationId: upload_user_avatar
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              required: [file]
              properties:
                file:
                  type: string
                  format: binary
                  description: Image file (JPEG, PNG). Max 5MB.
            encoding:
              file:
                contentType: image/jpeg, image/png
      responses:
        "200":
          description: Avatar uploaded
          content:
            application/json:
              schema:
                type: object
                properties:
                  url: { type: string, format: uri }
        "413":
          description: File too large
```

### Presigned URL Upload (Large Files)

The server generates a time-limited, pre-authorized URL for direct upload to cloud storage (S3, GCS). The file never passes through the application server.

**Step 1: Request upload URL**

```http
POST /uploads
Content-Type: application/json

{
  "filename": "report.pdf",
  "content_type": "application/pdf",
  "size_bytes": 52428800
}

HTTP/1.1 201 Created
{
  "upload_id": "upl_abc123",
  "upload_url": "https://storage.example.com/bucket/obj?X-Amz-Signature=...",
  "expires_at": "2024-06-15T15:00:00Z"
}
```

**Step 2: Upload directly to storage**

```http
PUT https://storage.example.com/bucket/obj?X-Amz-Signature=...
Content-Type: application/pdf

<binary file data>
```

**Step 3: Confirm upload**

```http
POST /uploads/upl_abc123/complete

HTTP/1.1 200 OK
{
  "file_id": "file_xyz789",
  "url": "https://cdn.example.com/files/file_xyz789",
  "size_bytes": 52428800,
  "content_type": "application/pdf"
}
```

### Resumable Multipart Upload

For files over 100 MB or unreliable networks, split the file into chunks:

1. **Initiate**: `POST /uploads` → returns `upload_id` and chunk size
2. **Upload chunks**: `PUT /uploads/{id}/parts/{partNumber}` → upload each chunk
3. **Finalize**: `POST /uploads/{id}/complete` → server assembles the file

Each chunk can be retried independently, and uploads can be paused and resumed.

### Security

- Never trust client-provided `Content-Type` — validate server-side after upload
- Scan uploaded files for malware before making them accessible
- Enforce file size limits at the API layer (`413 Payload Too Large`)
- Set short expiry times on presigned URLs (15-60 minutes)
- Implement lifecycle policies to clean up abandoned uploads

---

## Health Check Endpoints

Production APIs need standardized health check endpoints for orchestration platforms (Kubernetes, ECS), load balancers, and monitoring systems.

### Endpoint Design

| Endpoint | Purpose | Checks | Failure Consequence |
|----------|---------|--------|-------------------|
| `GET /health/live` | Is the process alive? | Process is running and responsive | Container restart |
| `GET /health/ready` | Can it serve traffic? | Database, cache, and critical dependencies reachable | Removed from load balancer |
| `GET /health/startup` | Has initialization completed? | Config loaded, migrations run, caches warmed | Delays liveness/readiness probes |

### Liveness: Keep It Simple

The liveness check answers one question: "Is this process stuck?" It should be a minimal, zero-dependency check that always responds if the server is running.

```http
GET /health/live

HTTP/1.1 200 OK
Content-Type: application/json

{ "status": "UP" }
```

**Never check external dependencies in liveness probes.** If your database goes down and every liveness probe fails, the orchestrator restarts all pods simultaneously — making recovery impossible (thundering herd).

### Readiness: Be Comprehensive

The readiness check confirms the service can handle requests. Check critical dependencies:

```http
GET /health/ready

HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "UP",
  "components": {
    "database": { "status": "UP", "latency_ms": 3 },
    "cache": { "status": "UP", "latency_ms": 1 },
    "search": { "status": "DOWN", "error": "Connection refused" }
  }
}
```

When a critical component is down:

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "status": "DOWN",
  "components": {
    "database": { "status": "DOWN", "error": "Connection timeout" }
  }
}
```

### Security

- Do not expose sensitive diagnostic information (connection strings, hostnames) in public-facing health checks
- Consider serving health endpoints on a separate internal port or restricting access via network policy
- Keep health check responses lightweight — they run frequently (every 5-10 seconds)

### OpenAPI Schema

```yaml
paths:
  /health/live:
    get:
      summary: Liveness probe
      operationId: check_liveness
      tags: [Operations]
      security: []   # No auth required
      responses:
        "200":
          description: Process is alive
          content:
            application/json:
              schema: { $ref: "#/components/schemas/HealthStatus" }

  /health/ready:
    get:
      summary: Readiness probe
      operationId: check_readiness
      tags: [Operations]
      security: []
      responses:
        "200":
          description: Ready to serve traffic
          content:
            application/json:
              schema: { $ref: "#/components/schemas/HealthStatus" }
        "503":
          description: Not ready
          content:
            application/json:
              schema: { $ref: "#/components/schemas/HealthStatus" }

components:
  schemas:
    HealthStatus:
      type: object
      required: [status]
      properties:
        status:
          type: string
          enum: [UP, DOWN]
        components:
          type: object
          additionalProperties:
            type: object
            properties:
              status:     { type: string, enum: [UP, DOWN] }
              latency_ms: { type: integer }
              error:      { type: string }
```

---

## CORS Configuration

Cross-Origin Resource Sharing governs which browser-based applications can call your API. CORS is a browser-level mechanism — it does not affect server-to-server requests, `curl`, or mobile apps. It is not a security feature; all authentication and authorization must be enforced independently.

### Essential Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Access-Control-Allow-Origin` | Which origin may read the response | `https://app.example.com` |
| `Access-Control-Allow-Methods` | Allowed HTTP methods | `GET, POST, PUT, PATCH, DELETE` |
| `Access-Control-Allow-Headers` | Allowed custom request headers | `Authorization, Content-Type, X-Request-ID` |
| `Access-Control-Max-Age` | How long to cache preflight results (seconds) | `86400` (24 hours) |
| `Access-Control-Allow-Credentials` | Allow cookies/auth headers | `true` |
| `Vary` | Prevent cache poisoning | `Origin` |

### Preflight Handling

Non-simple requests (PUT, DELETE, PATCH, custom headers, `application/json` content type) trigger a preflight `OPTIONS` request. The server must respond correctly:

```http
OPTIONS /users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: Authorization, Content-Type

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type, X-Request-ID
Access-Control-Max-Age: 86400
Vary: Origin
```

Cache preflight results with `Access-Control-Max-Age` to avoid redundant OPTIONS requests on every API call.

### Configuration Rules

1. **Never use `*` with credentials** — Browsers reject `Access-Control-Allow-Origin: *` when `Access-Control-Allow-Credentials: true`
2. **Whitelist specific origins** — Validate the `Origin` header against a strict allow-list. Never blindly reflect the Origin header back.
3. **Always include `Vary: Origin`** — When dynamically setting the Allow-Origin header, `Vary: Origin` prevents CDN cache poisoning
4. **Environment-specific config** — Dev (`localhost:3000`), staging, and production environments need different origin lists
5. **Log rejected origins** — Monitor blocked cross-origin requests for misconfigurations or probes

### Antipatterns

| Antipattern | Risk |
|------------|------|
| `Access-Control-Allow-Origin: *` for authenticated APIs | Any site can read responses with user credentials |
| Reflecting Origin header without validation | Effectively disables CORS protection |
| No `Vary: Origin` on dynamic responses | Cache serves wrong origin's CORS headers |
| Relying on CORS as a security boundary | CORS only constrains browsers, not other clients |

---

## Response Compression

Compress JSON responses to reduce bandwidth and improve transfer times, especially for mobile clients and large payloads.

### Content Negotiation

```http
GET /users?limit=100
Accept-Encoding: gzip, br

HTTP/1.1 200 OK
Content-Encoding: br
Vary: Accept-Encoding
Content-Type: application/json
```

### Algorithm Priority

| Algorithm | Compression Ratio | Speed | Compatibility | Recommendation |
|-----------|-------------------|-------|--------------|----------------|
| Brotli (`br`) | Excellent (15-30% smaller than gzip) | Variable | All modern clients | Primary — use for JSON/text |
| Gzip (`gzip`) | Good | Very fast | Universal | Fallback — for legacy clients |
| Zstandard (`zstd`) | Excellent | Very fast | Limited | Consider for internal APIs |

### Implementation Rules

1. **Negotiate via headers** — Respect `Accept-Encoding` and respond with `Content-Encoding`
2. **Include `Vary: Accept-Encoding`** — Prevent caches from serving compressed responses to clients that can't decompress
3. **Set a minimum threshold** — Do not compress responses smaller than ~1 KB. The compression overhead can produce larger output for tiny payloads.
4. **Never double-compress** — Binary content (images, videos, archives) is already compressed. Compressing again wastes CPU and may increase size.
5. **Offload to edge** — Enable compression at the reverse proxy or CDN layer (Nginx, Cloudflare) to save application server CPU
6. **Support request body compression** — For large request bodies (bulk operations, file metadata), accept `Content-Encoding` on requests

---

## Collection-Level Caching

The `http-semantics.md` reference covers single-resource ETags. Collections require additional strategies because their state depends on multiple underlying entities.

### Weak ETags for Collections

Use weak ETags (`W/"..."`) for collection responses. Weak ETags indicate semantic equivalence — the same data content even if the byte representation differs slightly (e.g., different serialization order):

```http
GET /users?status=active&limit=20

HTTP/1.1 200 OK
ETag: W/"collection-v42"
Cache-Control: private, max-age=60
Vary: Accept-Encoding, Authorization
```

### ETag Generation Strategies

| Strategy | Implementation | Trade-off |
|----------|---------------|-----------|
| **Max timestamp** | Hash of the most recent `updated_at` across all matching records | Efficient, catches most changes |
| **Version counter** | Increment a counter on every write to the collection | Very fast lookup, requires write coordination |
| **Query hash** | Hash of the full query result set | Accurate but expensive to compute |

### Conditional Request Flow

```http
GET /users?status=active
If-None-Match: W/"collection-v42"

# If collection hasn't changed:
HTTP/1.1 304 Not Modified

# If collection has changed:
HTTP/1.1 200 OK
ETag: W/"collection-v43"
{ "data": [...], "pagination": {...} }
```

### Cache-Control Strategy for Collections

| Collection Type | Cache-Control | Rationale |
|----------------|---------------|-----------|
| Public reference data (countries, currencies) | `public, max-age=86400` | Changes rarely |
| User-specific data (my orders) | `private, max-age=60` | Per-user, changes occasionally |
| Real-time feed (notifications) | `no-store` | Always fresh |
| Shared listing (product catalog) | `public, max-age=300` | Changes moderately, acceptable staleness |

### Invalidation

When a resource within a collection is created, updated, or deleted:
- Increment the collection's version counter or update its timestamp
- CDN/proxy caches expire based on `max-age` or `s-maxage`
- Clients holding stale ETags will receive the full response on next conditional request

Avoid proactive invalidation (purging caches) unless latency requirements demand it. TTL-based expiry with conditional requests is simpler and sufficient for most APIs.
