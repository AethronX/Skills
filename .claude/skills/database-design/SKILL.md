---
name: database-design
description: Designs database schemas, indexes, and migrations. Use when creating new tables/collections, choosing between SQL and NoSQL, adding indexes, planning a migration, designing for multi-tenancy, or reviewing a schema for correctness and performance.
---

# Database Design

## Overview

Schema decisions are the hardest thing to change later — by the time a table has production data and five features reading from it, a "simple" rename is a migration project. Get the shape right before writing the queries that depend on it, and favor changes that are additive and reversible over changes that aren't.

## When to Use

- Designing a new table/collection or a significant addition to one
- Choosing between a relational (SQL) and document/key-value (NoSQL) store
- Adding, removing, or reviewing indexes
- Planning a schema migration
- Designing for multi-tenancy
- Reviewing an existing schema for normalization, indexing, or migration-safety issues

## Choosing SQL vs. NoSQL

```
What does the data look like?
├── Relationships matter (orders belong to users, have many line items)
│   AND you need transactional consistency across them
│   → Relational (Postgres, MySQL). Default choice for most application data.
├── Data is naturally document-shaped, read/written as a whole unit,
│   schema varies per record, and cross-document joins are rare
│   → Document store (MongoDB, Firestore) — but note most of these now
│     also support relational-style joins reasonably well; it's a spectrum.
├── Access pattern is pure key→value lookups at very high throughput
│   → Key-value store (Redis, DynamoDB) for that specific hot path —
│     rarely the *only* store in a real application.
└── Data is inherently time-series or event-stream shaped
    → Purpose-built store (ClickHouse, TimescaleDB) alongside your primary store,
      not instead of it.
```

**Default to relational for application data** (users, orders, content) unless you have a concrete reason not to — it gives you constraints, transactions, and ad hoc querying for free, all of which you will want the first time you debug a data inconsistency.

## Normalization

Normalize by default; denormalize deliberately, for a measured reason.

```sql
-- Normalized: each fact lives in one place
CREATE TABLE users (id UUID PRIMARY KEY, name TEXT, email TEXT UNIQUE);
CREATE TABLE orders (id UUID PRIMARY KEY, user_id UUID REFERENCES users(id), total_cents INT);

-- Denormalized (deliberate): snapshot the user's name onto the order
-- at time of purchase, because "what name did they use when they ordered"
-- is a real business requirement, not a derived value.
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  customer_name_snapshot TEXT, -- intentional duplication, not laziness
  total_cents INT
);
```

Denormalize when you have a specific, named reason (a read-heavy path measured to need it, or a business requirement to preserve a value at a point in time) — not as a default to "save a join." Joins are cheap; inconsistent duplicated data is not.

## Indexing

```sql
-- Index foreign keys used in joins
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Index columns used in WHERE/ORDER BY on large tables
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index: column order matters — most-selective / most-common-filter first
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
-- Serves: WHERE status = ? AND created_at > ?
-- Does NOT efficiently serve: WHERE created_at > ? (alone) — leading column must be used
```

- **Index every foreign key** used in a join; most databases don't do this automatically.
- **Index columns that appear in `WHERE`, `ORDER BY`, or `JOIN ... ON`** on tables large enough to matter.
- **Composite index column order matters**: put the column used in equality filters before the one used in range filters/sorting.
- **Don't over-index.** Every index speeds reads but slows every write to that table and costs storage — add indexes to answer a real, measured query pattern, not preemptively for every column.
- **Verify with `EXPLAIN`/`EXPLAIN ANALYZE`** that a query actually uses the index you added — an index that isn't chosen by the planner is dead weight.

This connects directly to the `performance-optimization` skill's N+1 and unbounded-query guidance — missing indexes are the most common reason a query that's fine at 1,000 rows falls over at 1,000,000.

## Migrations

### Additive-First (mirrors the `api-and-interface-design` skill's "prefer addition over modification")

```sql
-- Step 1 (deploy): add the new column, nullable, no default backfill required
ALTER TABLE users ADD COLUMN display_name TEXT;

-- Step 2 (backfill, can run after deploy, doesn't block it)
UPDATE users SET display_name = name WHERE display_name IS NULL;

-- Step 3 (a LATER deploy, once all code paths write display_name):
ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;

-- Step 4 (a deploy after that, once nothing reads the old column):
ALTER TABLE users DROP COLUMN name;
```

Never rename or drop a column in the same deploy that stops using it — old server instances (during a rolling deploy) or long-lived connections may still expect it to exist. Split into add → dual-write/backfill → cut over reads → remove old, across separate deploys.

### Migration Safety Checklist

- [ ] Every migration has a tested rollback path (or is explicitly additive-only, which needs none)
- [ ] Adding a `NOT NULL` column to an existing table either has a default or is added nullable-then-backfilled-then-constrained, never both in one step on a large table (full-table rewrite/lock risk)
- [ ] Migrations run automatically in CI/CD, never applied by hand against production
- [ ] Long-running migrations on large tables use online/concurrent index creation (e.g. Postgres `CREATE INDEX CONCURRENTLY`) to avoid locking writes
- [ ] Destructive migrations (drop column/table) only run after a deploy that's been live long enough to confirm nothing still reads the old shape

## Common Patterns

### IDs: UUID vs. Auto-Increment

| | Auto-increment integer | UUID |
|---|---|---|
| Guessability | Sequential, guessable (`/orders/1235`) | Not guessable |
| Merge-friendliness | Collides across environments/shards | Globally unique, merges cleanly |
| Index size/locality | Smaller, better locality | Larger, can fragment indexes (mitigated by UUIDv7/ULID) |
| Exposing in URLs | Leaks record count/growth rate | Safe to expose |

Default to UUID (ideally a time-sortable variant like UUIDv7 or ULID) for anything exposed externally or that might need to merge across systems; plain auto-increment is fine for purely internal, single-database tables.

### Timestamps

Every table should default to `created_at` and `updated_at` (auto-set, not application-managed — use `DEFAULT now()` and a trigger, or your ORM's built-in support, so it can't be forgotten in one code path).

### Soft Deletes

```sql
ALTER TABLE tasks ADD COLUMN deleted_at TIMESTAMPTZ;
-- Queries must remember to filter: WHERE deleted_at IS NULL
```

Soft deletes (a `deleted_at` flag instead of removing the row) preserve history and support "undo," but every query against that table must remember to filter deleted rows — easy to forget, and it silently breaks uniqueness constraints (two "deleted" rows with the same unique email). Prefer a partial unique index (`WHERE deleted_at IS NULL`) when using soft deletes with unique columns, and consider whether you actually need history vs. just an audit log.

### Multi-Tenancy

```
Shared table, tenant_id column     → Simplest, cheapest, most common. Every
                                      query MUST filter by tenant_id — enforce
                                      via row-level security, not just app code.
Schema-per-tenant                  → Stronger isolation, harder to manage at
                                      scale (migrations run N times).
Database-per-tenant                → Strongest isolation, highest operational
                                      cost. Reserve for compliance-driven cases.
```

For the shared-table approach, enforce tenant isolation at the database layer (row-level security policies) as a backstop — an app-code bug that forgets a `WHERE tenant_id = ?` filter should not be the only thing standing between one tenant and another's data. This is the same trust-boundary thinking as `security-and-hardening`'s authorization guidance, applied to the data layer.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll add indexes when it's slow" | By then it's a production incident, not a code review comment. Index known access patterns up front. |
| "Let's just drop and recreate the column, it's just a rename" | On a live table with running code, that's a window where writes fail or read the wrong shape. Additive, multi-step migrations exist for this reason. |
| "NoSQL is more scalable" | Scalability is a function of access patterns and operational work, not the category of database. Most apps outgrow bad schema design before they outgrow Postgres. |
| "We don't need tenant_id checks in the DB, the app always filters correctly" | "Always" lasts until one endpoint is added by someone who didn't know the rule. Enforce isolation at the data layer too. |
| "Auto-increment IDs are simpler" | They're simpler until you expose one in a URL and leak your order volume to competitors, or need to merge two databases. |

## Red Flags

- Foreign key columns with no index
- A migration that renames or drops a column in the same deploy that stops using it
- `NOT NULL` added to an existing large table without a default or backfill step
- Money or quantities stored as floating point
- Soft-delete flag with no partial unique index, breaking uniqueness on "deleted" rows
- Multi-tenant table relying only on application code to filter by tenant
- Schema decided by copying whatever the ORM's quickstart generated, unreviewed

## Verification

- [ ] Every foreign key has a supporting index
- [ ] New indexes are verified with `EXPLAIN`/`EXPLAIN ANALYZE` to actually be used
- [ ] Migrations are additive-first; renames/drops are split across multiple deploys
- [ ] Large-table migrations use online/concurrent index creation, not a blocking rewrite
- [ ] Every table has `created_at`/`updated_at` set automatically, not by application code
- [ ] IDs exposed externally are non-sequential (UUID/ULID), not guessable integers
- [ ] Multi-tenant tables enforce isolation at the database layer, not app code alone
- [ ] Rollback path exists (or the migration is provably additive-only)
