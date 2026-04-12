# Security Architecture & Rate Limiting

## Table of Contents

- [Zero Trust Foundation](#zero-trust-foundation)
- [OAuth 2.1 with PKCE](#oauth-21-with-pkce)
- [Mutual TLS for Service Communication](#mutual-tls-for-service-communication)
- [Rate Limiting with IETF Headers](#rate-limiting-with-ietf-headers)
- [Transport Security](#transport-security)
- [Fine-Grained Authorization](#fine-grained-authorization)
- [API Key Management](#api-key-management)
- [OpenAPI Security Schemes](#openapi-security-schemes)
- [Security Antipatterns](#security-antipatterns)

---

## Zero Trust Foundation

Every request is authenticated, authorized, and validated regardless of origin. There is no trusted perimeter — internal services authenticate with the same rigor as external clients.

| Security Layer | Protocol | Purpose |
|---------------|----------|---------|
| User-delegated access | OAuth 2.1 + PKCE | Secure authorization for browsers, mobile, SPAs |
| Service-to-service | Mutual TLS (mTLS) | Cryptographic identity for internal communication |
| Data in transit | TLS 1.3 | Mandatory encryption for all traffic |
| Traffic control | IETF RateLimit headers | Transparent quota management and abuse prevention |

---

## OAuth 2.1 with PKCE

OAuth 2.1 consolidates OAuth 2.0 best practices into a single specification. It eliminates insecure flows (Implicit Grant, Resource Owner Password Credentials) and mandates PKCE for all client types.

### Authorization Code Flow with PKCE

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks. It is mandatory for all client types — public clients (SPAs, mobile apps) and confidential clients (server applications).

```
Client                          Auth Server                    API Server
  |                                  |                              |
  | 1. Generate code_verifier        |                              |
  |    + code_challenge              |                              |
  |                                  |                              |
  | 2. GET /authorize                |                              |
  |    ?response_type=code           |                              |
  |    &client_id=...                |                              |
  |    &code_challenge=...           |                              |
  |    &code_challenge_method=S256   |                              |
  |--------------------------------->|                              |
  |                                  |                              |
  | 3. User authenticates + consents |                              |
  |                                  |                              |
  | 4. Redirect with auth code       |                              |
  |<---------------------------------|                              |
  |                                  |                              |
  | 5. POST /token                   |                              |
  |    grant_type=authorization_code |                              |
  |    code=...                      |                              |
  |    code_verifier=...             |                              |
  |--------------------------------->|                              |
  |                                  |                              |
  | 6. { access_token, refresh_token }                              |
  |<---------------------------------|                              |
  |                                  |                              |
  | 7. GET /api/resource             |                              |
  |    Authorization: Bearer <token> |                              |
  |-------------------------------------------------------------->  |
  |                                  |                              |
  | 8. 200 OK { ... }               |                              |
  |<--------------------------------------------------------------  |
```

### Key Requirements

- **PKCE is mandatory** — For all client types, including server-side applications
- **Short-lived access tokens** — 15-60 minute expiry
- **Refresh tokens** — For obtaining new access tokens without re-authentication
- **Scope-based authorization** — Tokens carry the minimum scopes required
- **Token rotation** — Issue new refresh tokens on each use, invalidate the old one

---

## Mutual TLS for Service Communication

For service-to-service communication within a cluster, mTLS provides cryptographic identity verification. Unlike standard TLS where only the server proves its identity, mTLS requires both parties to present X.509 certificates.

### When to Use

- Internal microservice-to-microservice calls
- Sensitive data pipelines
- Regulatory compliance requirements (financial, healthcare)
- API gateway to backend service authentication

### Implementation

Typically implemented at the API gateway or service mesh layer (Istio, Linkerd) rather than in application code:

```yaml
# API Gateway configuration (conceptual)
upstream:
  tls:
    client_certificate: /certs/client.pem
    client_key: /certs/client-key.pem
    ca_certificate: /certs/ca.pem
    verify_peer: true
```

---

## Rate Limiting with IETF Headers

Use the standardized **IETF RateLimit** and **RateLimit-Policy** headers instead of legacy `X-RateLimit-*` headers.

### Response Headers

```http
HTTP/1.1 200 OK
RateLimit: limit=100, remaining=95, reset=3500
RateLimit-Policy: 100;w=3600
```

| Header | Description |
|--------|-------------|
| `RateLimit` | Current quota state: `limit`, `remaining`, `reset` (seconds until reset) |
| `RateLimit-Policy` | Quota policy: `<limit>;w=<window_seconds>` |

### Rate Limit Exceeded Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
RateLimit: limit=100, remaining=0, reset=60
RateLimit-Policy: 100;w=3600
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "You have exceeded 100 requests per hour. Retry after 60 seconds."
}
```

### Rate Limiting Strategy

| Tier | Limit | Window | Use Case |
|------|-------|--------|----------|
| Free | 100 req | 1 hour | Trial and evaluation |
| Standard | 1,000 req | 1 hour | Production workloads |
| Enterprise | 10,000 req | 1 hour | High-volume integrations |
| Burst | 50 req | 1 second | Prevent request flooding |

Always include RateLimit headers in every response (not just 429) so clients can proactively track their usage.

### OpenAPI Schema for Rate Limit Headers

```yaml
components:
  headers:
    RateLimit:
      description: Current rate limit state
      schema:
        type: string
        example: "limit=100, remaining=95, reset=3500"
    RateLimitPolicy:
      description: Rate limit policy definition
      schema:
        type: string
        example: "100;w=3600"
    RetryAfter:
      description: Seconds to wait before retrying
      schema:
        type: integer
        example: 60
```

---

## Transport Security

### TLS 1.3 Mandatory

All API traffic must use HTTPS with TLS 1.3 (minimum TLS 1.2). Redirect HTTP requests to HTTPS. Never accept unencrypted traffic.

### Security Headers

Include these headers in all API responses:

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Cache-Control: no-store                # For responses with sensitive data
```

---

## Fine-Grained Authorization

Move beyond simple role-based access control (RBAC) to contextual, policy-driven authorization:

### Scope-Based (OAuth)

```yaml
security:
  - OAuth2:
      - users:read
      - users:write
```

### Attribute-Based (ABAC)

Evaluate authorization based on:
- **Who** — The authenticated identity and their roles
- **What** — The specific resource being accessed
- **How** — The action being performed (read, write, delete)
- **Where** — The request's origin (IP, geolocation)
- **When** — Time-based constraints (business hours only)

### Object-Level Authorization

Verify ownership or access rights for every resource access. Never assume that authentication alone implies authorization to access a specific object. This prevents the OWASP "Broken Object-Level Authorization" vulnerability.

```
GET /users/{id}/orders
  → Verify: Does the authenticated user own user {id}?
  → Or: Does the authenticated user have admin scope?
```

---

## API Key Management

API keys are suitable for identifying applications (not users) and metering usage. They do not replace OAuth for user-delegated access.

### Best Practices

- **Never pass keys in URLs** — Use the `Authorization` header or a custom `X-API-Key` header
- **Use prefixed keys** — `pk_live_abc123` (public), `sk_live_xyz789` (secret) for visual differentiation
- **Support key rotation** — Allow multiple active keys with overlap periods
- **Scope keys** — Assign minimum required permissions per key
- **Log key usage** — Track which key accessed which endpoint for audit trails

```http
GET /products
X-API-Key: pk_live_abc123def456
```

---

## OpenAPI Security Schemes

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT access token obtained via OAuth 2.1 + PKCE

    ApiKey:
      type: apiKey
      in: header
      name: X-API-Key
      description: Application-level API key

    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            users:read: Read user data
            users:write: Create and update users
            users:delete: Delete user accounts
            admin: Full administrative access

# Apply globally
security:
  - BearerAuth: []
```

Override per-operation when different auth is needed:

```yaml
paths:
  /public/health:
    get:
      security: []     # No auth required
  /admin/config:
    put:
      security:
        - OAuth2: [admin]     # Requires admin scope
```

---

## Security Antipatterns

| Antipattern | Risk | Fix |
|------------|------|-----|
| API keys in URL parameters | Keys appear in server logs, browser history | Use `X-API-Key` header |
| Implicit Grant flow | Token exposed in URL fragment | Use Authorization Code + PKCE |
| No object-level authorization | Users access other users' data | Verify ownership on every request |
| Long-lived access tokens | Extended window of compromise | Short-lived tokens + refresh rotation |
| Sensitive data in logs | PII/credential exposure | Scrub auth headers from logs |
| CORS wildcard (`*`) for auth endpoints | Credential theft from any origin | Explicit allow-list of origins |
| No rate limiting | Denial of service, brute force | Implement IETF RateLimit headers |
| Silent 200 OK on auth failure | Hides security events from monitoring | Return 401/403 with Problem Details |
