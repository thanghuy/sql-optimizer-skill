## Short answer

Create one partial composite index. Its key order matches your `WHERE` and `ORDER BY`:

```sql
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated
    ON products (tenant_id, updated_at DESC, price)
    WHERE is_active;
```

Then check the plan with `EXPLAIN (ANALYZE, BUFFERS)` against your largest tenant, under both custom and generic plans (steps below). Once the index has proven itself, think about dropping `idx_products_price`.

## Why this index

**What's slow today.** The planner has two choices, and both are bad:

- It can use `idx_products_tenant` and fetch every row for the tenant. For the 6M-product tenant, that means about 5.1M active rows, of which about 1.5M fall in the price range. It reads them all from the heap, sorts them, and keeps 24.
- It can use `idx_products_price` and filter by tenant, which is just as wasteful.

Either way it reads millions of tuples to return 24. That explains the ~900 ms p95.

**What the new index does.** Each part of the index has a job:

| Part | Purpose |
|---|---|
| `tenant_id` (1st key) | Equality filter. The scan stays inside one tenant's slice of the index. |
| `updated_at DESC` (2nd key) | The index is already in `ORDER BY` order. There is no Sort node, and the `LIMIT 24` can stop the scan early. |
| `price` (3rd key, **not** `INCLUDE`) | Because `price` is a key column, `price BETWEEN $2 AND $3` becomes part of the **Index Cond**. It is checked against the index entries themselves, so the heap is only read for rows that match. If `price` were an `INCLUDE` column, a plain Index Scan would check it as a Filter after each heap fetch. |
| `WHERE is_active` | Leaves out the ~15% of rows that are inactive, so the index is smaller and nothing is spent on those rows. Your query has the literal `is_active = true`, so the planner can prove the partial index applies even in a generic plan. |

**Expected cost.** With ~30% price selectivity, the scan walks about 24 / 0.3 ≈ 80 index entries in the tenant's newest-first order and makes about 24 heap fetches. That's a few dozen buffers and should take well under 1–2 ms, even for the biggest tenant. It also no longer depends on tenant size.

**Why the order is not `(tenant_id, price, updated_at)`.** Putting `price` second would turn the price filter into a range seek. But the rows would then come out in price order, so the planner would have to read every matching row (~1.5M for the big tenant) and sort them. At 30% selectivity, "walk in `updated_at` order and stop after 24" is far cheaper.

**Why there's no covering index (`INCLUDE (title)`) for an Index Only Scan.**

- `title` is unbounded text, so including it would make the index much larger.
- With ~150 updates/s spread across the table, many heap pages won't be marked all-visible at any given moment. The "index-only" scan would still visit the heap for many rows.
- About 24 heap fetches is already cheap. It isn't worth the size or the write cost.

## Write-side impact (150 updates/s, 20 inserts/s)

- **HOT updates.** You already lose HOT updates on price changes, because `idx_products_price` indexes `price`. With `updated_at` in the new index, essentially every update becomes a non-HOT update that writes an entry in **every** index on the table. That is about 170 extra index insertions per second for this index, which RDS handles easily. The thing to watch is bloat, not throughput. Keep autovacuum healthy on this table, for example by lowering `autovacuum_vacuum_scale_factor` for the table (around 0.02–0.05) so vacuum doesn't wait for millions of dead tuples on a 42M-row table.
- **Rows that change state.** When `is_active` flips, the row enters or leaves the partial index on the next update. Nothing extra is needed for that.
- **Size.** The index will have about 36M entries, roughly 35–40 bytes each, so expect around 1.3–1.8 GB. Check the real number with `pg_relation_size` after the build.

## Rollout

1. **Build it online.** Run this in its own session. `CONCURRENTLY` cannot run inside a transaction block, and some migration tools wrap statements in one:
   ```sql
   SET maintenance_work_mem = '1GB';   -- session-level; size to your instance class
   CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated
       ON products (tenant_id, updated_at DESC, price)
       WHERE is_active;
   ```
   Expect it to take several minutes to tens of minutes. It waits for older transactions to finish, so long-running transactions will block it.

2. **Make sure the build didn't fail.** A failed `CONCURRENTLY` build leaves behind an INVALID index that still adds write cost:
   ```sql
   SELECT c.relname, i.indisvalid, i.indisready
   FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid
   WHERE c.relname = 'idx_products_tenant_active_updated';
   -- If indisvalid = false:
   --   DROP INDEX CONCURRENTLY idx_products_tenant_active_updated;  then retry
   ```

3. **Refresh statistics:** `ANALYZE products;`

## How to verify it worked

### 1. Plan shape for the worst tenant (custom plan)

Look up the largest tenant's id (`SELECT tenant_id, count(*) FROM products GROUP BY 1 ORDER BY 2 DESC LIMIT 5;`). Then run the query with literal values:

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = <big_tenant> AND is_active = true
  AND price BETWEEN 20 AND 80          -- a typical ~30% range
ORDER BY updated_at DESC
LIMIT 24;
```

What a good plan looks like:

```
Limit (actual rows=24 ...)
  -> Index Scan using idx_products_tenant_active_updated on products
       Index Cond: ((tenant_id = ...) AND (price >= ...) AND (price <= ...))
       Buffers: shared hit=~30-100
```

Check these points:

- **There is no `Sort` node.** If there is one, the index isn't giving the order.
- **`price` appears in `Index Cond`,** not in `Filter:`.
- **There is no `Rows Removed by Filter`,** or only a tiny number.
- **Buffers are in the tens to low hundreds,** not in the hundreds of thousands.
- **Execution time is in the low single-digit milliseconds** or below.

Repeat with a small tenant, a very wide price range, and a very narrow price range.

### 2. Plan shape for the prepared statement (generic plan)

This step matters. Your Go driver (pgx by default) runs this as a prepared statement. After 5 executions, PostgreSQL may switch to a **generic plan** that doesn't know the parameter values. That's a common reason a query is fast in psql but slow from the app. PG 16 can show the generic plan directly:

```sql
EXPLAIN (GENERIC_PLAN)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC
LIMIT 24;
```

You can also run the real executions under the generic plan:

```sql
SET plan_cache_mode = force_generic_plan;
PREPARE q(int, numeric, numeric) AS
  SELECT id, title, price, updated_at FROM products
  WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
  ORDER BY updated_at DESC LIMIT 24;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(<big_tenant>, 20, 80);
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q(<small_tenant>, 20, 80);
DEALLOCATE q; RESET plan_cache_mode;
```

The generic plan should be the same Index Scan with no Sort.

If it isn't, set `plan_cache_mode = force_custom_plan` for the app's role:

```sql
ALTER ROLE app_user SET plan_cache_mode = force_custom_plan;
```

You can also set it per connection via pgx's `RuntimeParams`. The cost is that every execution gets planned, which adds a fraction of a millisecond.

### 3. In production

- **Index usage.** `idx_scan` should climb at about the endpoint's call rate:
  ```sql
  SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch,
         pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes WHERE relname = 'products';
  ```
- **Query stats.** Reset or snapshot `pg_stat_statements` before the change, then compare `mean_exec_time`, `max_exec_time` and `shared_blks_hit + shared_blks_read` per call for this query. Blocks per call should drop by several orders of magnitude.
- **Latency.** Measure the p95 itself in your service metrics or in RDS Performance Insights. `pg_stat_statements` doesn't report percentiles.
- **Plan drift (optional).** Enable `auto_explain` with `auto_explain.log_min_duration = '50ms'` and `log_analyze = on`. Any execution that slips back to a bad plan, such as a generic-plan flip or a bloated index, will be logged with its plan.
- **Write side.** Over the next days, watch `n_tup_hot_upd / n_tup_upd` and `n_dead_tup` in `pg_stat_user_tables`, autovacuum frequency on `products`, and the index size.

## Cleanup after it's proven (a week or so of stable stats)

**`idx_products_price`.** It's probably redundant once this query uses the new index, and it costs a write on every price update. Before dropping it, check two things:

- `idx_scan` for it stays flat in `pg_stat_user_indexes`. On RDS, if you have read replicas, check them too: replica index usage isn't reflected on the primary.
- No other query, such as reports or admin tools, relies on it.

Then drop it with `DROP INDEX CONCURRENTLY idx_products_price;`.

**`idx_products_tenant`.** Don't drop it blindly. The new index is partial, so queries that filter on tenant but not on `is_active = true` (inactive-product views, counts across all rows) still need it or something like it.

## Caveats and possible follow-ups

- **Very narrow price ranges.** The scan still walks a tenant's index entries newest-first until it finds 24 matches. Because the price check happens inside the index, that is cheap per entry. But if a range matches almost nothing (say 0.01%) for the 6M-product tenant, it could walk millions of entries, roughly 100+ ms. If narrow ranges are common in practice, consider:
  - a second index `ON products (tenant_id, price) WHERE is_active`, plus `force_custom_plan`, so the planner can pick per call based on the actual range; or
  - capping or validating the range in the API.
- **Tie-breaking and pagination.** `ORDER BY updated_at DESC` isn't deterministic when timestamps tie. If this endpoint pages through results, use `ORDER BY updated_at DESC, id DESC` with keyset pagination (`AND (updated_at, id) < ($4, $5)`) rather than `OFFSET`. That would justify an index of `(tenant_id, updated_at DESC, id DESC, price) WHERE is_active`.
- **The target.** Index work alone should take the query from ~900 ms to the low-millisecond range, well under 50 ms. If p95 is still high afterwards, the plan above will show it. The cause is then likely outside the query plan: connection pool waits, network, or I/O on a cold cache (look for high `shared read` instead of `hit` in BUFFERS).
