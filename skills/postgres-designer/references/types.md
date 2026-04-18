# Column Types Reference

## Table of Contents
- [Type Selection Philosophy](#type-selection-philosophy)
- [Numeric Types](#numeric-types)
- [Text Types](#text-types)
- [Temporal Types](#temporal-types)
- [Boolean](#boolean)
- [Identifiers](#identifiers)
- [Enums vs Lookup Tables](#enums-vs-lookup-tables)
- [Arrays](#arrays)
- [Range Types](#range-types)
- [Network Types](#network-types)
- [Geometric and Spatial Types](#geometric-and-spatial-types)
- [Full-Text Search Types](#full-text-search-types)
- [Domain Types](#domain-types)
- [Composite Types](#composite-types)
- [Binary Data](#binary-data)
- [Vector Types (pgvector)](#vector-types-pgvector)
- [Banned Types](#banned-types)

---

## Type Selection Philosophy

Choose the most specific correct type for every column. The right type:
- Prevents invalid data at the storage layer (no application code needed)
- Optimizes storage footprint (smaller rows → more rows per page → faster scans)
- Enables type-appropriate indexing (B-tree for scalars, GIN for containers, GiST for ranges)

---

## Numeric Types

### Integers

| Type | Size | Range | Use When |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Storage-constrained columns with known small range |
| `INTEGER` | 4 bytes | -2.1B to 2.1B | Moderate-range counters, non-PK integers |
| `BIGINT` | 8 bytes | ±9.2 quintillion | **Default for PKs, FKs, and counters.** Prevents 32-bit overflow |

- **Default to `BIGINT`** unless storage is critical. The 4-byte saving of `INTEGER` is rarely worth the risk of future overflow.
- Sequence/identity columns have gaps — this is expected. Never try to fill them.

### Floating Point

| Type | Size | Precision | Use When |
|---|---|---|---|
| `REAL` | 4 bytes | 6 decimal digits | Storage-constrained approximate values |
| `DOUBLE PRECISION` | 8 bytes | 15 decimal digits | Scientific, statistical computations |
| `NUMERIC(p,s)` | Variable | Exact | **Financial data, monetary values, precision math** |

- **Never use `REAL` or `DOUBLE PRECISION` for money.** Floating-point rounding is catastrophic for financial calculations.
- `NUMERIC` has no rounding errors but is slower for arithmetic than hardware floats.

```sql
-- Money: always NUMERIC with explicit precision
total NUMERIC(12,2) NOT NULL CHECK (total > 0)
```

---

## Text Types

| Type | Use When |
|---|---|
| `TEXT` | **Default for all strings.** Variable length, no artificial limit |
| `TEXT` + `CHECK` | When a length limit is a first-class business rule |
| `BYTEA` | Binary data (encrypted blobs, raw bytes) |

```sql
-- Enforce length as a business rule, not a type constraint
username TEXT NOT NULL CHECK (length(username) BETWEEN 3 AND 30)
```

### TOAST (Transparent Oversized Attribute Storage)

Large values (>2KB default) are automatically compressed and stored out-of-line via TOAST.

| Strategy | Behavior | Use Case |
|---|---|---|
| `EXTENDED` (default) | Compress + out-of-line | Most columns |
| `EXTERNAL` | Out-of-line, no compression | Pre-compressed data |
| `MAIN` | Compress, keep inline if possible | Small values that benefit from compression |
| `PLAIN` | No TOAST | Fixed-width types |

### Case-Insensitive Text

- **Expression index on `LOWER(col)`:** Preferred for queries. Works with standard `TEXT`.
- **`CITEXT` extension:** Case-insensitive type. Use only when you need case-insensitive `UNIQUE` constraints or FK matching.
- **Non-deterministic collations:** For locale/accent-aware case folding.

```sql
-- Preferred: expression index
CREATE UNIQUE INDEX uq_users_email_lower ON users (LOWER(email));
-- Query must match: WHERE LOWER(email) = 'user@example.com'
```

---

## Temporal Types

| Type | Use When |
|---|---|
| `TIMESTAMPTZ` | **All timestamps.** Stores instant in UTC, displays in session timezone |
| `DATE` | Date-only values (birth dates, calendar days) |
| `INTERVAL` | Durations, time differences |
| `TSTZRANGE` | Time intervals for temporal tables, scheduling |

- **Never use `TIMESTAMP` (without time zone).** It is ambiguous in multi-region deployments.
- **Never use `TIMETZ`.** It is nearly useless — timezone without date context is meaningless.
- `now()` returns transaction start time (stable within a transaction). Use `clock_timestamp()` for wall-clock time.

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

---

## Boolean

```sql
is_active BOOLEAN NOT NULL DEFAULT true
```

- Always add `NOT NULL` unless tri-state semantics (true/false/unknown) are explicitly required.
- Optimized 1-byte storage.

---

## Identifiers

See `table-design.md` for the full PK decision matrix. Summary:

| Type | Size | Use When |
|---|---|---|
| `BIGINT GENERATED ALWAYS AS IDENTITY` | 8 bytes | Internal-only PKs, single-node |
| `UUID` via `uuidv7()` | 16 bytes | Distributed, external APIs, offline clients |
| `UUID` via `gen_random_uuid()` | 16 bytes | PG17 and earlier, moderate insert rates |

---

## Enums vs Lookup Tables

### PostgreSQL ENUM

```sql
CREATE TYPE day_of_week AS ENUM ('mon', 'tue', 'wed', 'thu', 'fri', 'sat', 'sun');
```

- Use for **small, stable, domain-fixed** sets (days of week, compass directions).
- Compact storage (4 bytes), type-safe.
- **Cannot remove values** — only append new ones. This makes enums dangerous for business-logic-driven status fields.

### TEXT + CHECK (Preferred for Business Logic)

```sql
status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'active', 'suspended', 'deleted'))
```

- Easy to add/remove values via migration.
- No type migration needed — just update the `CHECK` constraint.
- Slightly more storage than ENUM but far more migration-friendly.

### Lookup Table

```sql
CREATE TABLE order_statuses (
    code TEXT NOT NULL,
    label TEXT NOT NULL,
    sort_order INTEGER,
    CONSTRAINT pk_order_statuses PRIMARY KEY (code)
);
```

- Use when the set of values has metadata (labels, ordering, permissions) or needs to be user-manageable.
- FK integrity: `status TEXT REFERENCES order_statuses(code)`.

---

## Arrays

```sql
tags TEXT[] NOT NULL DEFAULT '{}'
```

- Use for ordered lists of values where you query elements by containment (`@>`, `<@`) or overlap (`&&`).
- Index with **GIN** for fast containment queries.
- Access: `arr[1]` (1-indexed), `arr[1:3]` (slicing).
- Literal syntax: `'{val1,val2}'` or `ARRAY['val1','val2']`.

```sql
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);
-- Find articles with both 'postgres' and 'indexing' tags
SELECT * FROM articles WHERE tags @> ARRAY['postgres', 'indexing'];
```

**Do not use arrays for relationships.** Use junction tables instead. Arrays lack FK integrity and become unwieldy for large sets.

---

## Range Types

| Type | Underlying | Use Case |
|---|---|---|
| `INT4RANGE` / `INT8RANGE` | Integer | Version ranges, numeric intervals |
| `NUMRANGE` | Numeric | Price ranges, measurement intervals |
| `DATERANGE` | Date | Calendar periods, availability windows |
| `TSTZRANGE` | TimestampTZ | Scheduling, temporal versioning |

```sql
CREATE TABLE room_bookings (
    id            BIGINT GENERATED ALWAYS AS IDENTITY,
    room_id       BIGINT      NOT NULL,
    booking_period TSTZRANGE  NOT NULL,

    CONSTRAINT pk_room_bookings PRIMARY KEY (id),
    -- Prevent double-booking: no overlapping intervals for the same room
    CONSTRAINT excl_bookings_no_overlap
        EXCLUDE USING GIST (room_id WITH =, booking_period WITH &&)
);
```

- Support operators: overlap (`&&`), containment (`@>`), adjacent (`-|-`).
- Index with **GiST** for range operations.
- Pick a bounds convention and use it consistently. Prefer `[)` (inclusive start, exclusive end).

---

## Network Types

| Type | Use Case |
|---|---|
| `INET` | IP addresses (v4 and v6) |
| `CIDR` | Network ranges |
| `MACADDR` | MAC addresses |

Support containment operators: `<<` (contained), `>>` (contains), `&&` (overlap).

---

## Geometric and Spatial Types

Built-in: `POINT`, `LINE`, `POLYGON`, `CIRCLE`, `BOX`, `PATH`. Index with **GiST**.

For anything beyond basic 2D geometry, use **PostGIS** — it provides thousands of spatial functions, proper geodetic calculations, and spatial reference system support.

---

## Full-Text Search Types

| Type | Purpose |
|---|---|
| `TSVECTOR` | Pre-computed search document |
| `TSQUERY` | Search query |

```sql
-- Generated column for full-text search
ALTER TABLE articles
    ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(body, '')), 'B')
    ) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Query
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres & indexing');
```

- **Always specify language:** `to_tsvector('english', col)` and `to_tsquery('english', 'query')`. Never use single-argument versions — they skip stemming and stop-word removal.
- For fuzzy/trigram matching, use the `pg_trgm` extension with GIN index.

---

## Domain Types

Reusable custom types with built-in validation.

```sql
CREATE DOMAIN email AS TEXT
    CHECK (VALUE ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$');

CREATE DOMAIN positive_amount AS NUMERIC
    CHECK (VALUE > 0);

-- Usage
CREATE TABLE invoices (
    id     BIGINT GENERATED ALWAYS AS IDENTITY,
    to_email  email           NOT NULL,
    amount    positive_amount NOT NULL,

    CONSTRAINT pk_invoices PRIMARY KEY (id)
);
```

Domains enforce the same constraint everywhere they are used, reducing duplication.

---

## Composite Types

Structured data within a single column.

```sql
CREATE TYPE address AS (
    street TEXT,
    city   TEXT,
    state  TEXT,
    zip    TEXT
);

-- Access with (col).field syntax
SELECT (shipping_address).city FROM orders;
```

Use sparingly. Prefer separate columns or JSONB unless the composite is truly a single conceptual unit that is always read/written together.

---

## Binary Data

| Type | Use Case |
|---|---|
| `BYTEA` | Encrypted blobs, small binary data, hashes |

For large binary objects (images, videos), store a reference (URL/path) in the database and the blob in object storage. PostgreSQL is not a file server.

---

## Vector Types (pgvector)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id        BIGINT  GENERATED ALWAYS AS IDENTITY,
    content   TEXT    NOT NULL,
    embedding VECTOR(1536) NOT NULL,  -- OpenAI ada-002 dimensionality

    CONSTRAINT pk_embeddings PRIMARY KEY (id)
);
```

- `VECTOR(n)` — fixed-dimension float32 vector. Max 2,000 dimensions for HNSW indexing.
- `HALFVEC(n)` — float16 vector. Max 4,000 dimensions, 50% storage savings. Use for high-dimensional models (e.g., 3,072d).
- Distance operators: `<->` (L2), `<=>` (cosine), `<#>` (inner product).

See `indexes.md` for HNSW vs IVFFlat index configuration.

---

## Banned Types

These types must not be used. The table explains why and provides the correct alternative.

| Banned Type | Problem | Use Instead |
|---|---|---|
| `TIMESTAMP` (without TZ) | Ambiguous in multi-region | `TIMESTAMPTZ` |
| `CHAR(n)` | Pad-filled, wastes storage | `TEXT` |
| `VARCHAR(n)` | Artificial limit, no performance benefit | `TEXT` + `CHECK` if needed |
| `MONEY` | Locale-dependent, rounding issues | `NUMERIC(p,s)` |
| `TIMETZ` | Timezone without date is meaningless | `TIMESTAMPTZ` |
| `SERIAL` / `BIGSERIAL` | Permission/ownership quirks | `BIGINT GENERATED ALWAYS AS IDENTITY` |
| `TIMESTAMPTZ(0)` | Precision truncation causes subtle bugs | `TIMESTAMPTZ` (full precision) |
| `JSON` | Not binary-optimized, no indexing | `JSONB` (unless key order must be preserved) |
