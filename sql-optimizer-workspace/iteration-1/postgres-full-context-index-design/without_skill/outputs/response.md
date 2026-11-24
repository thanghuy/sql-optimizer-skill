# Index design for the products listing query

## TL;DR

Create one partial composite index that matches the query's **equality → sort → filter** shape:

```sql
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated_price
    ON products (tenant_id, updated_at DESC, price)
    WHERE is_active;
```

Then confirm the plan is `Limit -> Index Scan using idx_products_tenant_active_updated_price` with **no Sort node** and only a few dozen buffers touched. After that, look at whether `idx_products_tenant` and `idx_products_price` are still used and drop them if they aren't.

The reasoning, the alternatives I rejected, the rollout steps and how to verify it follow below.

---

## Why the current setup is slow

The only useful indexes are `idx_products_tenant` and `idx_products_price`, so Postgres has two choices, and both are bad:

1. Fetch every row for the tenant through `idx_products_tenant`, filter `is_active` and `price` on the heap, then sort by `updated_at` (a top-N heapsort) to get 24 rows. For your biggest tenant that means about 6M heap tuples visited and about 1.5M rows surviving the filter (6M × 85% × 30%) before the sort.
2. BitmapAnd of the tenant and price indexes, then a heap scan and the same sort. The work is the same order of magnitude.

Either way the cost grows with **tenant size** rather than with `LIMIT 24`. That's why p95 (driven by the large tenants) is around 900ms.

## The index, and why the column order matters

```
(tenant_id, updated_at DESC, price) WHERE is_active
```

| Part | Purpose |
|---|---|
| `tenant_id` first | Equality predicate. The scan jumps straight to this tenant's slice of the index. |
| `updated_at DESC` second | The index already returns rows in `ORDER BY updated_at DESC` order, so there's **no sort**, and the `LIMIT 24` can stop the scan early. (A plain ASC column would also work because btrees scan backward, but `DESC` documents the intent.) |
| `price` third (key column, not `INCLUDE`) | Postgres can't use it as a range *boundary* because it sits after the sort column. It still shows up as an `Index Cond`, though, and btree checks it **on each index tuple before visiting the heap**. Rows outside the price range are skipped inside the index, so you only pay heap fetches for the 24 rows you return. With `INCLUDE (price)` and a plain Index Scan, PG 16 would check price on the heap instead, which is worse. |
| `WHERE is_active` | The query has the literal `is_active = true`, so the planner can match the partial index, and this holds for generic plans too. It removes the ~15% inactive rows and keeps the index smaller. |

**Expected work per call:** with ~30% of active rows matching the price range, the scan reads about 24 / 0.30 ≈ 80 index tuples, usually on 1–3 leaf pages, plus 24 heap fetches. That's a handful of buffers whether the tenant has 100 products or 6M, so sub-millisecond to a few ms when the data is cached, well under your 50ms target.

### Alternatives I rejected

- **`(tenant_id, price, updated_at)`**: this is the "obvious" order: equality, then range. The price range becomes a tight boundary, but the rows come out in price order, so Postgres still has to sort about 1.5M rows for the big tenant. The rule of thumb is that an ORDER BY + LIMIT column has to come **before** any range column, or you lose the early stop.
- **`(tenant_id, updated_at DESC) INCLUDE (price, title)` for index-only scans**: on paper this avoids the 24 heap fetches. But at ~150 updates/sec the visibility map on hot pages is often not all-visible, so the "index-only" scan ends up doing heap fetches anyway. Adding `title` (unbounded text) also makes the index much larger and slows every write. 24 random heap fetches on a hot, cached table are cheap. Only revisit this if verification shows heap I/O is the remaining bottleneck.
- **Separate single-column `updated_at` index**: this would make the planner walk the global `updated_at` order and filter out the other 1,799 tenants. It's terrible for small tenants.

## Write-side impact (150 updates/sec + 20 inserts/sec)

- Updates mostly change `price` and `updated_at`. `price` is already indexed (`idx_products_price`), so those updates are **already non-HOT**. The new index adds one more index insertion per update and insert, about 170 index writes/sec. That's modest for RDS.
- Expect some bloat on the new index, because `updated_at` changes move entries to the "hot" end of the index. Normal autovacuum handles this at these rates. Keep an eye on index size over the first weeks.
- You can **win write capacity back** by dropping indexes you no longer need (see below).

## Rollout steps

1. **Grab a baseline first** (see "Verification" below), so you have before and after numbers.
2. Build it online. `CONCURRENTLY` does not block reads or writes, but it **can't run inside a transaction block**, so run it outside any migration tool that wraps statements in `BEGIN`:
   ```sql
   SET maintenance_work_mem = '1GB';   -- session-only; size to your instance class
   CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated_price
       ON products (tenant_id, updated_at DESC, price)
       WHERE is_active;
   ```
   On 42M rows, expect minutes, not seconds. Watch it with:
   ```sql
   SELECT phase, blocks_done, blocks_total, tuples_done, tuples_total
   FROM pg_stat_progress_create_index;
   ```
3. **Check that it is valid.** A failed or cancelled concurrent build leaves an INVALID index that still costs writes:
   ```sql
   SELECT indexrelid::regclass, indisvalid, indisready
   FROM pg_index WHERE indrelid = 'products'::regclass;
   ```
   If it's invalid, `DROP INDEX CONCURRENTLY` it and retry.
4. `ANALYZE products;` (cheap, and it refreshes the planner's stats).
5. If you have read replicas, the index replicates automatically. Verify there too if the endpoint reads from replicas.

## Verification

### 1. The plan shape (custom plan)

Use real values for your **largest tenant**, plus a small tenant:

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = <big_tenant_id> AND is_active = true
  AND price BETWEEN 10 AND 50
ORDER BY updated_at DESC
LIMIT 24;
```

What a good result looks like:
```
Limit  (actual rows=24 ...)
  -> Index Scan using idx_products_tenant_active_updated_price on products
       Index Cond: ((tenant_id = ...) AND (price >= ...) AND (price <= ...))
       Buffers: shared hit=~30-100
```
- **No `Sort` node** and no `Bitmap Heap Scan`.
- `Buffers` in the tens, not tens of thousands.
- `Execution Time` in the low milliseconds.

### 2. The generic plan (important, because it's a prepared statement)

Your Go driver (pgx and friends) prepares the statement, so after 5 executions Postgres may switch to a **generic plan** that ignores the actual `$1..$3` values. That plan is the one production actually runs, so check it as well. PG 16 lets you do that directly:

```sql
EXPLAIN (GENERIC_PLAN)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC
LIMIT 24;
```

To also get actual timings for the generic plan:
```sql
PREPARE q(int, numeric, numeric) AS
  SELECT id, title, price, updated_at FROM products
  WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
  ORDER BY updated_at DESC LIMIT 24;
SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(<big_tenant_id>, 10, 50);
RESET plan_cache_mode;
DEALLOCATE q;
```
It should show the same `Limit -> Index Scan using idx_products_tenant_active_updated_price` shape. This index is the right choice for every parameter value in the normal range, so generic and custom plans should agree.

### 3. Edge cases to test

- **Very narrow or empty price ranges on the big tenant** (for example a range that matches 0.1% of rows, or none at all). This is the weak spot of the design: the scan walks `updated_at` order until it finds 24 matches. If nothing matches, it reads that tenant's whole slice of the index (about 5M entries), which can take hundreds of ms.
  - If your UI allows filters like that and they matter, add a second index, `CREATE INDEX CONCURRENTLY ... ON products (tenant_id, price) WHERE is_active;`. Note that the planner can only pick between the two per call with **custom** plans. Generic plans pick one for everyone. In that case, consider running this statement with `plan_cache_mode = force_custom_plan` (set per role, or per session or connection in the pool's `AfterConnect`), or use pgx's `QueryExecModeExec`/`SimpleProtocol` for this query. Planning cost is small next to a 50ms budget.
  - If typical filters really are ~30% selective, skip this until monitoring shows a slow tail.
- **Small tenant.** It should also be a tiny index range scan, and nothing should regress.

### 4. Production confirmation

- **The index is being used:**
  ```sql
  SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch,
         pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes WHERE relname = 'products';
  ```
  `idx_scan` on the new index should climb at roughly your call rate (~1,200/s at peak).
- **Latency:** in `pg_stat_statements`, compare `mean_exec_time`, `stddev_exec_time` and `shared_blks_hit + shared_blks_read` per call for this query before and after (`SELECT pg_stat_statements_reset();` after deploying makes the comparison clean). For **p95**, use your Go service or APM histograms, since pg_stat_statements doesn't give percentiles.
- **Catch slow outliers:** enable `auto_explain` through the RDS parameter group (`shared_preload_libraries` or `session_preload_libraries` = `auto_explain`, `auto_explain.log_min_duration = 50ms`, `auto_explain.log_analyze = on`, `auto_explain.log_buffers = on`). That logs the exact plan for any call that still misses the target, which is how you'd spot the narrow-range case above.
- **Write side:** in `pg_stat_user_tables`, check `n_tup_upd` against `n_tup_hot_upd`, and watch the new index's size and RDS write IOPS and WAL volume for a few days.

## Clean up the old indexes (after verification)

These two indexes cost writes on every update, so check whether anything still uses them:

```sql
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes
WHERE relname = 'products' AND indexrelname IN ('idx_products_tenant', 'idx_products_price');
```

Note the counts, wait through a full business cycle (including batch jobs and reports), and check again.

- **`idx_products_tenant`**: the new index covers `tenant_id` lookups, but **only for active rows**. Queries like `WHERE tenant_id = $1` that need inactive products, and foreign-key-style lookups, still need it. Drop it only if nothing else needs it.
- **`idx_products_price`**: a global price index without tenant scoping is rarely useful in a multi-tenant table. If `idx_scan` doesn't move, drop it. That also gives `price`-only updates a chance to become HOT again if no other index covers `price`, but the new index does cover it, so the bigger gain is simply one less index to maintain.

```sql
DROP INDEX CONCURRENTLY idx_products_price;   -- only after confirming it's unused
```

## Optional: make the ordering deterministic

`ORDER BY updated_at DESC LIMIT 24` returns an arbitrary pick when timestamps tie. That matters if you paginate, because rows can repeat or go missing between pages. If you paginate, use keyset pagination with a tie-breaker and build the index to match instead:

```sql
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated_id_price
    ON products (tenant_id, updated_at DESC, id DESC, price)
    WHERE is_active;

-- page N+1:
... WHERE tenant_id = $1 AND is_active AND price BETWEEN $2 AND $3
      AND (updated_at, id) < ($4, $5)
    ORDER BY updated_at DESC, id DESC
    LIMIT 24;
```

Avoid `OFFSET`-based paging on this endpoint. Deep pages would scan `OFFSET + 24` matching rows.

---

**One thing that would help:** if you can share the current `EXPLAIN (ANALYZE, BUFFERS)` output for a large-tenant call, I can confirm the diagnosis above. The recommendation doesn't depend on it, though: the query's shape (equality + ORDER BY + LIMIT + a non-selective range filter) points to this index whatever the current plan is.
