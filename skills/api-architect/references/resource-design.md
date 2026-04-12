# Resource Design & URI Architecture

## Table of Contents

- [Resource-Oriented Architecture](#resource-oriented-architecture)
- [URI Naming Conventions](#uri-naming-conventions)
- [Resource Relationships and Nesting](#resource-relationships-and-nesting)
- [Query Parameter Design](#query-parameter-design)
- [Content Negotiation](#content-negotiation)
- [HATEOAS and Hypermedia](#hateoas-and-hypermedia)
- [Antipatterns](#antipatterns)

---

## Resource-Oriented Architecture

REST APIs are built around resources, not actions. A resource is any concept that can be identified, named, and addressed — a user, an order, a product, a configuration. The URI identifies the resource; the HTTP method expresses the action.

This separation is foundational because it enables:
- **Cacheability** — GET requests to the same URI can be cached transparently
- **Discoverability** — Developers can predict URI structure from naming patterns
- **Tooling** — OpenAPI generators, mock servers, and SDK builders rely on resource semantics

### Good Resource URIs

```
GET    /users                  # Collection of users
GET    /users/{id}             # Individual user
GET    /users/{id}/orders      # Orders belonging to a specific user
POST   /users                  # Create a new user
PUT    /users/{id}             # Replace a user entirely
PATCH  /users/{id}             # Partially update a user
DELETE /users/{id}             # Remove a user
```

### Bad Resource URIs

```
POST   /getUser                # Verb in URI — use GET /users/{id}
POST   /createUser             # Verb in URI — use POST /users
GET    /user?action=delete     # Action as query param — use DELETE /users/{id}
POST   /deleteAllInactiveUsers # Action + adjective — model as a resource or async job
```

---

## URI Naming Conventions

### Pluralization

Use **plural nouns** for all collection URIs. This removes ambiguity: `/users` is the collection, `/users/{id}` is a member of that collection.

```
/users           ✓
/user            ✗  (singular is ambiguous — is it a single user or the user resource?)
/orders          ✓
/order           ✗
```

### Casing

Use **kebab-case** (lowercase with hyphens) for multi-word resource names. This is the URL-native format — readable in browser address bars, consistent with standard URL encoding, and unambiguous across case-sensitive systems.

```
/user-profiles          ✓  kebab-case
/shipping-addresses     ✓  kebab-case
/userProfiles           ✗  camelCase — blends into URLs poorly
/user_profiles          ✗  snake_case — underscores are less idiomatic in URLs
```

### Property Naming

While URIs use kebab-case, JSON property names within request and response bodies should use a **single consistent convention** chosen at project inception:

- **snake_case** — Dominant in Ruby, Python, Go ecosystems (`created_at`, `first_name`)
- **camelCase** — Dominant in JavaScript/TypeScript ecosystems (`createdAt`, `firstName`)

Pick one. Apply it everywhere. Document the choice in the API style guide and enforce it via linting.

### Trailing Slashes

Choose one convention and enforce it. The industry norm is **no trailing slash**.

```
/users           ✓  preferred
/users/          ✗  redirect to /users or reject
```

Configure the web framework or API gateway to redirect or reject the opposite convention to avoid duplicate caching entries.

---

## Resource Relationships and Nesting

### The Two-Level Rule

Nest sub-resources to express direct ownership, but never go deeper than two levels. Deep nesting creates rigid URI structures that break when business relationships change.

```
/users/{userId}/orders                    ✓  One level of nesting
/users/{userId}/orders/{orderId}          ✓  Two levels (acceptable boundary)
/users/{userId}/orders/{orderId}/items    ✗  Three levels — flatten this
```

### Flatten Beyond the Relationship

After establishing the ownership relationship, provide independent access to child resources through their own top-level collection:

```
GET /users/{userId}/orders      # "Show me this user's orders"
GET /orders/{orderId}           # "Show me this specific order, regardless of user"
GET /orders/{orderId}/items     # Items for a specific order (one level from /orders)
```

This "hybrid flat" approach gives clients the flexibility to traverse the graph from any starting point. It also simplifies caching, since `/orders/{id}` can be cached independently of `/users/{userId}/orders`.

### Complementing with HATEOAS

Include hypermedia links in responses to connect related resources. The client does not need to construct URIs from templates — it follows links returned by the server.

```json
{
  "id": "ord_abc123",
  "status": "shipped",
  "total": 149.99,
  "_links": {
    "self":     { "href": "/orders/ord_abc123" },
    "customer": { "href": "/users/usr_xyz789" },
    "items":    { "href": "/orders/ord_abc123/items" },
    "invoice":  { "href": "/orders/ord_abc123/invoice" }
  }
}
```

### Functional Actions (Non-CRUD State Changes)

Some operations don't map cleanly to CRUD — activating a user, archiving an order, retrying a payment. Model these as sub-resource actions:

```
POST /users/{id}/activate           # Activate a user account
POST /orders/{id}/cancel            # Cancel an order
POST /payments/{id}/retry           # Retry a failed payment
```

Use POST for actions with side effects. The action URI is a noun-phrase (the activation, the cancellation), even though it reads like a verb.

---

## Query Parameter Design

### Filtering

Use field-level query parameters for simple filtering, and comparison suffixes for range queries:

```
GET /products?category=books                       # Exact match
GET /products?category=books,electronics           # Logical OR (comma-separated)
GET /users?age_gte=21&status=active                # Greater-than-or-equal
GET /orders?created_at_lt=2024-06-01T00:00:00Z     # Less-than
```

**Standard suffixes:**

| Suffix | Meaning |
|--------|---------|
| `_eq` | Equal (optional, exact match is default) |
| `_neq` | Not equal |
| `_lt` | Less than |
| `_lte` | Less than or equal |
| `_gt` | Greater than |
| `_gte` | Greater than or equal |
| `_in` | In a set (comma-separated values) |

For complex queries beyond simple filtering (joins, nested conditions), consider adopting a standard query language like OData or FIQL rather than inventing custom syntax.

### Sorting

Use a `sort` parameter with comma-separated field names. Prefix with `-` for descending order:

```
GET /users?sort=created_at              # Ascending (default)
GET /users?sort=-created_at             # Descending
GET /users?sort=-created_at,name        # Primary: newest first, secondary: name A-Z
```

### Field Selection (Sparse Fieldsets)

Allow clients to request only the fields they need to reduce payload size:

```
GET /users?fields=id,name,email         # Only return these fields
```

This is especially valuable for mobile clients on constrained networks and for AI agents that need to minimize token consumption.

### Search

Provide a generic `q` parameter for free-text search across multiple fields:

```
GET /users?q=john                       # Search name, email, etc.
GET /products?search=wireless+keyboard  # Alternative naming
```

---

## Content Negotiation

### Accept Header

Clients declare their preferred response format via the `Accept` header:

```http
GET /users/123
Accept: application/json

GET /users/123
Accept: application/hal+json
```

### Response Content-Type

The server declares the format of its response:

```http
Content-Type: application/json; charset=utf-8
Content-Type: application/problem+json            # For error responses
Content-Type: application/hal+json                 # For HATEOAS responses
```

---

## HATEOAS and Hypermedia

Hypermedia-driven APIs include links to related resources and available actions, making the API self-documenting and enabling clients to navigate without hardcoding URI templates.

### HAL (Hypertext Application Language)

```json
{
  "id": "usr_123",
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self":    { "href": "/users/usr_123" },
    "orders":  { "href": "/users/usr_123/orders" },
    "update":  { "href": "/users/usr_123", "method": "PATCH" },
    "delete":  { "href": "/users/usr_123", "method": "DELETE" }
  },
  "_embedded": {
    "recent_orders": [
      {
        "id": "ord_456",
        "total": 99.99,
        "_links": {
          "self": { "href": "/orders/ord_456" }
        }
      }
    ]
  }
}
```

### When to Use HATEOAS

HATEOAS is most valuable when:
- The API serves multiple client types with different navigation paths
- The URI structure may evolve — clients following links are insulated from changes
- Discoverability is a priority (public APIs, developer portals)

For internal microservice APIs with tight coupling and code-generated clients, HATEOAS adds overhead without proportional benefit. Use it selectively.

---

## Antipatterns

| Antipattern | Problem | Fix |
|------------|---------|-----|
| Verb-oriented URIs (`/createOrder`) | Ignores HTTP method semantics | Use `POST /orders` |
| Inconsistent naming (`/userProfiles` + `/order_items`) | Increases cognitive load | Pick one convention, enforce it |
| Deep nesting (`/a/{id}/b/{id}/c/{id}/d`) | Rigid, hard to maintain | Flatten beyond two levels |
| Exposing implementation (`/api/v1/mysql/users/table`) | Leaks internal architecture | Abstract behind resource names |
| Missing plural collections (`/user/{id}`) | Ambiguous semantics | Always use `/users/{id}` |
| Action verbs in query params (`?action=delete`) | Bypasses HTTP method routing | Use `DELETE /resource/{id}` |
