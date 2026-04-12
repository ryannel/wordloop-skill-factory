# Error Handling with RFC 9457 Problem Details

## Table of Contents

- [Why RFC 9457](#why-rfc-9457)
- [Problem Details Anatomy](#problem-details-anatomy)
- [Error Categories](#error-categories)
- [Validation Error Extensions](#validation-error-extensions)
- [Error Code Catalog](#error-code-catalog)
- [Request ID Tracking](#request-id-tracking)
- [Retry Guidance](#retry-guidance)
- [Internationalization](#internationalization)
- [OpenAPI Integration](#openapi-integration)
- [Antipatterns](#antipatterns)

---

## Why RFC 9457

RFC 9457 (superseding RFC 7807) defines "Problem Details for HTTP APIs" — a standard schema using `application/problem+json`. It eliminates the "invent a new error format per project" problem and enables clients, developers, AI agents, and tooling to parse errors using a known contract.

---

## Problem Details Anatomy

```json
{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with ID 'usr_99999' does not exist.",
  "instance": "/users/usr_99999"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | URI string | Yes | Stable, documented URI identifying the error class |
| `title` | string | Yes | Short summary — does not change between occurrences |
| `status` | integer | Yes | HTTP status code |
| `detail` | string | No | Occurrence-specific, human-readable, actionable explanation |
| `instance` | URI string | No | Identifies this specific occurrence for log correlation |

Extension fields (e.g., `errors[]`, `request_id`) are added alongside core fields.

---

## Error Categories

### 400 Bad Request — Malformed Input

```json
{
  "type": "https://api.example.com/errors/malformed-request",
  "title": "Malformed Request",
  "status": 400,
  "detail": "Request body is not valid JSON. Unexpected token at position 42."
}
```

### 401 Unauthorized — Authentication Required

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token"
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.com/errors/authentication-failed",
  "title": "Authentication Failed",
  "status": 401,
  "detail": "The access token has expired. Obtain a new token from the authorization endpoint."
}
```

Always include the `WWW-Authenticate` header with 401 responses.

### 403 Forbidden — Insufficient Permissions

```json
{
  "type": "https://api.example.com/errors/insufficient-permissions",
  "title": "Insufficient Permissions",
  "status": 403,
  "detail": "The 'users:delete' scope is required. Your token has: ['users:read', 'users:write']."
}
```

### 404 Not Found

```json
{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with ID 'usr_99999' does not exist."
}
```

### 409 Conflict — State Conflict

```json
{
  "type": "https://api.example.com/errors/resource-already-exists",
  "title": "Resource Already Exists",
  "status": 409,
  "detail": "A user with email 'john@example.com' already exists.",
  "existing_resource": "/users/usr_123"
}
```

### 410 Gone — Permanently Removed (Sunset)

```json
{
  "type": "https://api.example.com/errors/endpoint-sunset",
  "title": "Endpoint Removed",
  "status": 410,
  "detail": "This API version was decommissioned. Migrate to v2.",
  "migration_guide": "https://api.example.com/docs/migration/v1-to-v2"
}
```

### 422 Unprocessable Entity — Semantic Validation Failure

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Request validation failed.",
  "errors": [
    { "pointer": "/email", "code": "INVALID_FORMAT", "detail": "Must be a valid email address." },
    { "pointer": "/age", "code": "OUT_OF_RANGE", "detail": "Must be between 18 and 120.", "constraints": { "min": 18, "max": 120 } }
  ]
}
```

### 429 Too Many Requests

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
RateLimit: limit=100, remaining=0, reset=60
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "You have exceeded the rate limit of 100 requests per hour. Retry after 60 seconds."
}
```

### 500 Internal Server Error

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred. Reference ID: req_abc123.",
  "request_id": "req_abc123"
}
```

Never expose stack traces, database errors, or internal paths.

---

## Validation Error Extensions

### Field-Level with JSON Pointers (RFC 6901)

```json
{
  "errors": [
    {
      "pointer": "/billing_address/zip_code",
      "code": "INVALID_FORMAT",
      "detail": "ZIP code must be 5 digits.",
      "constraints": { "pattern": "^[0-9]{5}$" }
    },
    {
      "pointer": "/items/0/quantity",
      "code": "OUT_OF_RANGE",
      "detail": "Quantity must be at least 1.",
      "constraints": { "min": 1, "max": 1000 }
    }
  ]
}
```

### Cross-Field Validation

```json
{
  "errors": [
    {
      "pointer": "/end_date",
      "code": "INVALID_RANGE",
      "detail": "End date must be after start date.",
      "related_fields": ["/start_date", "/end_date"]
    }
  ]
}
```

Always validate the entire request and return all errors at once — never one error at a time.

---

## Error Code Catalog

| Error Code | HTTP Status | Description |
|-----------|-------------|-------------|
| `MALFORMED_REQUEST` | 400 | Request body is unparseable |
| `VALIDATION_ERROR` | 422 | Field validation failed |
| `AUTHENTICATION_FAILED` | 401 | Missing or invalid credentials |
| `TOKEN_EXPIRED` | 401 | Token has expired |
| `INSUFFICIENT_PERMISSIONS` | 403 | Missing required scope |
| `RESOURCE_NOT_FOUND` | 404 | Resource does not exist |
| `RESOURCE_ALREADY_EXISTS` | 409 | Duplicate resource |
| `CONCURRENT_MODIFICATION` | 409 | Resource modified between read and write |
| `ENDPOINT_SUNSET` | 410 | Endpoint permanently removed |
| `RATE_LIMIT_EXCEEDED` | 429 | Quota exhausted |
| `INTERNAL_ERROR` | 500 | Unexpected server failure |
| `SERVICE_UNAVAILABLE` | 503 | Temporary unavailability |

Adding new codes is non-breaking; removing or renaming existing codes is breaking.

---

## Request ID Tracking

Include a unique request ID in every response via `X-Request-ID` header and in error bodies:

```json
{ "request_id": "req_abc123" }
```

Generate server-side (UUID v7 or ULID for sortability). Echo client-provided `X-Request-ID` if present.

---

## Retry Guidance

### Retryable Errors

| Status | Strategy |
|--------|----------|
| 408 | Retry immediately |
| 429 | Wait for `Retry-After` |
| 500, 502, 504 | Exponential backoff |
| 503 | Wait for `Retry-After` |

### Non-Retryable Errors

400, 401, 403, 404, 409, 422 — Client must fix the request.

---

## Internationalization

Support `Accept-Language` for localized messages:

```http
GET /users/invalid
Accept-Language: es

HTTP/1.1 404 Not Found
Content-Language: es
```

Always include machine-readable `type` and `code` so clients can implement their own translations.

---

## OpenAPI Integration

### ProblemDetail Schema

```yaml
components:
  schemas:
    ProblemDetail:
      type: object
      required: [type, title, status]
      properties:
        type:       { type: string, format: uri, example: "https://api.example.com/errors/validation-error" }
        title:      { type: string, example: "Validation Error" }
        status:     { type: integer, example: 422 }
        detail:     { type: string, example: "The 'email' field must be a valid email address." }
        instance:   { type: string, format: uri }
        request_id: { type: string, example: "req_abc123" }
        errors:
          type: array
          items: { $ref: "#/components/schemas/FieldError" }

    FieldError:
      type: object
      required: [pointer, detail]
      properties:
        pointer:     { type: string, example: "/email" }
        code:        { type: string, example: "INVALID_FORMAT" }
        detail:      { type: string, example: "Must be a valid email address." }
        constraints: { type: object, additionalProperties: true }
```

---

## Antipatterns

| Antipattern | Fix |
|------------|-----|
| Returning 200 for errors | Use appropriate 4xx/5xx status codes |
| Generic "Error occurred" messages | Include specific `detail` and error codes |
| Exposing stack traces | Log internally, return sanitized `detail` |
| Inconsistent error shapes | Use RFC 9457 ProblemDetail everywhere |
| One validation error at a time | Return all errors in `errors[]` |
| Missing request IDs | Include `request_id` in every response |
