# Normalization vs Denormalization

A practical guide to choosing between normalized and denormalized data models, with real-world patterns, decision frameworks, and production examples.

---

## Table of Contents

1. [What Normalization Actually Solves](#1-what-normalization-actually-solves)
2. [Normal Forms — Quick Refresher](#2-normal-forms--quick-refresher)
3. [The Cost of Normalization](#3-the-cost-of-normalization)
4. [What Denormalization Actually Solves](#4-what-denormalization-actually-solves)
5. [Denormalization Patterns](#5-denormalization-patterns)
6. [Decision Framework](#6-decision-framework)
7. [Real-World Case Study: File Access Checks](#7-real-world-case-study-file-access-checks)
8. [Consistency Strategies for Denormalized Data](#8-consistency-strategies-for-denormalized-data)
9. [When Companies Denormalize](#9-when-companies-denormalize)
10. [Anti-Patterns and Mistakes](#10-anti-patterns-and-mistakes)
11. [Hybrid Approaches](#11-hybrid-approaches)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What Normalization Actually Solves

Normalization eliminates **update anomalies** — situations where the same fact is stored in multiple places and can become inconsistent.

### The Three Anomalies

**Insert anomaly**: You can't store a fact without storing an unrelated fact.
```
-- Bad: project name stored on every file
| file_id | file_name | project_id | project_name |
|---------|-----------|------------|--------------|
| 1       | doc.pdf   | 10         | Legal Review |

-- Can't create project "Tax Audit" without also creating a file.
```

**Update anomaly**: Changing one fact requires changing it in multiple rows.
```
-- Renaming "Legal Review" to "Legal Analysis" requires updating every file row.
-- Miss one row → inconsistency.
```

**Delete anomaly**: Deleting all files in a project also deletes the project name.
```
-- Delete the last file → project name is gone from the database.
```

Normalization solves all three by storing each fact exactly once:
```
projects:       | id | name         |
uploaded_files: | id | project_id   |    ← FK, no name duplication
```

### The Core Principle

**Every non-key column must depend on the key, the whole key, and nothing but the key.**

This is the informal statement of 3NF/BCNF. If a column depends on something other than the primary key, it belongs in a different table.

---

## 2. Normal Forms — Quick Refresher

### 1NF — Atomic Values

Every column holds a single value, no repeating groups.

```sql
-- Violates 1NF: tags is a comma-separated string
| id | name    | tags              |
|----|---------|-------------------|
| 1  | doc.pdf | legal,tax,review  |

-- 1NF: separate table for tags
files: | id | name    |
tags:  | file_id | tag    |
```

**In practice**: JSON/JSONB columns technically violate 1NF but are acceptable when the data is truly semi-structured (configuration blobs, API responses). Don't use JSONB to avoid creating a proper table for structured, queryable data.

### 2NF — No Partial Dependencies

Every non-key column depends on the **entire** composite key, not just part of it.

```sql
-- Violates 2NF: project_name depends only on project_id, not on (file_id, project_id)
| file_id | project_id | project_name | file_size |
|---------|------------|--------------|-----------|

-- 2NF: project_name moves to its own table
files:    | file_id | project_id | file_size |
projects: | project_id | project_name |
```

**In practice**: This only matters with composite primary keys. If you use surrogate keys (auto-incrementing `id`), 2NF violations are uncommon.

### 3NF — No Transitive Dependencies

Every non-key column depends directly on the primary key, not through another non-key column.

```sql
-- Violates 3NF: org_name depends on org_id, which depends on project_id
| file_id | project_id | org_id | org_name |
|---------|------------|--------|----------|

-- 3NF: each fact in its own table
files:    | file_id | project_id |
projects: | project_id | org_id |
orgs:     | org_id | org_name |
```

**In practice**: This is the most commonly violated normal form during denormalization. The `UploadedFile.project_id` denormalization is a deliberate 3NF violation — `project_id` transitively depends on `file_id` through `folder_id`.

### BCNF — Every Determinant is a Candidate Key

Stricter than 3NF. Rarely relevant in application databases.

### Beyond BCNF (4NF, 5NF)

Handle multi-valued dependencies and join dependencies. Almost never relevant in practice. If you're debugging a 4NF violation, you're either in academia or dealing with a very unusual schema.

---

## 3. The Cost of Normalization

Normalization isn't free. The cost is paid at **read time**.

### Join Overhead

Every normalized relationship requires a JOIN at query time. One join is cheap. Five joins are noticeable. Ten joins with filters on each table can be expensive.

```sql
-- Fully normalized: 5 joins to get a file's full context
SELECT f.name, fo.name, p.name, o.name, u.clerk_email
FROM uploaded_files f
JOIN folders fo ON f.folder_id = fo.id
JOIN projects p ON fo.project_id = p.id
JOIN organizations o ON p.organization_id = o.id
JOIN users u ON fo.user_id = u.id
WHERE f.uuid = '...';
```

Each join multiplies the planner's search space and can introduce nested loops, hash builds, or sort operations.

### Recursive Queries for Hierarchies

Normalized hierarchies (adjacency list pattern: `parent_folder_id` FK to self) require recursive CTEs or application-level loops to traverse. This is the most expensive normalized read pattern.

```sql
-- "Find all ancestors of folder X" requires recursion
WITH RECURSIVE ancestors AS (
    SELECT * FROM folders WHERE id = X
    UNION ALL
    SELECT f.* FROM folders f JOIN ancestors a ON f.id = a.parent_folder_id
)
SELECT * FROM ancestors;
```

The cost scales with tree depth, and PostgreSQL can't predict depth at planning time — leading to poor cost estimates.

### N+1 Queries in Application Code

ORM-based code often produces N+1 patterns with normalized schemas:

```python
# N+1: 1 query for folders + N queries for files
folders = await db.scalars(select(Folder).where(...))
for folder in folders:
    files = await db.scalars(select(UploadedFile).where(UploadedFile.folder_id == folder.id))
```

This is solvable with eager loading (`selectinload`, `joinedload`), but the temptation to write N+1 code is higher with deeply normalized schemas.

### Multiple Round-Trips in Distributed Systems

In microservice architectures, normalized data spread across services means multiple network calls:

```
Request → Service A (get file) → Service B (get project) → Service C (check membership)
```

Each hop adds latency. Denormalization within a service boundary eliminates inter-service calls.

---

## 4. What Denormalization Actually Solves

Denormalization trades **write complexity for read performance**. It stores redundant data to avoid expensive read-time computation.

### The Fundamental Trade-off

```
Normalized:    One source of truth    → Reads are joins    → Writes are simple
Denormalized:  Redundant copies exist → Reads are lookups  → Writes must sync copies
```

### What It Optimizes

1. **Eliminates joins**: Store `project_id` on the file instead of traversing `file → folder → project`.
2. **Eliminates recursion**: Pre-compute hierarchy results instead of walking trees at query time.
3. **Eliminates aggregation**: Store `file_count` on folders instead of `COUNT(*)` on every render.
4. **Eliminates cross-service calls**: Cache remote data locally instead of fetching per-request.

### What It Costs

1. **Storage**: Redundant data consumes disk and memory.
2. **Write amplification**: Changing the source requires updating all copies.
3. **Sync complexity**: More code paths to maintain, more places for bugs.
4. **Consistency risk**: Copies can drift if a sync path is missed.

---

## 5. Denormalization Patterns

### Pattern 1: Duplicated Column

Copy a column from a related table onto the current table.

```sql
-- Normalized
uploaded_files: | id | folder_id |
folders:        | id | project_id |

-- Denormalized: project_id copied onto uploaded_files
uploaded_files: | id | folder_id | project_id |
folders:        | id | project_id |
```

**When to use**: The copied value is read frequently, changes rarely, and has few mutation paths.

**Sync strategy**: Set at creation, update in the same transaction when the source changes.

**Real example**: `UploadedFile.project_id` denormalized from `Folder.project_id`. Set at file upload, synced in `add/remove_folder_from_project`.

### Pattern 2: Pre-computed Aggregate

Store a computed value that would otherwise require an aggregation query.

```sql
-- Normalized: count files on every request
SELECT COUNT(*) FROM uploaded_files WHERE folder_id = X;

-- Denormalized: store count on folder
folders: | id | file_count |
-- Update on every INSERT/DELETE to uploaded_files
```

**When to use**: The aggregation is expensive (large tables), reads vastly outnumber writes, and exact accuracy isn't critical (slight staleness is acceptable).

**Sync strategy**: Increment/decrement on write, or periodic recalculation.

**Watch out for**: Race conditions on concurrent writes. Use `UPDATE folders SET file_count = file_count + 1` (atomic increment), not read-modify-write.

### Pattern 3: Materialized View / Summary Table

Pre-compute an entire query result and store it as a table.

```sql
-- Materialized view: refreshed periodically
CREATE MATERIALIZED VIEW org_usage_daily AS
SELECT organization_id, date_trunc('day', created_at) AS day, COUNT(*) AS file_count
FROM uploaded_files
GROUP BY organization_id, day;

-- Refresh (locks the view during refresh in PG < 9.4, concurrent refresh available after)
REFRESH MATERIALIZED VIEW CONCURRENTLY org_usage_daily;
```

**When to use**: Complex aggregations over large datasets, analytics dashboards, reporting queries that don't need real-time accuracy.

**Sync strategy**: Periodic refresh (cron), event-driven refresh, or incremental maintenance.

### Pattern 4: Denormalized Lookup Table

Store a flattened version of a relationship for fast lookups.

```sql
-- Normalized: check file access via recursive folder → project → membership
-- Requires CTE or multiple joins

-- Denormalized: pre-computed access table
file_access: | user_id | file_id | granted_at |
-- One row per (user, file) pair that has access
```

**When to use**: The permission model is complex (nested groups, inheritance, multiple sources), and checking access is the hottest query path.

**Sync strategy**: Event-driven — insert/delete rows when membership or hierarchy changes.

**Watch out for**: Fan-out on permission changes (adding a user to a project with 100K files requires 100K inserts). Consider async processing with eventual consistency.

### Pattern 5: Cached Hierarchy (Closure Table / Materialized Path)

Store transitive relationships explicitly to avoid recursion.

```sql
-- Closure table: every ancestor-descendant pair
folder_closure: | ancestor_id | descendant_id | depth |
-- For tree: A → B → C
-- Rows: (A,A,0), (A,B,1), (A,C,2), (B,B,0), (B,C,1), (C,C,0)

-- "All descendants of A" — no recursion
SELECT descendant_id FROM folder_closure WHERE ancestor_id = A;
```

**When to use**: Hierarchical data with frequent ancestor/descendant queries, moderate mutation rate.

**Sync strategy**: On insert, add closure rows for all ancestors. On move, delete old closure rows and insert new ones.

### Pattern 6: JSONB Embedding

Embed related data directly as JSONB instead of a separate table.

```sql
-- Normalized
files:    | id | name |
metadata: | file_id | key | value |

-- Denormalized: metadata embedded as JSONB
files: | id | name | metadata JSONB |
-- {"author": "John", "pages": 42, "language": "en"}
```

**When to use**: The embedded data is always read together with the parent, rarely queried independently, and has variable structure.

**Watch out for**: JSONB can't enforce referential integrity, has no schema validation (without CHECK constraints), and partial updates require rewriting the entire JSONB value (though `jsonb_set` helps).

---

## 6. Decision Framework

### Step-by-Step Evaluation

```
1. Is there a measured performance problem?
   └── No  → Stay normalized. Stop here.
   └── Yes ↓

2. Can you fix it with an index, query rewrite, or eager loading?
   └── Yes → Do that instead. Stop here.
   └── No  ↓

3. What's the read:write ratio for the affected data?
   └── Write-heavy (writes > reads) → Don't denormalize. Optimize writes instead.
   └── Read-heavy (reads >> writes) ↓

4. How many code paths mutate the source data?
   └── Many (>5)  → High sync risk. Consider caching (Redis, materialized view)
   |                 with TTL instead of denormalization.
   └── Few (1-4)  ↓

5. How often does the source data change?
   └── Frequently (per-request) → Denormalization has high write amplification.
   |                               Consider materialized views with periodic refresh.
   └── Rarely (config-level)    ↓

6. Can you sync the denormalized copy in the same transaction?
   └── No  → You'll have eventual consistency. Is that acceptable?
   |   └── No  → Don't denormalize.
   |   └── Yes → Use event-driven sync with monitoring.
   └── Yes ↓

7. Denormalize. Document:
   - What column/table is denormalized
   - What it mirrors (source of truth)
   - Every mutation path that must sync it
   - How to verify consistency (audit query)
```

### Scorecard

Rate each factor 1-5, multiply by weight:

| Factor | Weight | Favors Normalization (1) | Favors Denormalization (5) |
|---|---|---|---|
| Read frequency | 3x | Rare reads | Hot path, every request |
| Write frequency of source | 2x | Changes every request | Changes rarely or never |
| Number of mutation paths | 2x | Many (>5 code paths) | Few (1-3 code paths) |
| Query complexity | 2x | Simple JOIN | Recursive CTE, multi-hop |
| Data consistency requirement | 1x | Must be real-time consistent | Eventual consistency OK |
| Team familiarity | 1x | Team expects normalized | Team understands trade-offs |

**Score > 30**: Strong case for denormalization.
**Score 20-30**: Consider it, but weigh the maintenance cost.
**Score < 20**: Stay normalized.

---

## 7. Real-World Case Study: File Access Checks

### The Problem

Hierarchical folder structure: `Organization → Project → Folder (recursive) → File`.

To check if a user can access a file, the system walked the folder tree via a recursive CTE to find the project, then checked project membership.

### The Numbers

| Metric | Value |
|---|---|
| Query frequency | 67,000+ calls/day |
| Single-file latency | 0.07ms (recursive CTE) |
| Batch latency (50-500 files) | 5.08ms (recursive CTE) |
| Source data change frequency | Near zero (cross-project moves disallowed) |
| Mutation paths | 4 code paths |

### The Decision

Applied the framework:
1. Performance problem? Marginal now, but CTE cost scales with folder depth.
2. Index fix? Already indexed. The problem was structural (recursion).
3. Read:write ratio? 67K reads/day vs ~0 project_id changes/day. Extreme read-heavy.
4. Mutation paths? 4 — manageable.
5. Change frequency? Effectively never (cross-project moves blocked).
6. Same-transaction sync? Yes — all 4 paths update in the same `db.commit()`.

**Decision: Denormalize.** Added `project_id` directly on `uploaded_files`.

### The Result

| Metric | Before | After |
|---|---|---|
| Single-file check | 0.07ms (CTE) | ~0.03ms (index lookup) |
| Batch check | 5.08ms (CTE) | ~0.5ms (index scan) |
| Code complexity | 30-line CTE per call site | 1 function call |
| Inline duplicates | 8 copies of access check logic | 1 shared helper |

### What Made It Safe

- Cross-project moves are disallowed (invariant enforced at API level)
- All sync happens in the same database transaction
- Shared helper function (`file_access_condition`) is the single source of truth
- Column has a comment explaining what it mirrors and when it's updated
- Backfill script exists for the migration

---

## 8. Consistency Strategies for Denormalized Data

### Strategy 1: Same-Transaction Sync (Strongest)

Update the source and all copies in the same database transaction.

```python
# Both updates commit or neither does
folder.project_id = new_project_id
await db.execute(
    update(UploadedFile)
    .where(UploadedFile.folder_id.in_(all_folder_ids))
    .values(project_id=new_project_id)
)
await db.commit()  # atomic
```

**Guarantees**: Strong consistency. No window of inconsistency.
**Limitation**: All copies must be in the same database.

### Strategy 2: Application-Level Sync (Moderate)

Update the source, then update copies in a separate step (but same request).

```python
await update_project_membership(user_id, project_id)  # source
await refresh_file_access_cache(user_id, project_id)   # copy
```

**Risk**: If the second call fails (crash, timeout), copies are stale until next retry.
**Mitigation**: Retry logic, periodic consistency checks.

### Strategy 3: Event-Driven Async Sync (Eventual)

Emit an event when the source changes; a consumer updates copies asynchronously.

```
Source change → Event (Kafka/SQS/DB trigger) → Consumer → Update copies
```

**Guarantees**: Eventual consistency. There is a lag window.
**Best for**: High fan-out operations (changing a permission that affects millions of rows).
**Mitigation**: Monitor lag, alert on drift, provide consistency check endpoints.

### Strategy 4: Periodic Reconciliation (Weakest)

A scheduled job compares source and copies, fixing any drift.

```sql
-- Find inconsistencies
SELECT uf.id, uf.project_id AS file_project, f.project_id AS folder_project
FROM uploaded_files uf
JOIN folders f ON uf.folder_id = f.id
WHERE uf.project_id != f.project_id;
```

**Best for**: Belt-and-suspenders safety net alongside another strategy. Not sufficient on its own for data that must be accurate.

### Choosing a Strategy

| Strategy | Consistency | Complexity | Best When |
|---|---|---|---|
| Same-transaction | Strong | Low | Same database, few copies |
| Application-level | Near-strong | Medium | Same service, multiple steps |
| Event-driven | Eventual | High | Cross-service, high fan-out |
| Periodic reconciliation | Weak | Low | Safety net, audit compliance |

---

## 9. When Companies Denormalize

### Google Drive — Pre-computed Permission Lists

Every file stores its effective permission list. When a folder's permissions change, Google propagates to all descendants asynchronously. They accept eventual consistency (you might see a file for a few seconds after losing access).

### Twitter/X — Timeline Fanout

When you tweet, Twitter writes copies of the tweet ID into every follower's timeline cache (fan-out on write). Reading a timeline is a simple cache read, not a complex query joining followers × tweets.

### Shopify — Denormalized Product Data

Product listing pages don't join 15 tables. Shopify stores denormalized product documents (name, price, images, variants) for fast reads. Writes go through a single service that updates the normalized source and the denormalized copies.

### Stripe — Materialized Balances

Account balances are pre-computed and stored, not calculated from transaction history on every API call. Every transaction updates the materialized balance atomically.

### Common Thread

All of these share the same pattern:
1. The read path is extremely hot (millions of requests/second)
2. The source data changes through a controlled set of operations
3. They accept some form of eventual consistency or invest heavily in sync infrastructure
4. They monitor for drift and have reconciliation processes

---

## 10. Anti-Patterns and Mistakes

### 1. Premature Denormalization

**Symptom**: "This JOIN might be slow someday, let's denormalize now."

**Problem**: You're adding sync complexity for a problem that doesn't exist. The JOIN might never be slow. If it becomes slow, an index might fix it.

**Rule**: Measure first, then decide. Check query insights, run `EXPLAIN ANALYZE`, profile the hot path. If the query is < 10ms and called < 1000 times/day, it's not a problem.

### 2. Denormalizing Volatile Data

**Symptom**: Copying a value that changes on every request.

```sql
-- Bad: last_login changes constantly
users:    | id | last_login |
sessions: | id | user_id | user_last_login |  ← stale immediately
```

**Problem**: Every source change triggers a cascade of updates. Write amplification kills performance. The copies are always stale.

**Rule**: Only denormalize data that changes through few, well-defined operations. "Few" typically means 1-5 mutation paths.

### 3. Unbounded Fan-Out

**Symptom**: One source change requires updating millions of rows.

```
-- Adding a user to a project with 1M files → 1M file_access inserts
```

**Problem**: The write becomes slower than the read it was trying to optimize. Long-running transactions, lock contention, replication lag.

**Rule**: If the fan-out ratio exceeds ~10,000:1, use async processing or a different data model. Or avoid the materialized table entirely if the live join is fast enough.

### 4. Forgetting a Mutation Path

**Symptom**: Denormalized data is correct for most operations but inconsistent after a specific, rare operation.

**Problem**: You identified 3 of 4 mutation paths. The 4th path (e.g., an admin bulk-move endpoint added 6 months later) doesn't sync the copy.

**Mitigation**:
- Document every mutation path in a comment on the denormalized column
- Create a shared helper function that all mutation paths must use
- Add a periodic reconciliation check that alerts on drift
- Code review checklist: "Does this endpoint change X? If so, does it sync Y?"

### 5. Denormalizing Across Consistency Boundaries

**Symptom**: Source in Database A, copy in Database B (or Redis, or another service).

```
-- Source updated in PostgreSQL
-- Copy updated in Redis
-- Server crashes between the two → permanent inconsistency
```

**Problem**: You can't use a database transaction to keep both in sync. You now need distributed transactions (2PC) or saga patterns.

**Rule**: If the source and copy are in different systems, you must accept eventual consistency. Build monitoring and reconciliation. Don't pretend it's strongly consistent.

### 6. Over-Denormalizing

**Symptom**: Copying 5 columns from 3 different tables onto one table.

```sql
-- uploaded_files with everything denormalized
| id | folder_id | project_id | project_name | org_id | org_name | user_email |
```

**Problem**: Each additional column multiplies the sync surface. Changing a project name now requires updating every file row. Renaming a user requires touching every file they uploaded.

**Rule**: Denormalize the minimum needed. If you only need `project_id` for access checks, don't also copy `project_name`. The JOIN to get the name is cheap and the name changes more often than the ID.

---

## 11. Hybrid Approaches

### Computed Columns (Generated Columns)

PostgreSQL 12+ supports stored generated columns — computed from other columns in the same row, always in sync.

```sql
ALTER TABLE uploaded_files ADD COLUMN search_text TEXT
  GENERATED ALWAYS AS (name || ' ' || COALESCE(content_type, '')) STORED;
```

**Limitation**: Can only reference columns in the same row, not other tables. Not useful for cross-table denormalization.

### Database Triggers

Automatically sync copies when the source changes.

```sql
CREATE FUNCTION sync_file_project_id() RETURNS TRIGGER AS $$
BEGIN
    UPDATE uploaded_files SET project_id = NEW.project_id
    WHERE folder_id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_file_project_id
    AFTER UPDATE OF project_id ON folders
    FOR EACH ROW EXECUTE FUNCTION sync_file_project_id();
```

**Pros**: Sync is guaranteed at the database level, impossible to miss a code path.
**Cons**: Hidden logic (not in application code), harder to debug, performance implications on high-write tables, doesn't handle subfolder recursion well.

### Materialized Views with CONCURRENTLY

Pre-computed query results that refresh without blocking reads.

```sql
CREATE MATERIALIZED VIEW file_access_mv AS
SELECT uf.uuid AS file_id, pum.user_id
FROM uploaded_files uf
JOIN folders f ON uf.folder_id = f.id
JOIN project_user_memberships pum ON f.project_id = pum.project_id;

-- Refresh without locking (requires a unique index)
CREATE UNIQUE INDEX ON file_access_mv (file_id, user_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY file_access_mv;
```

**Best for**: Analytics, dashboards, read-heavy queries where staleness up to N minutes is acceptable.

### Application-Level Caching (Redis)

Cache the result of expensive queries with a TTL.

```python
cache_key = f"file_access:{user_id}:{file_id}"
cached = await redis.get(cache_key)
if cached is not None:
    return cached == "1"

has_access = await check_file_access(file_id, user_id, org_id)
await redis.setex(cache_key, 300, "1" if has_access else "0")  # 5 min TTL
return has_access
```

**Best for**: Expensive queries with high repetition, where 5-minute staleness is acceptable.
**Watch out for**: Cache invalidation is hard. TTL-based expiry is simple but means stale data for up to TTL seconds. Event-driven invalidation is correct but complex.

---

## 12. Quick Reference Cheat Sheet

### When to Stay Normalized

- No measured performance problem
- Data changes frequently through many code paths
- Strong consistency is required (financial, legal, audit)
- The JOIN is cheap (well-indexed, small tables)
- The team is small and sync code would be a maintenance burden

### When to Denormalize

- Measured read performance problem that indexes can't solve
- Source data changes rarely through few (1-4) code paths
- Can sync in the same database transaction
- Read:write ratio is > 100:1
- The alternative is a recursive CTE or multi-hop query

### Denormalization Checklist

Before denormalizing, ensure you have:

- [ ] Measured the current query performance (EXPLAIN ANALYZE, Query Insights)
- [ ] Confirmed indexes can't solve the problem
- [ ] Identified every mutation path for the source data
- [ ] Planned the sync strategy (same-transaction, event-driven, etc.)
- [ ] Added a comment on the denormalized column explaining what it mirrors
- [ ] Created a shared helper function for all reads (single source of truth)
- [ ] Written a backfill script for existing data
- [ ] Documented a reconciliation query to detect drift
- [ ] Considered the migration strategy (nullable → backfill → NOT NULL)

### Reconciliation Query Template

```sql
-- Detect drift between denormalized column and source of truth
SELECT child.id, child.denormalized_col, parent.source_col
FROM child_table child
JOIN parent_table parent ON child.parent_id = parent.id
WHERE child.denormalized_col != parent.source_col
   OR (child.denormalized_col IS NULL AND parent.source_col IS NOT NULL);
```

### Red Flags That Denormalization Has Gone Wrong

| Signal | Meaning |
|---|---|
| Reconciliation query returns rows | Sync path was missed or failed |
| New endpoint added that changes source without syncing | Documentation/review gap |
| Denormalized column updated more often than read | Wrong trade-off — re-normalize |
| Multiple "fix denormalized data" scripts in git history | Systemic sync problem |
| Different values for the same fact in different tables | Consistency has drifted |

---

## Further Reading

- [Martin Kleppmann — Designing Data-Intensive Applications, Ch. 2-3](https://dataintensive.net/)
- [Database Normalization (Wikipedia)](https://en.wikipedia.org/wiki/Database_normalization)
- [Markus Winand — SQL Performance Explained](https://sql-performance-explained.com/)
- [PostgreSQL Documentation — Materialized Views](https://www.postgresql.org/docs/current/rules-materializedviews.html)
- [Stripe Engineering — Online Migrations at Scale](https://stripe.com/blog/online-migrations)
