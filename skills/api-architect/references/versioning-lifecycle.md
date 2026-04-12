# Versioning & Lifecycle Management

## Table of Contents

- [Versioning Strategies](#versioning-strategies)
- [Date-Based Versioning](#date-based-versioning)
- [URI Path Versioning](#uri-path-versioning)
- [Header Versioning](#header-versioning)
- [Breaking vs Non-Breaking Changes](#breaking-vs-non-breaking-changes)
- [Deprecation and Sunset Headers](#deprecation-and-sunset-headers)
- [Lifecycle Phases](#lifecycle-phases)
- [Migration Strategy](#migration-strategy)
- [Version Discovery](#version-discovery)
- [OpenAPI Versioning](#openapi-versioning)
- [Antipatterns](#antipatterns)

---

## Versioning Strategies

Three mainstream approaches exist. The choice depends on the API's audience, scale, and evolution velocity.

| Strategy | Mechanism | Best For |
|----------|-----------|----------|
| Date-based | Header: `API-Version: 2024-06-15` | High-velocity APIs with frequent releases |
| URI path | `/v1/users`, `/v2/users` | Public APIs prioritizing simplicity |
| Header | `Accept: application/vnd.myapi.v2+json` | APIs prioritizing URI stability |

---

## Date-Based Versioning

Date-based versioning pins an integration to a specific API release date (e.g., `2024-06-15`). The server maintains a "downgrade" module that transforms its current response into the structure expected by older versions.

### How It Works

```http
GET /users/123
API-Version: 2024-06-15

HTTP/1.1 200 OK
API-Version: 2024-06-15
```

The server internally runs the latest code but applies version-specific transformations before returning the response. This allows the platform to evolve continuously while guaranteeing that pinned clients see a stable contract.

### Advantages

- **Continuous evolution** — New features ship without creating new major versions
- **No breaking changes for pinned clients** — Each date version is frozen
- **Clear upgrade path** — Clients update their version header when ready
- **Granular** — Each release is independently addressable

### When to Use

- APIs with frequent, incremental changes
- Platform APIs serving many third-party integrations
- Organizations practicing continuous deployment

---

## URI Path Versioning

The most common and visible approach. The version is part of the URL path.

```http
GET /v1/users/123
GET /v2/users/123
```

### Advantages

- Explicit and discoverable — visible in URL
- Simple routing and caching
- Easy to understand and debug
- Can run multiple versions simultaneously

### Disadvantages

- Same resource at different URIs (technically violates REST)
- Requires updating all client code to change version
- Coarse-grained — entire API versioned together

### Version Format

Use simple major versions: `v1`, `v2`, `v3`. Do not use minor versions in public APIs — they create ambiguity about breaking changes.

```
/v1/users       ✓
/v2/users       ✓
/v1.1/users     ✗  Minor versions confuse consumers
/v2.3.1/users   ✗  SemVer in URLs is noise
```

---

## Header Versioning

Version specified via Accept header or custom header. URIs remain stable.

### Accept Header (Content Negotiation)

```http
GET /users/123
Accept: application/vnd.myapi.v2+json
```

### Custom Header

```http
GET /users/123
API-Version: 2
```

### Advantages

- URIs remain stable (more RESTful)
- Separates versioning from resource identification

### Disadvantages

- Less visible — harder to debug without inspecting headers
- Complex caching (Vary by Accept or custom header)
- Difficult to test in browser

---

## Breaking vs Non-Breaking Changes

### Breaking Changes (require new version)

- Removing or renaming fields
- Changing field types (string → integer)
- Adding required fields to request bodies
- Changing response structure or nesting
- Removing endpoints
- Changing HTTP status codes for the same scenario
- Changing authentication mechanisms
- Modifying pagination format

### Non-Breaking Changes (safe without new version)

- Adding new endpoints
- Adding optional request fields
- Adding new fields to response bodies (clients should ignore unknown fields)
- Bug fixes
- Performance improvements
- Adding new HTTP methods to existing resources
- Adding new enum values (if clients handle unknown values gracefully)
- Adding new error codes

---

## Deprecation and Sunset Headers

RFC 8594 defines the `Sunset` header for communicating endpoint decommissioning. The `Deprecation` header (IETF draft) signals that an endpoint is no longer recommended.

### Deprecation Header

Signals that the endpoint still works but is no longer recommended:

```http
HTTP/1.1 200 OK
Deprecation: true
Link: </v2/users/123>; rel="successor-version"
```

### Sunset Header

Announces the scheduled removal date:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Wed, 15 Jan 2025 00:00:00 GMT
Link: </v2/users/123>; rel="successor-version"
```

### After Sunset

Return `410 Gone` with migration guidance:

```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/endpoint-sunset",
  "title": "Endpoint Removed",
  "status": 410,
  "detail": "API v1 was decommissioned. Please migrate to v2.",
  "migration_guide": "https://api.example.com/docs/migration/v1-to-v2"
}
```

---

## Lifecycle Phases

### Phase 1: Introduction

New version released alongside existing version:

```
/v1/users  → Status: Current
/v2/users  → Status: Newly released
```

Announce via changelog, blog post, and developer portal. Provide a migration guide.

### Phase 2: Deprecation

Mark old version as deprecated. Minimum 6-12 months before sunset:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 15 Jun 2025 00:00:00 GMT
```

Continue to fully support the deprecated version during this phase.

Communication channels:
- Response headers (automatic for every request)
- Email to registered developers
- Changelog and developer portal banner
- Dashboard notifications

### Phase 3: Sunset

On the announced date, stop serving the old version:

```http
HTTP/1.1 410 Gone
```

**Recommended timelines:**
- Announce deprecation: 6-12 months before sunset
- Parallel support: 12-24 months minimum
- Grace period: 30 days of 410 responses before complete shutdown

---

## Migration Strategy

### Migration Guide Template

```markdown
# Migrating from v1 to v2

## Breaking Changes

### User Resource
- `name` field split into `first_name` and `last_name`
- `status` enum: added `suspended`, removed `disabled`

### Authentication
- API key auth removed; use OAuth 2.1 + PKCE

## Step-by-Step Migration

1. Update authentication flow to OAuth 2.1
2. Update User model to use `first_name` / `last_name`
3. Handle new `suspended` status in client code
4. Update SDK to v2 (`npm install @myapi/sdk@2`)
5. Change base URL from `/v1` to `/v2`
6. Run integration tests against v2 staging

## SDK Updates

v1: `npm install @myapi/sdk@1`
v2: `npm install @myapi/sdk@2`

## Support

- Migration support: migration@example.com
- FAQ: https://api.example.com/docs/v2/faq
```

### Migration Tools

- Migration scripts for common transformations
- SDK updates with upgrade guides
- API diff viewer (programmatic comparison of v1 vs v2 specs)
- Compatibility layer (temporary proxy that translates v1 requests to v2)

---

## Version Discovery

### Root Endpoint

```http
GET /

HTTP/1.1 200 OK
{
  "versions": {
    "v1": {
      "status": "deprecated",
      "sunset_date": "2025-01-15",
      "documentation": "https://api.example.com/docs/v1"
    },
    "v2": {
      "status": "current",
      "documentation": "https://api.example.com/docs/v2"
    }
  }
}
```

### Changelog

Maintain a public changelog documenting every change:

| Change Type | Communication Method |
|------------|---------------------|
| Non-breaking | Changelog entry |
| New version | Blog post + migration guide |
| Deprecation | Response headers + email + changelog |
| Decommissioning | Sunset header → 410 Gone + migration links |

---

## OpenAPI Versioning

### Separate Specs Per Version

```
openapi/
├── openapi-v1.yaml
├── openapi-v2.yaml
└── openapi-v3.yaml
```

Each spec is complete and independent. This is the simplest approach and works well with code generation tools.

### Single Spec with Server Variables

```yaml
openapi: 3.1.0
info:
  title: My API
  version: 2.0.0
servers:
  - url: https://api.example.com/v1
    description: Version 1 (deprecated)
  - url: https://api.example.com/v2
    description: Version 2 (current)
```

---

## Antipatterns

| Antipattern | Problem | Fix |
|------------|---------|-----|
| Breaking changes without version bump | Breaks existing clients silently | Always version breaking changes |
| Too many active versions (> 3) | Maintenance burden, test matrix explosion | Sunset aggressively with clear timelines |
| Short deprecation periods (< 6 months) | Frustrated developers, broken integrations | Minimum 6-12 month deprecation window |
| No migration path | Developers cannot upgrade | Provide migration guides, scripts, and SDK updates |
| Surprise sunsets | Production breakages | Announce via headers, email, and portal |
| Versioning individual endpoints | Inconsistent API surface | Version the entire API consistently |
| Minor versions in URLs (`/v1.2/`) | Confusion about breaking changes | Use major versions only |
| No changelog | Developers unaware of changes | Maintain public changelog for every release |
