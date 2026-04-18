# Constraints Reference

## Table of Contents
- [Constraint Philosophy](#constraint-philosophy)
- [NOT NULL](#not-null)
- [CHECK Constraints](#check-constraints)
- [UNIQUE Constraints](#unique-constraints)
- [Foreign Key Constraints](#foreign-key-constraints)
- [EXCLUDE Constraints](#exclude-constraints)
- [Deferrable Constraints](#deferrable-constraints)
- [Naming Conventions](#naming-conventions)
- [Column-Level vs Table-Level Placement](#column-level-vs-table-level-placement)

---

## Constraint Philosophy

Constraints are the database's immune system. Application code is fallible — it has bugs, race conditions, and deployment gaps. The schema is the final authority on data integrity.

**Enforce at the DB layer:**
- `NOT NULL` on every column unless `NULL` has explicit semantic meaning
- `CHECK` for domain invariants (positive prices, valid status transitions)
- `UNIQUE` for natural keys and business uniqueness rules
- `FOREIGN KEY` for referential integrity
- `EXCLUDE` for complex overlap/collision prevention

---

## NOT NULL

### Default Stance: NOT NULL by Default

Every column should be `NOT NULL` unless you can articulate why `NULL` is a valid state in the domain.

```sql
-- ✅ Good: explicit about nullability
title      TEXT        NOT NULL,
started_at TIMESTAMPTZ,              -- NULL means "not yet started"
deleted_at TIMESTAMPTZ,              -- NULL means "not deleted" (soft-delete)

-- ❌ Bad: nullable without justification
email TEXT,  -- Can an entity exist without an email? If no, add NOT NULL
```

### NULL Semantics

- `NULL` means "unknown" or "not applicable" — not "empty string" or "zero."
- `NULL` propagates through expressions: `NULL + 1 = NULL`, `NULL = NULL` evaluates to `NULL` (not `TRUE`).
- `NULL` passes `CHECK` constraints (three-valued logic). To enforce a positive-and-required constraint: `price NUMERIC NOT NULL CHECK (price > 0)`.

---

## CHECK Constraints

Row-local invariants that prevent invalid data from entering the table.

### Single-Column Checks

```sql
-- Enforce positive pricing
CONSTRAINT chk_products_positive_price CHECK (price > 0)

-- Enforce status enumeration
CONSTRAINT chk_orders_valid_status
    CHECK (status IN ('pending', 'paid', 'shipped', 'canceled'))

-- Enforce string length as business rule
CONSTRAINT chk_users_username_length
    CHECK (length(username) BETWEEN 3 AND 30)
```

### Multi-Column Checks

```sql
-- End must be after start
CONSTRAINT chk_meetings_timeline
    CHECK (ended_at IS NULL OR ended_at > started_at)

-- Discount cannot exceed price
CONSTRAINT chk_line_items_discount
    CHECK (discount_amount <= unit_price * quantity)
```

### CHECK + NULL Interaction

`CHECK` constraints evaluate using three-valued logic. `NULL` values **pass** the check:

```sql
-- This allows NULL prices!
CHECK (price > 0)  -- NULL passes (NULL > 0 = NULL, which is not FALSE)

-- To enforce non-null AND positive, combine:
price NUMERIC NOT NULL CHECK (price > 0)
```

### JSONB Type Validation

```sql
-- Ensure JSONB column is always an object (not array, string, etc.)
CONSTRAINT chk_config_is_object
    CHECK (jsonb_typeof(config) = 'object')

-- With pg_jsonschema extension: full JSON Schema validation
CONSTRAINT chk_config_schema
    CHECK (jsonschema_is_valid(config, '{
        "type": "object",
        "properties": {
            "theme": {"type": "string", "enum": ["light", "dark"]},
            "notifications": {"type": "boolean"}
        },
        "required": ["theme"]
    }'::jsonschema_compiled))
```

---

## UNIQUE Constraints

Enforce uniqueness. Automatically create a B-tree index.

### Simple Unique

```sql
CONSTRAINT uq_users_email UNIQUE (email)
```

### Composite Unique

```sql
-- Unique combination of workspace and slug
CONSTRAINT uq_projects_workspace_slug UNIQUE (workspace_id, slug)
```

### UNIQUE and NULLs

By default, `UNIQUE` allows **multiple NULLs** (each NULL is considered distinct).

```sql
-- PG15+: treat NULLs as equal (only one NULL allowed)
CONSTRAINT uq_users_phone UNIQUE NULLS NOT DISTINCT (phone)

-- Pre-PG15: use a partial unique index
CREATE UNIQUE INDEX uq_users_phone ON users (phone) WHERE phone IS NOT NULL;
```

### Expression-Based Unique

```sql
-- Case-insensitive email uniqueness
CREATE UNIQUE INDEX uq_users_email_lower ON users (LOWER(email));
```

### Partial Unique (Soft-Delete Safe)

```sql
-- Unique slug only among non-deleted records
CREATE UNIQUE INDEX uq_articles_slug_active
    ON articles (slug)
    WHERE deleted_at IS NULL;
```

---

## Foreign Key Constraints

Referential integrity between parent and child tables.

```sql
CONSTRAINT fk_orders_users FOREIGN KEY (user_id)
    REFERENCES users (id) ON DELETE CASCADE
```

### ON DELETE / ON UPDATE Actions

| Action | Behavior | Use When |
|---|---|---|
| `CASCADE` | Delete/update children automatically | Child is meaningless without parent |
| `RESTRICT` | Block parent modification immediately | Defensive default — prevent accidental data loss |
| `SET NULL` | Set FK column to NULL | Child exists independently (e.g., `assigned_to`) |
| `SET DEFAULT` | Reset FK to its DEFAULT value | Rarely used |
| `NO ACTION` | Like RESTRICT but deferred to end of statement | Useful with deferrable constraints for circular FKs |

**Always be explicit.** Never rely on the implicit default (`NO ACTION`).

### FK Index Requirement

**PostgreSQL does NOT auto-index FK columns.** Without an index on the FK column:
- Joins on the FK column scan the entire child table
- Deleting a parent row acquires a `SHARE` lock on the child table (blocks concurrent writes)

```sql
-- Always pair FK with an index
CONSTRAINT fk_orders_users FOREIGN KEY (user_id)
    REFERENCES users (id) ON DELETE CASCADE
-- ...
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

### Multi-Column FKs

```sql
CONSTRAINT fk_line_items_order_product
    FOREIGN KEY (order_id, product_id)
    REFERENCES order_products (order_id, product_id)
```

The referenced columns must have a matching `UNIQUE` or `PRIMARY KEY` constraint.

---

## EXCLUDE Constraints

Prevent rows from violating complex overlap/collision rules. More powerful than `UNIQUE` — supports arbitrary operators.

### Range Overlap Prevention

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- needed for = operator with GiST

CREATE TABLE room_bookings (
    id             BIGINT GENERATED ALWAYS AS IDENTITY,
    room_id        BIGINT     NOT NULL,
    booking_period TSTZRANGE  NOT NULL,

    CONSTRAINT pk_room_bookings PRIMARY KEY (id),
    -- No two bookings for the same room can overlap in time
    CONSTRAINT excl_bookings_no_overlap
        EXCLUDE USING GIST (room_id WITH =, booking_period WITH &&)
);
```

### How It Works

`EXCLUDE USING GIST (col1 WITH op1, col2 WITH op2)` prevents any two rows where **all** operator conditions are true simultaneously. In the example above: no two rows can have the **same** `room_id` AND **overlapping** `booking_period`.

### Use Cases

- **Room/resource scheduling:** Prevent double-booking
- **Shift management:** No overlapping shifts for the same employee
- **Version ranges:** No overlapping version intervals for the same entity
- **Temporal tables:** Prevent overlapping validity periods (see `table-design.md`)

---

## Deferrable Constraints

Constraints that are checked at the **end of the transaction** rather than after each statement.

```sql
-- Useful for circular FK dependencies
CONSTRAINT fk_employees_manager FOREIGN KEY (manager_id)
    REFERENCES employees (id) ON DELETE SET NULL
    DEFERRABLE INITIALLY DEFERRED
```

### When to Use

- **Circular FKs:** Employee → Manager → Employee. Both inserts must happen in the same transaction.
- **Bulk data loading:** Temporarily defer FK checking during large batch inserts.
- **Complex multi-statement operations:** When intermediate states are temporarily invalid but the final state is consistent.

### How It Works

```sql
BEGIN;
-- With DEFERRABLE INITIALLY DEFERRED, these two inserts succeed
-- even though they reference each other:
INSERT INTO employees (id, name, manager_id) VALUES (1, 'Alice', 2);
INSERT INTO employees (id, name, manager_id) VALUES (2, 'Bob', 1);
COMMIT;  -- FKs checked here — both references are valid
```

### Caveat

- Deferred constraints make it harder to pinpoint which statement caused a violation.
- Use sparingly — prefer `INITIALLY IMMEDIATE` (the default) unless you specifically need cross-statement consistency.

---

## Naming Conventions

All constraints must be explicitly named. Anonymous constraints are a migration and diffing liability.

See `table-design.md` → Naming Conventions for the full convention table. Summary:

| Prefix | Type | Example |
|---|---|---|
| `pk_` | Primary key | `pk_users` |
| `fk_` | Foreign key | `fk_orders_users` |
| `uq_` | Unique | `uq_users_email` |
| `chk_` | Check | `chk_orders_positive_total` |
| `excl_` | Exclusion | `excl_bookings_no_overlap` |

---

## Column-Level vs Table-Level Placement

### Column-Level (Inline)

Use for single-column constraints that are directly associated with their column:

```sql
CREATE TABLE users (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT   NOT NULL UNIQUE
);
```

- Concise and readable for simple tables.
- **Cannot name the constraint** in standard column-level syntax (some tools allow `CONSTRAINT name` inline, but it's less portable).

### Table-Level (End-of-Table)

**Preferred for production schemas.** All constraints are named, visible, and easy to diff:

```sql
CREATE TABLE users (
    id    BIGINT GENERATED ALWAYS AS IDENTITY,
    email TEXT   NOT NULL,

    CONSTRAINT pk_users      PRIMARY KEY (id),
    CONSTRAINT uq_users_email UNIQUE (email)
);
```

- **Required** for multi-column constraints (`UNIQUE (a, b)`, composite FKs, `CHECK` across columns).
- Required for `EXCLUDE` constraints.
- Makes migration diffs cleaner — adding/removing a constraint is a single line change.

**Recommendation:** Use table-level placement with explicit names for all constraints in production schemas. Column-level is acceptable only for throwaway/prototype code.
