# Security & Row-Level Security Reference

## Table of Contents
- [Security Architecture](#security-architecture)
- [Row-Level Security (RLS)](#row-level-security-rls)
- [Multi-Tenant RLS Patterns](#multi-tenant-rls-patterns)
- [Least-Privilege Role Design](#least-privilege-role-design)
- [Column Encryption](#column-encryption)
- [Audit Logging](#audit-logging)
- [Secrets Management](#secrets-management)
- [Security Checklist](#security-checklist)

---

## Security Architecture

Defense in depth: apply security at **every layer** — network, role, table, row, and column.

| Layer | Mechanism | Purpose |
|---|---|---|
| Network | SSL/TLS, firewall rules, VPC | Encrypt in transit, restrict access |
| Role | GRANT/REVOKE, least privilege | Limit what each role can do |
| Table | GRANT SELECT/INSERT/etc. | Table-level access control |
| Row | Row-Level Security (RLS) | Row-level tenant isolation |
| Column | `pgcrypto` encryption | Encrypt sensitive data at rest |

---

## Row-Level Security (RLS)

RLS adds automatic WHERE clauses to every query, enforcing data isolation directly in the database.

### Enabling RLS

```sql
-- Step 1: Enable RLS on the table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Step 2: Force RLS even for table owner (critical for security)
ALTER TABLE documents FORCE ROW LEVEL SECURITY;
```

**Without `FORCE ROW LEVEL SECURITY`**, the table owner bypasses all policies — a major security hole if the application role is the table owner.

### Basic RLS Policy

```sql
-- Users can only see their own documents
CREATE POLICY policy_documents_user_select
    ON documents
    FOR SELECT
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::uuid);

-- Users can only insert documents for themselves
CREATE POLICY policy_documents_user_insert
    ON documents
    FOR INSERT
    TO app_user
    WITH CHECK (user_id = current_setting('app.current_user_id')::uuid);

-- Users can only update their own documents
CREATE POLICY policy_documents_user_update
    ON documents
    FOR UPDATE
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::uuid)
    WITH CHECK (user_id = current_setting('app.current_user_id')::uuid);

-- Users can only delete their own documents
CREATE POLICY policy_documents_user_delete
    ON documents
    FOR DELETE
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::uuid);
```

### Setting Context

The application sets the user/tenant context at the start of each transaction:

```sql
-- In your application's DB middleware:
SET LOCAL app.current_user_id = 'a1b2c3d4-...';
SET LOCAL app.tenant_id = 'workspace-uuid-here';
```

`SET LOCAL` scopes the setting to the current transaction — safe with connection poolers in transaction mode.

---

## Multi-Tenant RLS Patterns

### Shared-Table Multi-Tenancy

All tenants share the same tables, isolated by `tenant_id` and RLS.

```sql
-- All tenant-scoped tables include tenant_id
CREATE TABLE projects (
    id         UUID        NOT NULL DEFAULT uuidv7(),
    tenant_id  UUID        NOT NULL,
    name       TEXT        NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT pk_projects PRIMARY KEY (id),
    CONSTRAINT fk_projects_tenant FOREIGN KEY (tenant_id)
        REFERENCES tenants (id) ON DELETE CASCADE
);

-- Critical: index on tenant_id for both RLS performance and FK lookups
CREATE INDEX idx_projects_tenant ON projects (tenant_id);

-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects FORCE ROW LEVEL SECURITY;

-- Tenant isolation policy
CREATE POLICY policy_projects_tenant
    ON projects
    FOR ALL
    TO app_role
    USING (tenant_id = current_setting('app.tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

### Performance Considerations

- **Index the RLS predicate column** (`tenant_id`). Without it, RLS adds a full-table filter to every query.
- **Keep policies simple.** Complex logic (subqueries, joins) in `USING` clauses adds latency to every query.
- **Verify with `EXPLAIN ANALYZE`** — confirm that the RLS predicate is pushed down into index scans, not applied as a post-filter.

### Role-Based Access Within a Tenant

```sql
-- Admins see everything in their tenant
CREATE POLICY policy_projects_admin
    ON projects
    FOR ALL
    TO app_role
    USING (
        tenant_id = current_setting('app.tenant_id')::uuid
        AND current_setting('app.user_role') = 'admin'
    );

-- Members see only their assigned projects
CREATE POLICY policy_projects_member
    ON projects
    FOR SELECT
    TO app_role
    USING (
        tenant_id = current_setting('app.tenant_id')::uuid
        AND (
            current_setting('app.user_role') = 'admin'
            OR created_by = current_setting('app.current_user_id')::uuid
        )
    );
```

---

## Least-Privilege Role Design

### Role Hierarchy

```sql
-- Superuser: ONLY for emergency maintenance. NEVER for application connections.
-- pg_admin role: DDL operations, migrations

-- Application role: DML only, RLS enforced
CREATE ROLE app_role NOINHERIT NOLOGIN;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;

-- Connection user: logs in, assumes app_role
CREATE USER app_user LOGIN PASSWORD '...' IN ROLE app_role;
```

### Critical Security Rules

| Rule | Why |
|---|---|
| **Never use superuser for application connections** | Bypasses all RLS, can modify system catalogs |
| **Never grant `BYPASSRLS` to application roles** | Defeats the entire purpose of RLS |
| **Use `SECURITY INVOKER` for functions** | Ensures caller's RLS context is applied |
| **Audit `SECURITY DEFINER` functions** | They execute with creator's privileges — can bypass RLS |
| **Never grant `CREATE` on `public` schema** | Prevents privilege escalation via search_path injection |

### Function Security

```sql
-- ✅ SECURITY INVOKER (default): runs with caller's privileges and RLS
CREATE FUNCTION get_my_projects()
RETURNS SETOF projects
LANGUAGE sql
SECURITY INVOKER
AS $$ SELECT * FROM projects; $$;  -- RLS filters applied

-- ⚠️ SECURITY DEFINER: runs with function owner's privileges
-- Use ONLY when you need to bypass RLS for administrative operations
CREATE FUNCTION admin_count_all_projects()
RETURNS BIGINT
LANGUAGE sql
SECURITY DEFINER
SET search_path = public  -- prevent search_path injection
AS $$ SELECT COUNT(*) FROM projects; $$;
```

---

## Column Encryption

### pgcrypto (Symmetric Encryption)

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt sensitive data
INSERT INTO user_secrets (user_id, ssn_encrypted)
VALUES (
    123,
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key'))
);

-- Decrypt at read time
SELECT pgp_sym_decrypt(ssn_encrypted, current_setting('app.encryption_key'))
FROM user_secrets
WHERE user_id = 123;
```

### Key Management Rules

- **Never store encryption keys in migration files or source code.**
- Pass keys via session variables (`SET LOCAL app.encryption_key = '...'`) populated from a secrets manager.
- Rotate keys by re-encrypting with new key, similar to the shadow-column migration pattern.

### When to Encrypt at the Database Level

| Scenario | Encrypt at DB? | Rationale |
|---|---|---|
| SSN, credit card numbers | Yes | Regulatory requirement (PCI, HIPAA) |
| Passwords | No — use `crypt()` | One-way hash, never decrypt |
| API tokens | Consider | May be better handled by a secrets vault |
| General PII | Maybe | Evaluate compliance requirements vs performance cost |

### Password Hashing

```sql
-- Store password hash
UPDATE users SET password_hash = crypt('user_password', gen_salt('bf', 12))
WHERE id = 123;

-- Verify password
SELECT id FROM users
WHERE email = 'user@example.com'
  AND password_hash = crypt('attempted_password', password_hash);
```

---

## Audit Logging

### Trigger-Based Audit (Recommended)

See `table-design.md` → Audit Log Patterns for the full trigger function, table schema, and partitioning strategy.

### pgAudit Extension

Statement-level audit logging for compliance requirements.

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Log all DDL and DML from application roles
ALTER ROLE app_role SET pgaudit.log = 'ddl, write';

-- Log all operations on sensitive tables
ALTER TABLE user_secrets SET (pgaudit.log = 'all');
```

pgAudit writes to the PostgreSQL log. Pair with a log aggregation system (e.g., Cloud Logging, Datadog) for centralized audit trail.

### Audit Trigger + pgAudit Together

- **Trigger-based audit:** Captures row-level changes as structured JSONB. Queryable, searchable.
- **pgAudit:** Captures SQL statement text. For compliance (what SQL was executed, by whom).
- Use both for defense-in-depth in regulated environments.

---

## Secrets Management

### Rules

1. **No credentials in migration files.** Migration files are committed to version control.
2. **No credentials in `postgresql.conf`.** Use environment variables or a secrets manager.
3. **Use `current_setting()` for runtime secrets.** Application sets them via `SET LOCAL` from environment/secrets vault.
4. **Rotate database passwords regularly.** Use IAM database authentication (e.g., GCP Cloud SQL IAM auth) where possible.

### Connection String Security

```
# ✅ Good: environment variable
DATABASE_URL=$DB_URL  # Populated by secrets manager at deploy time

# ❌ Bad: hardcoded
DATABASE_URL=postgres://admin:password123@db.example.com/mydb
```

---

## Verification Steps

Before the schema is complete, confirm each of these:

1. **RLS enabled** on all tenant-scoped tables (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`)
2. **`FORCE ROW LEVEL SECURITY`** set on all RLS tables (prevents owner bypass)
3. **Application role** does NOT have `BYPASSRLS` or `SUPERUSER`
4. **Application role** cannot `CREATE` on `public` schema
5. **All `SECURITY DEFINER` functions** audited and include `SET search_path = public`
6. **`pg_stat_statements`** enabled for query auditing
7. **pgAudit** enabled for DDL/write audit trail in regulated environments
8. **No credentials** in migration files or source code
9. **SSL/TLS enforced** for all database connections
10. **`tenant_id` indexed** on all RLS-filtered tables
11. **RLS policies verified** with `EXPLAIN ANALYZE` — predicate pushdown confirmed, not post-filtering
