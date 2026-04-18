# Migration Conventions Reference

## Table of Contents
- [Migration Philosophy](#migration-philosophy)
- [The Expand-Contract Pattern](#the-expand-contract-pattern)
- [Lock Management](#lock-management)
- [Safe Operations Reference](#safe-operations-reference)
- [Dangerous Operations and Safe Alternatives](#dangerous-operations-and-safe-alternatives)
- [Backfill Strategy](#backfill-strategy)
- [Migration File Template](#migration-file-template)
- [Schema Linting and CI](#schema-linting-and-ci)

> **Related references:** `indexes.md` for `CREATE INDEX CONCURRENTLY`, `constraints.md` for `NOT VALID` constraint patterns, `query-performance.md` for validating plan changes after migration.

---

## Migration Philosophy

Every schema change must maintain **100% availability**. The application is always running during migrations — old pods and new pods coexist. A migration that works in a test database with 100 rows can cause a multi-minute outage on a production table with 50 million rows.

**Core principles:**
1. **Idempotent:** Every statement uses `IF NOT EXISTS` / `IF EXISTS` guards.
2. **Non-locking:** Avoid `ACCESS EXCLUSIVE` locks on large tables. Use `CONCURRENTLY`, `NOT VALID`, and the expand-contract pattern.
3. **Backward-compatible:** Old application code must work with the new schema. New application code must work with the old schema.
4. **One concern per migration:** Don't mix DDL and DML. Don't combine unrelated table changes.
5. **Lock-timeout disciplined:** Set `lock_timeout` before every DDL statement to fail fast instead of creating a lock queue that stalls all traffic.

---

## The Expand-Contract Pattern

The gold standard for breaking schema changes. Decouples the schema change from the code deployment.

### Phase 1: Expand

Add the new column/table. The new column must be nullable or have an immutable default so existing application code (unaware of the change) continues to work.

```sql
-- Migration 0024_add_display_name.sql
SET LOCAL lock_timeout = '5s';

ALTER TABLE users
    ADD COLUMN IF NOT EXISTS display_name TEXT;
-- Nullable — old code ignores it, new code starts writing to it
```

### Phase 2: Migrate (Backfill + Dual-Write)

1. Deploy code that writes to **both** old and new columns.
2. Backfill historical data in batches (see [Backfill Strategy](#backfill-strategy)).
3. Optionally deploy "shadow reads" — read from both and compare, alerting on mismatches.

### Phase 3: Contract

1. Deploy code that reads **only** from the new column.
2. Remove the old column (or add `NOT NULL` to the new one).

```sql
-- Migration 0026_finalize_display_name.sql
SET LOCAL lock_timeout = '5s';

-- Add NOT NULL constraint safely
ALTER TABLE users
    ADD CONSTRAINT chk_users_display_name_not_null
    CHECK (display_name IS NOT NULL) NOT VALID;

ALTER TABLE users VALIDATE CONSTRAINT chk_users_display_name_not_null;
```

### Timing

Each phase is a **separate deployment**. The entire lifecycle can span days or weeks for critical tables. Never rush all phases into a single release.

---

## Lock Management

Lock contention is the #1 cause of migration-related outages. Every DDL statement acquires some level of lock.

### Mandatory `lock_timeout`

```sql
-- Set BEFORE every DDL statement
SET LOCAL lock_timeout = '5s';
```

**Why:** If the DDL cannot acquire its lock within 5 seconds (because a long-running query holds a conflicting lock), it **fails immediately** instead of queuing. Without `lock_timeout`, the queued DDL blocks all subsequent queries behind it, creating a cascading pile-up that looks like a full outage.

**If the migration fails due to timeout:** Retry later. Investigate what long-running query was holding the conflicting lock. Consider killing the blocking query first (`pg_cancel_backend`).

### Lock Levels

| Operation | Lock Level | Blocks |
|---|---|---|
| `SELECT` | `ACCESS SHARE` | Nothing |
| `INSERT/UPDATE/DELETE` | `ROW EXCLUSIVE` | Nothing meaningful |
| `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | Other schema changes, not reads/writes |
| `ADD COLUMN` (nullable or immutable default) | `ACCESS EXCLUSIVE` | **Everything** — but instant (no rewrite) |
| `ADD COLUMN` (volatile default like `now()`) | `ACCESS EXCLUSIVE` | **Everything** — full table rewrite |
| `ALTER COLUMN TYPE` | `ACCESS EXCLUSIVE` | **Everything** — full table rewrite |
| `ADD CONSTRAINT ... NOT VALID` | `ACCESS EXCLUSIVE` | **Everything** — but instant (no scan) |
| `VALIDATE CONSTRAINT` | `SHARE UPDATE EXCLUSIVE` | Other schema changes, not reads/writes |
| `DROP COLUMN` | `ACCESS EXCLUSIVE` | **Everything** — but instant (marks as dropped) |

### Separation of DDL and DML

Never mix schema changes and data updates in the same transaction. Perform one DDL operation per transaction to minimize the time that high-level locks are held.

---

## Safe Operations Reference

### Adding a Column

```sql
SET LOCAL lock_timeout = '5s';

-- Safe: nullable column, instant (no table rewrite)
ALTER TABLE users ADD COLUMN IF NOT EXISTS bio TEXT;

-- Safe: column with IMMUTABLE default, instant (PG11+)
ALTER TABLE users ADD COLUMN IF NOT EXISTS is_active BOOLEAN NOT NULL DEFAULT true;

-- DANGEROUS: column with VOLATILE default (rewrites entire table!)
-- ALTER TABLE users ADD COLUMN IF NOT EXISTS created_at TIMESTAMPTZ DEFAULT now();
-- Instead, add nullable, backfill, then add NOT NULL constraint:
ALTER TABLE users ADD COLUMN IF NOT EXISTS created_at TIMESTAMPTZ;
```

### Removing a Column

```sql
-- Step 1: Stop reading/writing the column in application code (deploy first)
-- Step 2: Drop the column
SET LOCAL lock_timeout = '5s';
ALTER TABLE users DROP COLUMN IF EXISTS legacy_field;
```

Dropping a column is instant (PostgreSQL marks it as dropped; physical removal happens during VACUUM). However, the `ACCESS EXCLUSIVE` lock is still acquired briefly.

### Adding an Index

```sql
-- ALWAYS use CONCURRENTLY on live tables
-- Cannot run inside a transaction block
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_status
    ON orders (status);
```

If a concurrent build fails, it leaves an `INVALID` index. Clean up:

```sql
DROP INDEX IF EXISTS idx_orders_status;
-- Then retry
```

### Adding a Constraint

```sql
SET LOCAL lock_timeout = '5s';

-- Step 1: Add as NOT VALID (instant, no table scan)
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_positive_total
    CHECK (total > 0) NOT VALID;

-- Step 2: Validate (scans table but allows concurrent reads/writes)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_positive_total;
```

### Adding a NOT NULL Constraint

```sql
-- On PG12+, use CHECK + NOT VALID pattern instead of ALTER COLUMN SET NOT NULL
-- (SET NOT NULL acquires ACCESS EXCLUSIVE and scans entire table)

SET LOCAL lock_timeout = '5s';

ALTER TABLE users
    ADD CONSTRAINT chk_users_email_not_null
    CHECK (email IS NOT NULL) NOT VALID;

ALTER TABLE users VALIDATE CONSTRAINT chk_users_email_not_null;
```

### Adding a Foreign Key

```sql
SET LOCAL lock_timeout = '5s';

-- Step 1: Add FK as NOT VALID (instant)
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_users
    FOREIGN KEY (user_id) REFERENCES users (id)
    ON DELETE CASCADE
    NOT VALID;

-- Step 2: Validate (scans table, non-blocking)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_users;

-- Step 3: Add index on FK column (if not already present)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user_id
    ON orders (user_id);
```

---

## Dangerous Operations and Safe Alternatives

| Dangerous Operation | Problem | Safe Alternative |
|---|---|---|
| `ALTER COLUMN TYPE` | Full table rewrite, ACCESS EXCLUSIVE for duration | Shadow-column pattern (see below) |
| `ADD COLUMN ... DEFAULT now()` | Full table rewrite (volatile default) | Add nullable, backfill, add NOT NULL via CHECK |
| `ALTER COLUMN SET NOT NULL` | Full table scan under ACCESS EXCLUSIVE | `ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID` + `VALIDATE` |
| `CREATE INDEX` (without CONCURRENTLY) | Blocks all writes during entire build | `CREATE INDEX CONCURRENTLY` |
| `RENAME COLUMN` | Breaks old code immediately | Expand-contract: add new column, backfill, switch code, drop old |
| `RENAME TABLE` | Breaks all code referencing old name | Create new table, dual-write, switch code, drop old |

### Shadow-Column Type Change Pattern

When you need to change a column's type (e.g., `INTEGER` → `BIGINT`):

```sql
-- Phase 1: Add shadow column
ALTER TABLE events ADD COLUMN IF NOT EXISTS id_new BIGINT;

-- Phase 2: Backfill (in application or batched SQL)
-- Phase 3: Add trigger to keep shadow in sync with original
-- Phase 4: Switch application to read/write from shadow
-- Phase 5: Drop original column, rename shadow

ALTER TABLE events DROP COLUMN id;
ALTER TABLE events RENAME COLUMN id_new TO id;
```

Each phase is a separate migration deployed independently.

---

## Backfill Strategy

When migrating data to a new column, never run a single massive `UPDATE`. It acquires a long-held lock and generates enormous WAL traffic.

### Batched Updates

```sql
-- Process 5,000 rows at a time with a pause for autovacuum
DO $$
DECLARE
    batch_size INT := 5000;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE users
        SET display_name = COALESCE(first_name || ' ' || last_name, email)
        WHERE display_name IS NULL
          AND id IN (
              SELECT id FROM users
              WHERE display_name IS NULL
              ORDER BY id
              LIMIT batch_size
              FOR UPDATE SKIP LOCKED
          );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        COMMIT;                          -- release locks between batches
        PERFORM pg_sleep(0.1);           -- breathing room for autovacuum
    END LOOP;
END $$;
```

### Key Principles

- **Small batches:** 1,000–10,000 rows per batch depending on row width.
- **`FOR UPDATE SKIP LOCKED`:** Prevent blocking concurrent writes.
- **Commit between batches:** Release locks and reduce transaction size.
- **Pause between batches:** Allow autovacuum to process dead tuples.
- **Ordered by PK:** Ensures predictable progress and index-friendly access.

---

## Migration File Template

```sql
-- ============================================================
-- Migration: NNNN_description.sql
-- Description: Brief explanation of what this migration does
-- ============================================================

-- Set lock timeout to prevent lock queue cascades
SET LOCAL lock_timeout = '5s';

-- === DDL CHANGES ===

CREATE TABLE IF NOT EXISTS new_entity (
    id         UUID        NOT NULL DEFAULT uuidv7(),
    name       TEXT        NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT pk_new_entity PRIMARY KEY (id)
);

-- === INDEXES ===

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_new_entity_name
    ON new_entity (name);

-- === CONSTRAINTS (for existing tables, use NOT VALID + VALIDATE pattern) ===

-- ALTER TABLE existing_table
--     ADD CONSTRAINT chk_... CHECK (...) NOT VALID;
-- ALTER TABLE existing_table VALIDATE CONSTRAINT chk_...;
```

### Naming Convention

```
NNNN_description.sql
```

- `NNNN` — zero-padded sequential number (e.g., `0001`, `0024`, `0100`).
- `description` — snake_case, verb-first (e.g., `create_meetings`, `add_display_name_to_users`).

---

## Schema Linting and CI

Run automated linters in CI to catch dangerous migration patterns before they reach production.

### Squawk (Recommended)

Catches unsafe DDL operations:
- Missing `CONCURRENTLY` on index creation
- Adding `NOT NULL` without the `CHECK` pattern
- Adding columns with volatile defaults
- Missing `lock_timeout`
- Table rewrites

### Atlas

Declarative schema management — detects schema drift between the declared state and the actual database.

### CI Pipeline Checklist

1. **Static analysis:** Lint migration SQL for anti-patterns (`squawk`, `atlas lint`)
2. **Migration testing:** Apply migrations to an ephemeral database (Testcontainers or Neon branch)
3. **Schema drift detection:** Compare live schema against version-controlled definitions
4. **Rollback validation:** Verify that every up-migration has a corresponding down-migration
5. **Performance estimation:** For large tables, estimate lock duration and I/O impact

### Testcontainers (Recommended for Integration Tests)

Spin up a real PostgreSQL container matching your production version and extensions (pgvector, PostGIS).

- **Container reuse:** Use the singleton pattern to share one container across test suites. Clean data between tests with transaction rollback or `TRUNCATE`.
- **Port randomization:** Testcontainers maps to random host ports, enabling parallel test execution without conflicts.
- **Same migrations as production:** Seed the test database using the same migration scripts that run in production. This ensures schema parity and catches constraint/trigger issues before merge.

### Ephemeral Database Branching (Neon)

Neon enables instant copy-on-write clones of production databases. Branches reference the same data pages as the parent — creation is instantaneous regardless of dataset size.

- Run migration tests against a branch with real production data shape.
- Verify query plan changes with realistic table statistics.
- Branches are auto-deleted after CI completes — no cleanup overhead.
