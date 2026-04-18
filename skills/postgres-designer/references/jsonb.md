# JSONB Patterns Reference

## Table of Contents
- [When to Use JSONB](#when-to-use-jsonb)
- [When NOT to Use JSONB](#when-not-to-use-jsonb)
- [Decision Matrix](#decision-matrix)
- [JSONB Operators](#jsonb-operators)
- [GIN Index Strategies](#gin-index-strategies)
- [Hot-Path Extraction](#hot-path-extraction)
- [Schema Validation](#schema-validation)
- [Query Patterns](#query-patterns)
- [The Hybrid Column Pattern](#the-hybrid-column-pattern)
- [Anti-Patterns](#anti-patterns)

---

## When to Use JSONB

JSONB is appropriate when the data is:

- **Truly schemaless:** External API payloads, webhook bodies, third-party integrations where the schema is not under your control.
- **Sparse attributes:** User preferences, feature flags, metadata — fields that vary per row and are mostly NULL when normalized.
- **Document-shaped:** The entire object is typically read/written as a unit, not joined with other tables.
- **Evolving rapidly:** Attributes change frequently during development, and frequent `ALTER TABLE` migrations would be disruptive.
- **Low-frequency filtering:** Fields that are rarely used in WHERE clauses or JOIN conditions.

---

## When NOT to Use JSONB

JSONB is **the wrong choice** when:

- **The field is used in WHERE clauses frequently.** Promote it to a typed column with a proper B-tree index.
- **The field participates in JOINs.** Relational columns with FK integrity are strictly better.
- **The field requires aggregation.** `SUM`, `AVG`, `GROUP BY` on JSONB paths are slow and type-unsafe.
- **The field has a stable, known schema.** Typed columns enforce correctness and are more storage-efficient (JSONB repeats key names in every row).
- **The field requires referential integrity.** JSONB cannot have foreign keys.

---

## Decision Matrix

| Factor | Normalized Column | JSONB |
|---|---|---|
| Schema stability | Fixed, well-known | Evolving, unknown, sparse |
| Data integrity | FK, CHECK, UNIQUE constraints | Requires manual validation |
| Query performance | B-tree, fast equality/range | GIN, containment queries |
| Storage efficiency | Optimal (no key repetition) | Higher (keys stored per row) |
| Join capability | Full FK support | No FK integrity |
| Migration burden | ALTER TABLE per change | Zero migration for new fields |
| Aggregation | Native SUM/AVG/GROUP BY | JSON path extraction, type casting |

**Rule of thumb:** If you find yourself consistently querying a specific JSONB field with `WHERE (data->>'field')::type = value`, that field should be promoted to a typed column.

---

## JSONB Operators

### Access Operators

| Operator | Returns | Example | Result |
|---|---|---|---|
| `->` | JSONB element | `data -> 'name'` | `"Alice"` (JSONB) |
| `->>` | TEXT element | `data ->> 'name'` | `Alice` (TEXT) |
| `#>` | JSONB at path | `data #> '{address,city}'` | `"Berlin"` (JSONB) |
| `#>>` | TEXT at path | `data #>> '{address,city}'` | `Berlin` (TEXT) |

### Testing Operators

| Operator | Purpose | Example |
|---|---|---|
| `@>` | Contains (left contains right) | `data @> '{"status":"active"}'` |
| `<@` | Contained by | `'{"status":"active"}' <@ data` |
| `?` | Key exists | `data ? 'email'` |
| `?\|` | Any key exists | `data ?\| array['email','phone']` |
| `?&` | All keys exist | `data ?& array['email','phone']` |

### Modification Operators

| Operator | Purpose | Example |
|---|---|---|
| `\|\|` | Concatenate/merge | `data \|\| '{"new_key":"value"}'` |
| `-` | Remove key | `data - 'old_key'` |
| `#-` | Remove at path | `data #- '{nested,key}'` |

### JSONPath (PG12+)

```sql
-- JSONPath query
SELECT * FROM products
WHERE jsonb_path_exists(data, '$.tags[*] ? (@ == "featured")');

-- JSONPath with variables
SELECT jsonb_path_query(data, '$.items[*] ? (@.price > $min)',
    '{"min": 100}'::jsonb)
FROM orders;
```

---

## GIN Index Strategies

### Default GIN (All Operators)

```sql
CREATE INDEX idx_profiles_data ON profiles USING GIN (data);
```

Supports: `@>`, `?`, `?|`, `?&`

**Use when:** Your queries need key existence checking OR containment.

### jsonb_path_ops (Containment Only)

```sql
CREATE INDEX idx_profiles_data_pathops
    ON profiles USING GIN (data jsonb_path_ops);
```

Supports: `@>` only (no `?`, `?|`, `?&`)

**Use when:** Your queries only use containment (`@>`). The index is **3-4x smaller** and faster than the default.

### Choosing Between Them

| Need | Choose |
|---|---|
| Only `@>` queries | `jsonb_path_ops` — smaller, faster |
| `?` key existence queries | Default GIN ops |
| Mixed `@>` and `?` queries | Default GIN ops |
| Very large JSONB docs | `jsonb_path_ops` (size savings matter more) |

### Expression Index on Specific Path

When you filter on a specific scalar field inside JSONB, a B-tree expression index is faster than GIN:

```sql
-- B-tree on extracted field (faster for equality/range on this specific path)
CREATE INDEX idx_orders_data_status ON orders ((data ->> 'status'));

-- Query uses the expression index:
SELECT * FROM orders WHERE data ->> 'status' = 'shipped';
```

---

## Hot-Path Extraction

When a JSONB field becomes frequently queried, promote it to a typed column.

### Generated Columns (Preferred)

```sql
ALTER TABLE profiles
    ADD COLUMN theme TEXT
    GENERATED ALWAYS AS (data ->> 'theme') STORED;

CREATE INDEX idx_profiles_theme ON profiles (theme);

-- Query the typed column instead of JSONB path:
SELECT * FROM profiles WHERE theme = 'dark';
-- Instead of: WHERE data ->> 'theme' = 'dark'
```

**Benefits:**
- Type-safe (TEXT, INTEGER, etc.)
- Standard B-tree index for fast equality/range
- Automatically maintained by PostgreSQL on writes
- No application code changes needed

### Expression Index (Alternative)

If you don't want to add a column:

```sql
CREATE INDEX idx_profiles_theme ON profiles ((data ->> 'theme'));
-- Query: WHERE data ->> 'theme' = 'dark'
```

Less convenient but avoids schema changes.

---

## Schema Validation

### Built-In CHECK Constraints

```sql
-- Ensure JSONB column is always an object
CONSTRAINT chk_config_is_object
    CHECK (jsonb_typeof(config) = 'object')

-- Ensure required keys exist
CONSTRAINT chk_config_required_keys
    CHECK (config ?& array['version', 'theme'])

-- Ensure specific values
CONSTRAINT chk_config_valid_theme
    CHECK (config ->> 'theme' IN ('light', 'dark', 'system'))
```

### pg_jsonschema (Full JSON Schema Validation)

```sql
CREATE EXTENSION IF NOT EXISTS pg_jsonschema;

ALTER TABLE settings
    ADD CONSTRAINT chk_settings_schema
    CHECK (jsonschema_is_valid(config, '{
        "type": "object",
        "properties": {
            "theme": {"type": "string", "enum": ["light", "dark", "system"]},
            "notifications": {"type": "boolean"},
            "language": {"type": "string", "minLength": 2, "maxLength": 5}
        },
        "required": ["theme"],
        "additionalProperties": false
    }'::jsonschema_compiled));
```

**Performance note:** Cast the schema to `jsonschema_compiled` to parse it once at constraint creation, not on every write.

---

## Query Patterns

### Containment Query (GIN-Indexed)

```sql
-- Find users with admin role in their roles array
SELECT * FROM users WHERE profile @> '{"roles": ["admin"]}';

-- Find products in a specific category
SELECT * FROM products WHERE metadata @> '{"category": "electronics"}';
```

### Disjunction (OR-style Containment)

```sql
-- Find orders with either 'active' or 'pending' status
SELECT * FROM orders
WHERE data @> ANY(ARRAY[
    '{"status": "active"}'::jsonb,
    '{"status": "pending"}'::jsonb
]);
```

### Aggregation from JSONB

```sql
-- Extract and aggregate
SELECT
    data ->> 'category' AS category,
    COUNT(*) AS count,
    AVG((data ->> 'price')::numeric) AS avg_price
FROM products
GROUP BY data ->> 'category';
```

### jsonb_to_record (Typed Extraction)

```sql
-- Extract JSONB into typed columns
SELECT id, j.*
FROM orders,
     jsonb_to_record(metadata) AS j(
         tracking_number TEXT,
         weight_kg NUMERIC,
         expedited BOOLEAN
     );
```

### jsonb_agg (Build JSONB in Queries)

```sql
-- Aggregate child rows into JSONB array
SELECT
    p.id,
    p.name,
    jsonb_agg(jsonb_build_object(
        'id', c.id,
        'text', c.text,
        'created_at', c.created_at
    ) ORDER BY c.created_at) AS comments
FROM posts p
LEFT JOIN comments c ON c.post_id = p.id
GROUP BY p.id, p.name;
```

---

## The Hybrid Column Pattern

The most effective approach for most real-world schemas: typed columns for core data, JSONB for flexible metadata.

```sql
CREATE TABLE products (
    id          UUID        NOT NULL DEFAULT uuidv7(),
    name        TEXT        NOT NULL,
    price       NUMERIC     NOT NULL CHECK (price > 0),
    category    TEXT        NOT NULL,
    status      TEXT        NOT NULL DEFAULT 'draft',
    metadata    JSONB       NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Core data: typed columns with full constraint support
    CONSTRAINT pk_products PRIMARY KEY (id),
    CONSTRAINT chk_products_status CHECK (status IN ('draft', 'active', 'archived')),
    CONSTRAINT chk_products_metadata CHECK (jsonb_typeof(metadata) = 'object')
);

-- Typed columns get B-tree indexes
CREATE INDEX idx_products_category ON products (category);
CREATE INDEX idx_products_status ON products (status) WHERE status = 'active';

-- JSONB gets GIN for flexible queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata jsonb_path_ops);
```

This gives you:
- **FK integrity** on core relationships
- **CHECK constraints** on business rules
- **B-tree speed** on hot query paths
- **Schema flexibility** for volatile attributes
- **Zero migrations** for adding new metadata fields

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Using JSONB for relational data | No FK integrity, complex queries | Normalize into typed columns and tables |
| Querying JSONB paths in WHERE without index | Sequential scan on every query | Add GIN or expression index |
| Storing numeric aggregation targets in JSONB | Slow type casting, no optimizer hints | Promote to `NUMERIC` column |
| Using `JSON` instead of `JSONB` | No indexing, reparsed on every access | Always use `JSONB` |
| Deeply nested JSONB (>3 levels) | Unmaintainable queries, poor indexability | Flatten structure or normalize |
| JSONB for data that never changes shape | Wastes storage, loses constraint safety | Use typed columns |
| Updating single JSONB keys with full-doc replace | Rewrites entire JSONB value, high WAL cost | Use `jsonb_set()` for surgical updates |
