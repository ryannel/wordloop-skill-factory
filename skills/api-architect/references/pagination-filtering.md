# Pagination, Filtering & Sorting Patterns

## Table of Contents

- [Why Paginate](#why-paginate)
- [Cursor-Based Pagination](#cursor-based-pagination)
- [Offset-Based Pagination](#offset-based-pagination)
- [Keyset Pagination](#keyset-pagination)
- [Time-Based (Seek) Pagination](#time-based-seek-pagination)
- [Strategy Selection Guide](#strategy-selection-guide)
- [Filtering Patterns](#filtering-patterns)
- [Sorting Patterns](#sorting-patterns)
- [Response Envelope](#response-envelope)
- [Edge Cases](#edge-cases)
- [Best Practices](#best-practices)

---

## Why Paginate

Returning unbounded collections is never acceptable. Without pagination:
- Database queries scan entire tables, degrading performance
- Large payloads consume server memory and network bandwidth
- Client-side rendering freezes on massive datasets
- Network timeouts increase with payload size

Every collection endpoint must return a bounded result set with metadata indicating whether more results exist.

---

## Cursor-Based Pagination

Cursor-based pagination uses an opaque, encoded token pointing to the position of the last item in the current page. The server uses this cursor to fetch the next set of results using an indexed query, achieving constant-time performance regardless of dataset size.

This is the **default strategy** for new APIs. It avoids the performance degradation and data-drift problems inherent in offset-based approaches.

### Request

```http
GET /users?limit=10                                    # First page
GET /users?cursor=eyJpZCI6MzAsInNvcnQiOiJjcmVhdGVkX2F0In0&limit=10  # Next page
```

### Response

```json
{
  "data": [
    { "id": "usr_021", "name": "User 21", "created_at": "2024-01-15T11:00:00Z" },
    { "id": "usr_022", "name": "User 22", "created_at": "2024-01-15T11:30:00Z" }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6MzAsInNvcnQiOiJjcmVhdGVkX2F0In0",
    "has_more": true
  },
  "_links": {
    "next": "/users?cursor=eyJpZCI6MzAsInNvcnQiOiJjcmVhdGVkX2F0In0&limit=10"
  }
}
```

### Cursor Structure (base64 encoded)

The cursor is opaque to the client. Internally it encodes the sort key values needed to construct the `WHERE` clause:

```json
{
  "id": "usr_030",
  "created_at": "2024-01-15T12:00:00Z",
  "sort": ["created_at", "id"]
}
```

### SQL Implementation

```sql
-- First page
SELECT * FROM users
ORDER BY created_at DESC, id DESC
LIMIT 10;

-- Next page (cursor decodes to created_at and id of last item)
SELECT * FROM users
WHERE (created_at, id) < ('2024-01-15T12:00:00Z', 'usr_030')
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

The `(created_at, id)` tuple comparison leverages a composite index and executes in constant time.

### When to Use

- Large datasets (10,000+ records)
- Real-time or frequently changing data
- Infinite scroll UI patterns
- Performance-critical endpoints
- Feed-style interfaces (activity streams, notifications)

---

## Offset-Based Pagination

Offset pagination uses `offset` (skip) and `limit` (page size) parameters. It is simple to understand but degrades at scale.

### Request

```http
GET /users?offset=20&limit=10
```

### Response

```json
{
  "data": [
    { "id": "usr_021", "name": "User 21" },
    { "id": "usr_022", "name": "User 22" }
  ],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 150,
    "has_more": true
  },
  "_links": {
    "first": "/users?offset=0&limit=10",
    "prev":  "/users?offset=10&limit=10",
    "next":  "/users?offset=30&limit=10",
    "last":  "/users?offset=140&limit=10"
  }
}
```

### Page-Based Variant

```http
GET /users?page=3&per_page=10
```

Internally converts to offset: `offset = (page - 1) * per_page`.

### Known Limitations

- **Performance degradation**: The database must scan and discard `offset` rows before returning results. At offset 100,000, this is expensive.
- **Data drift**: If items are inserted or deleted during pagination, clients may see duplicates or skip items between pages.
- **COUNT overhead**: Calculating `total` requires a separate `COUNT(*)` query, which is expensive on large tables.

### When to Use

- Small to medium datasets (< 10,000 records)
- Data that changes infrequently
- Clients need random page access ("jump to page 5")
- Clients need a total count
- Admin dashboards, reports

---

## Keyset Pagination

Keyset pagination uses actual field values (not opaque tokens) to mark the position in the result set. It is a transparent variant of cursor pagination.

### Request

```http
GET /users?after_id=usr_020&limit=10
GET /events?after_created_at=2024-01-15T10:30:00Z&limit=10
```

### Response

```json
{
  "data": [
    { "id": "usr_021", "name": "User 21", "created_at": "2024-01-15T11:00:00Z" }
  ],
  "pagination": {
    "after_id": "usr_030",
    "limit": 10,
    "has_more": true
  },
  "_links": {
    "next": "/users?after_id=usr_030&limit=10"
  }
}
```

### When to Use

- Simple ordering (by ID, timestamp)
- Transparency is preferred over opaque cursors
- Clients already know the sort key values
- Proper composite indexes exist

---

## Time-Based (Seek) Pagination

A specialized form of keyset pagination for time-series data where windows of time are the natural access pattern.

### Request

```http
GET /events?since=2024-01-15T10:00:00Z&until=2024-01-15T11:00:00Z&limit=100
```

### Response

```json
{
  "data": [ ... ],
  "pagination": {
    "since": "2024-01-15T10:00:00Z",
    "until": "2024-01-15T11:00:00Z",
    "limit": 100,
    "has_more": true
  },
  "_links": {
    "next": "/events?since=2024-01-15T11:00:00Z&until=2024-01-15T12:00:00Z&limit=100"
  }
}
```

### When to Use

- Logs and audit trails
- Activity feeds and event streams
- Analytics and metrics data
- Any time-partitioned dataset

---

## Strategy Selection Guide

| Feature | Cursor | Offset/Page | Keyset | Seek |
|---------|--------|-------------|--------|------|
| Performance at scale | Excellent | Poor | Excellent | Excellent |
| Random page access | No | Yes | No | No |
| Total count | No | Yes | Optional | No |
| Data consistency | Excellent | Poor | Excellent | Excellent |
| Implementation complexity | Medium | Simple | Medium | Medium |
| Real-time data | Excellent | Poor | Excellent | Excellent |
| Database load | Low | High | Low | Low |
| Best for | Feeds, streams | Small datasets, admin UIs | Large datasets, simple sorts | Time-series |

**Default recommendation**: Use **cursor-based pagination** for all new collection endpoints. Fall back to offset when random page access is a hard requirement (admin UIs with page numbers).

---

## Filtering Patterns

### Simple Exact Match

```http
GET /users?status=active&role=admin
GET /products?category=electronics
```

### Comparison Operators

```http
GET /users?age_gte=21&age_lt=65
GET /orders?total_gt=100&created_at_gte=2024-01-01
```

### Logical OR (Comma-separated values)

```http
GET /orders?status=shipped,delivered,returned
GET /products?category=books,electronics
```

### Advanced Queries

For APIs that need complex boolean logic, consider:

- **OData syntax**: `GET /products?$filter=price lt 50 and category eq 'home'`
- **FIQL/RSQL**: `GET /users?filter=age=gt=21;status==active`

Document the query language choice explicitly and provide examples for every operator.

### Filter + Pagination Interaction

Filters are applied **before** pagination. The pagination cursor or offset operates on the filtered result set.

```http
GET /orders?status=shipped&cursor=abc123&limit=20
```

This returns the next 20 shipped orders after the cursor position — not the next 20 orders, then filtered.

---

## Sorting Patterns

### Syntax

```http
GET /users?sort=created_at                    # Ascending (default)
GET /users?sort=-created_at                   # Descending
GET /users?sort=-created_at,name              # Multi-field sort
```

### Cursor + Sort Interaction

When using cursor pagination with custom sort orders, the cursor must encode all sort key values. The cursor is sort-specific — a cursor generated with `sort=-created_at` is invalid for `sort=name`.

```json
{
  "cursor": {
    "created_at": "2024-01-15T10:30:00Z",
    "id": "usr_123",
    "sort_fields": ["-created_at", "id"]
  }
}
```

### Allowed Sort Fields

Document which fields support sorting and reject requests for unsortable fields:

```http
GET /users?sort=password_hash

HTTP/1.1 400 Bad Request
{
  "type": "https://api.example.com/errors/invalid-sort-field",
  "title": "Invalid Sort Field",
  "status": 400,
  "detail": "Field 'password_hash' is not sortable. Sortable fields: name, email, created_at, updated_at."
}
```

---

## Response Envelope

All collection endpoints must return a consistent response structure:

```json
{
  "data": [ ... ],
  "pagination": {
    "next_cursor": "eyJ...",
    "has_more": true
  }
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `data` | array | The array of resource objects |
| `pagination.has_more` | boolean | Whether additional pages exist |

### Conditional Fields

| Field | Type | Present When |
|-------|------|-------------|
| `pagination.next_cursor` | string | Cursor pagination, `has_more` is true |
| `pagination.total` | integer | Offset pagination, or when `include_total=true` |
| `_links.next` | string | HATEOAS navigation link |
| `_links.prev` | string | HATEOAS navigation link (if bidirectional) |

---

## Edge Cases

### Empty Results

Return 200 with an empty data array — not 404:

```json
{
  "data": [],
  "pagination": {
    "has_more": false
  }
}
```

### Invalid Cursor

```http
GET /users?cursor=corrupted_value

HTTP/1.1 400 Bad Request
{
  "type": "https://api.example.com/errors/invalid-cursor",
  "title": "Invalid Cursor",
  "status": 400,
  "detail": "The cursor value is malformed or has expired. Start pagination from the beginning by omitting the cursor parameter."
}
```

### Limit Boundaries

Always enforce minimum and maximum limits:

```yaml
parameters:
  Limit:
    name: limit
    in: query
    schema:
      type: integer
      minimum: 1
      maximum: 100
      default: 20
```

If a client requests `limit=1000`, return `400 Bad Request`:

```json
{
  "type": "https://api.example.com/errors/invalid-limit",
  "title": "Invalid Limit",
  "status": 400,
  "detail": "Limit must be between 1 and 100. Default is 20."
}
```

---

## Best Practices

1. **Always paginate collections** — Never return unbounded result sets
2. **Default to cursor pagination** — Use offset only when random access is required
3. **Set reasonable defaults** — Default limit of 20; maximum limit of 100
4. **Include `has_more`** — Always tell the client whether more results exist
5. **Provide navigation links** — Include `_links.next` for easy traversal
6. **Document cursor opacity** — Clients must not parse or modify cursor values
7. **Apply filters before pagination** — The pagination window operates on filtered results
8. **Be consistent** — Use the same pagination pattern across all collection endpoints
9. **Support sorting** — Allow clients to control result order with documented sort fields
10. **Handle edge cases gracefully** — Empty results return 200, invalid cursors return 400
