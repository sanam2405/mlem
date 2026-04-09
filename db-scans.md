# PostgreSQL Query Planning & Scan Types

A comprehensive guide to understanding how PostgreSQL plans and executes queries, using real-world examples from production systems.

---

## Table of Contents

1. [How Query Planning Works](#1-how-query-planning-works)
2. [Scan Types (Leaf Nodes)](#2-scan-types-leaf-nodes)
3. [Join Strategies](#3-join-strategies)
4. [Bitmap Operations](#4-bitmap-operations)
5. [CTE & Recursive Execution](#5-cte--recursive-execution)
6. [Aggregation & Sorting Nodes](#6-aggregation--sorting-nodes)
7. [Reading EXPLAIN Output](#7-reading-explain-output)
8. [Cost Model Deep Dive](#8-cost-model-deep-dive)
9. [Real-World Example: Recursive ACL Query](#9-real-world-example-recursive-acl-query)
10. [Index Design Principles](#10-index-design-principles)
11. [Common Pitfalls & Anti-Patterns](#11-common-pitfalls--anti-patterns)
12. [Tuning Knobs That Matter](#12-tuning-knobs-that-matter)
13. [Quick Reference Cheat Sheet](#13-quick-reference-cheat-sheet)

---

## 1. How Query Planning Works

When PostgreSQL receives a query, it goes through these stages:

```
SQL text → Parser → Rewriter → Planner/Optimizer → Executor → Results
```

### The Planner's Job

The planner generates multiple possible **execution plans** and picks the one with the lowest estimated **cost**. It considers:

- **Table statistics**: row counts, column distributions, most-common values, null fractions (stored in `pg_statistic`, viewable via `pg_stats`)
- **Available indexes**: which indexes exist, their types (B-tree, GiST, GIN, Hash), and column ordering
- **Join order**: for N tables, there are N! possible join orderings — the planner prunes this search space
- **Operator costs**: configurable constants like `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`

### Plan Structure

Plans are **trees of nodes**. Data flows from leaf nodes (scans) upward through join/filter/sort nodes to the root (final result). Each node:
- Produces a stream of tuples (rows)
- May consume tuples from child nodes
- Has an estimated **startup cost** (time before first row) and **total cost** (time for all rows)

```
                    Result (root)
                       |
                  Nested Loop
                 /           \
          Index Scan       Index Scan
         (uploaded_files)   (folders)
```

### EXPLAIN vs EXPLAIN ANALYZE

| Command | What it shows | Hits the DB? |
|---|---|---|
| `EXPLAIN` | Estimated costs, row counts, plan structure | No — planning only |
| `EXPLAIN ANALYZE` | All of the above + **actual** execution time, actual rows, loops | **Yes — runs the query** |
| `EXPLAIN (ANALYZE, BUFFERS)` | All of the above + buffer hits/reads (cache vs disk) | Yes |
| `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` | Machine-readable format with all details | Yes |

**Always use `EXPLAIN ANALYZE` when diagnosing real performance** — estimates can be wildly wrong if statistics are stale.

---

## 2. Scan Types (Leaf Nodes)

Scan nodes are the leaves of the plan tree — they read data from tables or indexes.

### Sequential Scan (`Seq Scan`)

Reads **every row** in the table, page by page, in physical storage order.

```
Seq Scan on users  (cost=0.00..35.50 rows=2550 width=4)
  Filter: (organization_id = 1)
```

**When the planner chooses this:**
- Table is small (fits in a few pages)
- Query needs most of the table (low selectivity filter)
- No useful index exists
- `random_page_cost` is high relative to `seq_page_cost` (makes index scans look expensive)

**Key insight**: A sequential scan is not always bad. For a 100-row table, it's faster than an index scan because it avoids the index traversal overhead. The planner knows this.

### Index Scan

Traverses the B-tree index to find matching entries, then **fetches each row from the heap (table)** using the tuple pointer (TID) stored in the index.

```
Index Scan using uploaded_files_uuid_key on uploaded_files
  Index Cond: (uuid = 'abc-123'::uuid)
  Rows Removed by Filter: 0
```

**Two I/O operations per row**: one to read the index leaf page, one to read the heap page. The heap fetch is a **random I/O** (expensive on spinning disks, cheap on SSDs).

**When the planner chooses this:**
- High selectivity (few rows match) — typically <5-15% of the table
- Index columns match the WHERE clause
- ORDER BY matches the index ordering (avoids a separate sort)

### Index Only Scan

Like an Index Scan but **never touches the heap**. All required columns are in the index itself (a "covering index").

```
Index Only Scan using idx_pum_user_project on project_user_memberships
  Index Cond: (user_id = 42)
  Heap Fetches: 0
```

**The `Heap Fetches` number matters**: PostgreSQL must check the **visibility map** to confirm the row hasn't been modified by a concurrent transaction. If the page isn't all-visible (e.g., recently updated, not yet VACUUMed), it must fetch from the heap anyway. A high `Heap Fetches` on an Index Only Scan means VACUUM is behind.

**When the planner chooses this:**
- All columns in SELECT, WHERE, and JOIN are in the index
- The visibility map is mostly clean (recent VACUUM)

**Pro tip**: Create covering indexes with `INCLUDE` for frequently queried columns:
```sql
CREATE INDEX idx_files_folder ON uploaded_files (folder_id) INCLUDE (uuid, name);
-- Now queries for uuid and name filtered by folder_id can use Index Only Scan
```

### Bitmap Index Scan + Bitmap Heap Scan

A **two-phase** approach that combines the precision of an index with the efficiency of sequential reads.

**Phase 1 — Bitmap Index Scan**: Traverses the index and builds a **bitmap** (bitset) of heap pages that contain matching rows. Does NOT fetch any heap data yet.

**Phase 2 — Bitmap Heap Scan**: Reads the heap pages identified by the bitmap **in physical order** (converting random I/O into sequential I/O). Rechecks the condition on each row.

```
Bitmap Heap Scan on folders
  Recheck Cond: ((project_id = 5) AND (organization_id = 1))
  -> BitmapAnd
       -> Bitmap Index Scan on ix_folders_project_id
            Index Cond: (project_id = 5)
       -> Bitmap Index Scan on ix_folders_organizato
            Index Cond: (organization_id = 1)
```

**Why "Recheck Cond"?** If there are more matching rows than fit in the bitmap's page-level granularity, PostgreSQL switches from exact (per-tuple) to lossy (per-page) tracking. On lossy pages, it must recheck each tuple against the original condition.

**When the planner chooses this:**
- Moderate selectivity — too many rows for an Index Scan (random I/O too expensive), too few for a Seq Scan
- Multiple indexes can be combined (BitmapAnd, BitmapOr)
- Typically 5-25% of the table

### BitmapAnd / BitmapOr

Combine bitmaps from multiple indexes using set operations **before** touching the heap.

```
BitmapAnd
  -> Bitmap Index Scan on idx_a  (condition A)
  -> Bitmap Index Scan on idx_b  (condition B)
```

This is how PostgreSQL uses **two single-column indexes** to efficiently filter on `WHERE col_a = X AND col_b = Y` without a composite index. The BitmapAnd intersects the two bitmaps, then a single Bitmap Heap Scan fetches only pages that match both conditions.

**Trade-off**: A single composite index `(col_a, col_b)` is still faster than BitmapAnd on two separate indexes, because it avoids building and intersecting bitmaps. But BitmapAnd is the planner's clever way to use what's available.

### Worktable Scan

Reads from a **temporary worktable** created by a recursive CTE. This is the recursive step re-reading its own output from the previous iteration.

```
WorkTable Scan on project_folders_hierarchy
```

This is internal to recursive query execution — see [Section 5](#5-cte--recursive-execution) for details.

### CTE Scan

Reads from a materialized CTE result. Prior to PostgreSQL 12, all CTEs were materialized (computed once, stored in memory/temp files). From PG12+, the planner can inline non-recursive CTEs.

```
CTE Scan on project_folders_hierarchy
  Filter: (id = uf.folder_id)
```

**Important**: Recursive CTEs are **always** materialized — they cannot be inlined. This means the entire recursive result set is computed before any downstream node reads from it.

### Function Scan

Reads rows from a set-returning function (e.g., `generate_series`, `unnest`, custom functions).

```
Function Scan on generate_series g  (cost=0.00..10.00 rows=1000)
```

### Values Scan

Reads from an inline `VALUES` clause.

### Subquery Scan

Wraps a subquery's output as a scannable node. Often appears with `UNION` or `UNION ALL`.

---

## 3. Join Strategies

PostgreSQL has three join algorithms. The planner picks based on table sizes, sort order, and available memory.

### Nested Loop

For each row in the **outer** (left) table, scan the **inner** (right) table for matches.

```
Nested Loop
  -> Index Scan on uploaded_files  (outer — 1 row)
  -> Index Scan on folders         (inner — executed once per outer row)
```

- **Best for**: Small outer table, indexed inner table. Cost is O(N × index_lookup) ≈ O(N log M).
- **Worst case**: Both tables are large with no index. O(N × M) — catastrophic.
- **The `loops` column in EXPLAIN ANALYZE tells you how many times the inner node executed.** If `loops=10000`, the inner node ran 10,000 times — multiply its per-loop time by loops for total.

### Hash Join

**Phase 1 (Build)**: Read the smaller table, hash each row's join key, store in an in-memory hash table.
**Phase 2 (Probe)**: Read the larger table, hash each row's join key, probe the hash table for matches.

```
Hash Join
  Hash Cond: (uf.folder_id = f.id)
  -> Seq Scan on uploaded_files
  -> Hash
       -> Seq Scan on folders
```

- **Best for**: Large tables with no useful index on the join key, or when the smaller table fits in `work_mem`.
- **Cost**: O(N + M) — linear in both table sizes. Very efficient.
- **Watch for**: `Batches: 4` in EXPLAIN ANALYZE means the hash table didn't fit in `work_mem` and spilled to disk. Increase `work_mem` or it becomes much slower.

### Merge Join

Both inputs are sorted on the join key (or come pre-sorted from an index). Walk through both in lockstep.

```
Merge Join
  Merge Cond: (a.id = b.a_id)
  -> Index Scan on a
  -> Sort
       -> Seq Scan on b
```

- **Best for**: Both inputs are already sorted (e.g., from an index scan on the join column). No extra memory needed.
- **Cost**: O(N log N + M log M) if sorting needed, O(N + M) if pre-sorted.
- **Produces sorted output** — useful if the query also has `ORDER BY` on the join key.

### Join Strategy Comparison

| Strategy | Time Complexity | Memory | Pre-sorted Input Needed? | Best When |
|---|---|---|---|---|
| Nested Loop | O(N × lookup) | O(1) | No | Small outer, indexed inner |
| Hash Join | O(N + M) | O(smaller table) | No | Large tables, no index |
| Merge Join | O(N + M) if sorted | O(1) | Yes (or adds sort) | Both sides sorted |

---

## 4. Bitmap Operations

Bitmaps are PostgreSQL's way of bridging the gap between "index scan is too many random I/Os" and "sequential scan reads too much data."

### How the Bitmap Works Internally

1. The bitmap is a **page-level** bitset: one bit per heap page (8KB block). A '1' means "this page has at least one matching row."
2. If the result set is small enough, PostgreSQL uses an **exact (per-tuple)** bitmap instead, tracking individual row positions.
3. If memory is tight, it degrades from exact → lossy (per-page), which is why Bitmap Heap Scan has a "Recheck Cond."

### When BitmapAnd Beats a Composite Index

Almost never for pure performance. But BitmapAnd is valuable when:
- You have many different query patterns hitting different column combinations
- You can't create composite indexes for every combination
- The individual indexes serve other queries too

A composite index `(a, b)` can satisfy `WHERE a = X AND b = Y` in one tree traversal. BitmapAnd on `idx_a` + `idx_b` needs two traversals plus a bitmap intersection. But if you also have queries with just `WHERE a = X` and just `WHERE b = Y`, two single-column indexes serve all three patterns.

---

## 5. CTE & Recursive Execution

### Non-Recursive CTEs (PG12+)

```sql
WITH active_users AS (
    SELECT * FROM users WHERE deleted_at IS NULL
)
SELECT * FROM active_users WHERE id = 42;
```

From PG12, the planner can **inline** this — pushing the `id = 42` filter into the CTE, making it equivalent to `SELECT * FROM users WHERE deleted_at IS NULL AND id = 42`. Before PG12 (or with `MATERIALIZED` hint), the CTE computes all active users first, then filters.

Force materialization: `WITH active_users AS MATERIALIZED (...)`
Force inlining: `WITH active_users AS NOT MATERIALIZED (...)`

### Recursive CTEs

```sql
WITH RECURSIVE folder_tree AS (
    -- Base case (non-recursive term)
    SELECT id, parent_folder_id, name FROM folders WHERE project_id = 5

    UNION ALL

    -- Recursive term
    SELECT f.id, f.parent_folder_id, f.name
    FROM folders f
    JOIN folder_tree ft ON f.parent_folder_id = ft.id
)
SELECT * FROM folder_tree;
```

**Execution model:**

1. Execute the **base case**. Store results in a **worktable** (temporary result set).
2. Execute the **recursive term**, reading from the worktable (WorkTable Scan). Store new results in an **intermediate table**.
3. If the intermediate table is non-empty, swap it into the worktable and go to step 2.
4. When the intermediate table is empty (no new rows), stop. Return the union of all iterations.

**In the query plan, you'll see:**

```
Recursive Union                     ← orchestrates the loop
  -> Index Scan on folders          ← base case (runs once)
  -> Nested Loop                    ← recursive term (runs per iteration)
       -> WorkTable Scan on folder_tree  ← reads previous iteration's output
       -> Index Scan on folders          ← joins to find children
```

**`Recursive Union` cost**: The estimated cost (`1,289.06` in the real example) reflects the planner's guess at how many iterations the recursion will need, based on table statistics. This estimate is often inaccurate for deep or wide trees.

**Performance characteristics:**
- Each iteration does an Index Scan for every row in the worktable
- For a tree of depth D with branching factor B: ~D iterations, each processing B^level rows
- Total work: O(N) where N is the number of nodes in the subtree (every node is visited exactly once)
- Memory: the worktable holds one level at a time, not the entire result

### UNION vs UNION ALL in Recursive CTEs

- `UNION ALL`: Allows duplicate rows. Faster (no dedup). **Use this for trees** — a tree node can't appear twice if the data is a proper tree.
- `UNION`: Deduplicates across iterations. Required for **graphs with cycles** to prevent infinite loops. But slower (must hash or sort to dedup).

---

## 6. Aggregation & Sorting Nodes

### Sort

Sorts tuples by one or more keys. Uses quicksort in memory, or external merge sort (disk) if the data exceeds `work_mem`.

```
Sort  (cost=1000.00..1025.00 rows=10000)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 1024kB     ← fits in memory, good
  -- or --
  Sort Method: external merge  Disk: 4096kB  ← spilled to disk, slow
```

**Tip**: If you see `external merge`, increase `work_mem` for that query:
```sql
SET LOCAL work_mem = '64MB';  -- per-transaction override
```

### HashAggregate

Groups rows by hashing the GROUP BY keys. Builds an in-memory hash table with one entry per group.

```
HashAggregate  (cost=100.00..110.00 rows=50)
  Group Key: organization_id
```

- Fast for moderate numbers of groups
- Can spill to disk if groups exceed `work_mem` (PG13+: `Batches: N` in EXPLAIN ANALYZE)

### GroupAggregate

Groups rows that are **already sorted** on the GROUP BY key. Walks through sorted input, accumulating each group.

- No extra memory needed (O(1) per group)
- Requires sorted input (from index or preceding Sort node)

### Unique

Removes duplicates from sorted input. Like GroupAggregate but simpler.

### Limit

Stops execution after N rows. **This is one of the most powerful nodes** — it can short-circuit the entire subtree. A `Limit 1` on an Index Scan means the scan stops after finding the first match.

### Materialize

Caches the output of a child node in memory (or temp files) so it can be re-scanned. Appears when a Nested Loop's inner side needs to be scanned multiple times and isn't an index scan.

---

## 7. Reading EXPLAIN Output

### The Format

```
Nested Loop  (cost=0.57..16.62 rows=1 width=16) (actual time=0.025..0.030 rows=1 loops=1)
```

| Field | Meaning |
|---|---|
| `cost=0.57..16.62` | Estimated startup cost..total cost (in arbitrary "cost units") |
| `rows=1` | Estimated number of rows this node will produce |
| `width=16` | Estimated average row width in bytes |
| `actual time=0.025..0.030` | Real startup time..total time in milliseconds **per loop** |
| `rows=1` (actual) | Real number of rows produced **per loop** |
| `loops=1` | Number of times this node was executed |

### Critical Rule: Multiply by Loops

If `loops=100` and `actual time=0.01..0.05`, the **true** time for this node is `0.05 × 100 = 5ms`, not 0.05ms. This is the most common mistake when reading EXPLAIN ANALYZE output.

### Estimated vs Actual Row Counts

When estimated `rows` differs dramatically from actual `rows`, the planner made a bad choice. Common causes:
- **Stale statistics**: run `ANALYZE tablename`
- **Correlated columns**: planner assumes independence (e.g., `city = 'Mumbai' AND country = 'India'` — it multiplies selectivities, underestimating matches)
- **Skewed data**: common values in `pg_stats.most_common_vals` might not cover the queried value

### Buffers Output

With `EXPLAIN (ANALYZE, BUFFERS)`:

```
Buffers: shared hit=5 read=2
```

| Field | Meaning |
|---|---|
| `shared hit=5` | 5 pages found in PostgreSQL's shared buffer cache (RAM) |
| `shared read=2` | 2 pages read from OS / disk |
| `shared dirtied=1` | 1 page modified (for writes) |
| `shared written=1` | 1 dirty page flushed to disk |
| `temp read/written` | Spilled to temp files (bad — increase `work_mem`) |

**A high `shared read` vs `shared hit` ratio means cold cache.** Run the query twice — the second run shows the warm-cache profile.

---

## 8. Cost Model Deep Dive

PostgreSQL's cost is a unitless number, but it's calibrated around **sequential page reads**.

### Key Cost Constants

| Parameter | Default | Meaning |
|---|---|---|
| `seq_page_cost` | 1.0 | Cost of reading one 8KB page sequentially |
| `random_page_cost` | 4.0 | Cost of reading one 8KB page randomly (seek penalty) |
| `cpu_tuple_cost` | 0.01 | Cost of processing one row |
| `cpu_index_tuple_cost` | 0.005 | Cost of processing one index entry |
| `cpu_operator_cost` | 0.0025 | Cost of applying one operator (=, <, etc.) |
| `effective_cache_size` | 4GB | Planner's estimate of total available cache (OS + PG) |

### Why `random_page_cost` Matters

On SSDs, random reads are nearly as fast as sequential reads. The default `4.0` (calibrated for spinning disks) makes the planner **over-penalize index scans** on SSDs. For SSD-backed databases (including Cloud SQL):

```sql
SET random_page_cost = 1.1;  -- common SSD tuning
```

This makes the planner more willing to use index scans.

### How the Planner Estimates Selectivity

For `WHERE organization_id = 1`:

1. Check `pg_stats` for `organization_id`: `n_distinct`, `most_common_vals`, `most_common_freqs`
2. If `1` is in `most_common_vals`, use its frequency directly
3. Otherwise, distribute remaining frequency uniformly: `(1 - sum(MCF)) / (n_distinct - len(MCV))`
4. Multiply selectivity by `reltuples` to estimate rows

For compound conditions (`AND`): multiply selectivities (assumes independence).
For `OR`: `P(A) + P(B) - P(A)×P(B)`.

---

## 9. Real-World Example: Recursive ACL Query

This is the query plan from the production file access check CTE:

```
Index Scan on uploaded_files using uploaded_files_uuid_key    ← Find the file by UUID
  Rows returned: 1, Latency: <1ms, Cost: 8.47
  |
  ├── CTE Scan (project_folders_hierarchy)                    ← Read materialized CTE result
  |     Rows returned: 0, Cost: 0.22
  |
  ├── Index Scan on folders using folders_pkey                ← Check: does user own the folder?
  |     Rows returned: 1, Latency: <1ms, Cost: 8.44
  |
  └── Recursive Union                                         ← Build the folder hierarchy
        Cost: 1,289.06 (highest cost node)
        |
        ├── Nested Loop                                       ← Base case: project folders × user memberships
        |     Cost: 0.01
        |     ├── Index Only Scan on project_user_memberships
        |     |     using idx_pum_user_project
        |     |     Rows returned: 0, Cost: 8.3
        |     └── Bitmap Heap Scan on folders                 ← Find folders by project_id AND org_id
        |           Cost: 4.02
        |           └── BitmapAnd
        |                 ├── Bitmap Index Scan (ix_folders_project_id)     Cost: 4.5
        |                 └── Bitmap Index Scan (ix_folders_organizato...)  Cost: 8.17
        |
        └── Nested Loop                                       ← Recursive step: find children
              Cost: 129.12
              ├── WorkTable Scan on project_folders_hierarchy  ← Read previous iteration
              |     Cost: 0.2
              └── Index Scan on folders                        ← Match parent_folder_id
                    using ix_folders_parent_fold...
                    Cost: 13.34
```

### Why It's Fast (0.09ms average)

1. **The file lookup is by UUID unique index** — always 1 row, O(log N)
2. **The folder ownership check is by primary key** — always 1 row, O(log N)
3. **The recursive CTE terminates quickly** — most folders have shallow nesting (2-4 levels)
4. **BitmapAnd on project_id + organization_id** combines two indexes efficiently
5. **CTE Scan returns 0 rows** in most cases — the ownership check (path 1) succeeds first, so the CTE result isn't even needed
6. **Index Only Scan on project_user_memberships** avoids heap access entirely

### Why the Estimated Cost is High (1,289) but Actual is Fast (0.09ms)

The planner **overestimates** the recursive union cost because it can't know in advance how deep the recursion will go. It assumes worst-case depth based on statistics. In practice, the recursion terminates in 1-3 iterations with few rows per iteration.

---

## 10. Index Design Principles

### The Golden Rules

**1. Index columns in this order: equality → range → sort → cover**

```sql
-- Query: WHERE org_id = 1 AND created_at > '2024-01-01' ORDER BY name
CREATE INDEX idx_example ON files (org_id, created_at, name);
--                                 equality  range       sort
```

The leading equality column narrows the B-tree to a small subtree. The range column further narrows within that subtree. The sort column avoids a separate Sort node.

**2. Partial indexes for soft-delete patterns**

```sql
-- Only index non-deleted rows (smaller index, faster scans)
CREATE INDEX idx_folders_active ON folders (user_id, organization_id)
  WHERE deleted_at IS NULL;
```

This is exactly what your codebase does with `idx_folders_user_org_active`.

**3. Covering indexes to enable Index Only Scan**

```sql
CREATE INDEX idx_files_folder_uuid ON uploaded_files (folder_id) INCLUDE (uuid);
-- Now "SELECT uuid FROM uploaded_files WHERE folder_id = X" is Index Only Scan
```

**4. Don't over-index**

Every index:
- Slows down INSERT/UPDATE/DELETE (must update all indexes)
- Consumes disk space and buffer cache
- Must be VACUUMed

Rule of thumb: if an index isn't used by any query (check `pg_stat_user_indexes.idx_scan`), drop it.

### Composite Index Column Order Matters

```sql
CREATE INDEX idx_ab ON t (a, b);
```

This index can serve:
- `WHERE a = 1` (yes — leading column)
- `WHERE a = 1 AND b = 2` (yes — both columns)
- `WHERE b = 2` (NO — can't skip leading column, needs full scan)
- `WHERE a = 1 ORDER BY b` (yes — b is sorted within each a value)

### B-tree Limitations

B-trees work for: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'prefix%'`

B-trees do NOT work for: `LIKE '%substring%'`, `@>` (containment), geometric operations, full-text search. Use GiST, GIN, or specialized indexes for these.

---

## 11. Common Pitfalls & Anti-Patterns

### 1. N+1 Query Pattern (disguised in SQL)

```sql
-- Bad: correlated subquery executes once per row in the outer query
SELECT f.name,
       (SELECT count(*) FROM uploaded_files uf WHERE uf.folder_id = f.id)
FROM folders f;
```

The planner *might* decorrelate this into a join, but often it runs the subquery N times.

### 2. Implicit Casts Defeating Indexes

```sql
-- If folder_id is integer but you pass a string:
WHERE folder_id = '123'
-- PostgreSQL casts '123' to integer — still uses the index (usually safe)

-- But if uuid column is UUID type and you compare with text:
WHERE uuid = '...'::text
-- This may NOT use the UUID index. Always match types exactly.
```

### 3. OR Conditions Preventing Index Use

```sql
-- This often becomes a Seq Scan:
WHERE user_id = 1 OR organization_id = 5

-- Fix: rewrite as UNION ALL
SELECT * FROM t WHERE user_id = 1
UNION ALL
SELECT * FROM t WHERE organization_id = 5 AND user_id != 1
```

Or PostgreSQL may use BitmapOr on two separate indexes — check the plan.

### 4. Functions on Indexed Columns

```sql
-- Cannot use index on created_at:
WHERE EXTRACT(year FROM created_at) = 2024

-- Fix: use range condition
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- Or create an expression index:
CREATE INDEX idx_year ON t (EXTRACT(year FROM created_at));
```

### 5. Large IN Lists

```sql
WHERE id IN (1, 2, 3, ..., 10000)
```

Above ~100 values, consider using `= ANY(ARRAY[...])` or a temporary table with a join. The planner may switch strategies.

### 6. Unnecessary DISTINCT / ORDER BY

Adding `DISTINCT` or `ORDER BY` on large result sets forces a Sort or HashAggregate. Only use when actually needed.

---

## 12. Tuning Knobs That Matter

### Session/Query Level

```sql
SET LOCAL work_mem = '256MB';          -- more memory for sorts/hashes (per-operation, not per-query!)
SET LOCAL random_page_cost = 1.1;      -- SSD-friendly
SET LOCAL enable_seqscan = off;        -- force index use (debugging only, NEVER in production)
SET LOCAL jit = off;                   -- disable JIT if it's adding overhead on simple queries
```

### Table Level

```sql
ANALYZE tablename;                     -- refresh statistics (run after bulk loads)
VACUUM ANALYZE tablename;              -- reclaim dead tuples + refresh stats
ALTER TABLE t SET (fillfactor = 80);   -- leave room for HOT updates
```

### Statistics Targets

```sql
-- Increase statistics granularity for skewed columns
ALTER TABLE folders ALTER COLUMN organization_id SET STATISTICS 1000;
ANALYZE folders;
```

Default is 100 most-common values. For columns with many distinct values and skewed distributions, increase to 500-1000. This gives the planner better selectivity estimates.

### Monitoring Queries

```sql
-- Find slow queries (requires pg_stat_statements extension)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find tables that need VACUUM
SELECT relname, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Cache hit ratio (should be >99%)
SELECT
  sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

---

## 13. Quick Reference Cheat Sheet

### Scan Type Decision Tree

```
Is there a useful index?
├── No → Seq Scan
└── Yes
    ├── Need very few rows (<5%)? → Index Scan
    ├── Need moderate rows (5-25%)?
    │   ├── Multiple indexes combinable? → BitmapAnd/Or + Bitmap Heap Scan
    │   └── Single index → Bitmap Index Scan + Bitmap Heap Scan
    ├── All needed columns in index? → Index Only Scan
    └── Need most rows (>25%)? → Seq Scan (even with index available)
```

### Join Strategy Decision Tree

```
Is one side very small + other has index?
├── Yes → Nested Loop
└── No
    ├── Both sides sorted on join key? → Merge Join
    └── Neither sorted → Hash Join
```

### Red Flags in EXPLAIN ANALYZE

| What you see | What it means | Fix |
|---|---|---|
| `Seq Scan` on large table with filter | Missing index | Add appropriate index |
| `loops=50000` on inner Nested Loop | Outer side is huge | Check if Hash Join is possible |
| `rows=1` estimated, `rows=50000` actual | Stale stats or correlated columns | `ANALYZE table` or increase statistics target |
| `Sort Method: external merge Disk` | Sort spilled to disk | Increase `work_mem` |
| `Heap Fetches: 49000` on Index Only Scan | VACUUM is behind | Run `VACUUM` on the table |
| `Buffers: shared read=10000` on repeated query | Cache is cold or too small | Check `shared_buffers` config |
| `Filter: ... Rows Removed by Filter: 99000` | Index returns too many rows, post-filtered | Need a more selective index |

---

## Further Reading

- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [Use The Index, Luke (indexing tutorial)](https://use-the-index-luke.com/)
- [pgMustard (EXPLAIN plan visualizer)](https://www.pgmustard.com/)
- [Dalibo explain.dalibo.com (free plan visualizer)](https://explain.dalibo.com/)
- [Markus Winand — SQL Performance Explained](https://sql-performance-explained.com/)
