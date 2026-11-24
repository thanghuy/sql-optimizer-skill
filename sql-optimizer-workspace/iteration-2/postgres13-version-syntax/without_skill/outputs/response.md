## Short answer

Add a **partial composite index** that matches the filter and the sort order:

```sql
CREATE INDEX CONCURRENTLY idx_orders_open_region_created
    ON orders (region, created_at DESC)
    WHERE status IN ('NEW','PAID');

ANALYZE orders;
```

With this index Postgres can jump straight to `region = $1`, read entries newest-first, stop after 50 rows, and skip the sort. Each execution touches about 50 index entries and 50 heap pages, which is fine at 400 QPS. Your current single-column indexes can't do that. `idx_orders_region` matches about 1/12 of the table (≈6.7M rows). `idx_orders_created` either scans the last day across all regions and filters, or walks backwards through the whole table.

Because it is partial, the index holds only the ≈3% of rows that are NEW/PAID (≈2.4M of 80M). It stays small and mostly in cache.

## Why this shape, and the trade-offs to know about

1. **Column order `(region, created_at DESC)`.** Equality column first, then the range/sort column. That gives a single ordered range scan with no Sort node. `DESC` isn't strictly needed, since a B-tree can be scanned backwards, but it documents intent and costs nothing.
2. **The partial predicate has to match the query text.** The planner only uses a partial index when it can prove the query's `WHERE` implies the index's `WHERE`. Your statuses are SQL literals, so the proof works, including for generic prepared-statement plans. **If anyone changes the Java code to bind statuses as parameters (`status IN (?, ?)` or `status = ANY(?)`), the generic plan can no longer use this index.** Keep them as literals, or use the non-partial alternative below.
3. **Status updates and HOT.** Putting `status` in an index's predicate makes `status` an "indexed column". From then on, updating it can never be a HOT update: each status change writes new entries into *every* index on `orders`. You do 300 inserts/s with 1-2 status changes per order, so that's roughly 300-600 non-HOT updates/s. In practice this often costs little, because an append-heavy table at the default `fillfactor = 100` rarely has room on the page for HOT anyway. But watch index bloat and autovacuum on `orders` after the change. For comparison, check the current HOT rate first: `SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'orders';`
4. **Non-partial alternative.** `CREATE INDEX CONCURRENTLY ... ON orders (region, created_at DESC);` doesn't index `status` (status updates stay HOT-eligible) and doesn't depend on literal statuses. It works well **if most orders from the last day are still NEW/PAID**, which is likely, since the 3% is across all history. In that case the scan finds 50 matches almost immediately. The cost is a much larger index (80M entries) maintained on every insert. If a large share of recent orders leave NEW/PAID quickly, the partial index wins clearly. I'd start with the partial index.
5. **Don't add INCLUDE columns** to get index-only scans. The rows this query returns are recent and frequently updated, so their pages are rarely all-visible in the visibility map. Postgres would visit the heap anyway, and you'd only make the index bigger.
6. **Cleanup.** `idx_orders_region` (12 distinct values) is almost certainly useless, and the new index covers its use case. Check `pg_stat_user_indexes.idx_scan` for it over a representative period, then `DROP INDEX CONCURRENTLY idx_orders_region;`. Dropping it removes an index that every insert and non-HOT update has to maintain. Keep `idx_orders_created` unless the same check shows nothing uses it.

`CREATE INDEX CONCURRENTLY` doesn't block writes, but it can't run inside a transaction block and takes longer. If it fails, it leaves an `INVALID` index that you must drop and recreate. Check with `\d orders` or `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;`.

## Seeing the plan the app actually gets

psql can show you something different because of how pgjdbc prepares statements:

- **pgjdbc switches to a server-side prepared statement** after `prepareThreshold` executions (default **5**) of the same `PreparedStatement`. After that, the server's plan cache can switch from a *custom plan* (planned with the real `region` value) to a **generic plan** (planned for `region = $1`, with no value). This happens with `plan_cache_mode = auto`, the default. A plain `EXPLAIN SELECT ... WHERE region = 'EU'` in psql always shows a custom plan.
- **Parameter types differ.** `setString()` sends the value as `varchar` by default (`stringtype` connection property). Usually harmless, but against some column types it can force casts that prevent index use.
- **Session settings differ.** The app's role/database settings (`ALTER ROLE ... SET`, `ALTER DATABASE ... SET`), `work_mem`, `search_path`, and **`TimeZone`** are all in play. pgjdbc sets the session time zone to the JVM's zone. If `created_at` is `timestamp without time zone`, `now() - interval '1 day'` then means a different moment than it does in your psql session.

Three ways to see the real plan, from most to least authoritative.

### 1. auto_explain: logs the plan the app actually executed

The `auto_explain` extension ships with Postgres 13 (it's in contrib, nothing to install). It logs the plan that was actually executed, generic or custom, with the app's real settings. At 400 QPS, use a duration threshold and/or sampling so you don't flood the log:

```conf
# postgresql.conf  (shared_preload_libraries change needs a restart)
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = '50ms'   # only slow executions
auto_explain.log_analyze = on            # real row counts/timings
auto_explain.log_buffers = on
auto_explain.log_timing = off            # cheaper; row counts still reported
auto_explain.log_nested_statements = off
auto_explain.sample_rate = 0.05          # 5% of statements
auto_explain.log_format = 'text'
```

If you can't restart, target just the application role. Superuser is required, and the setting takes effect for **new** connections, so recycle the connection pool:

```sql
ALTER ROLE app_user SET session_preload_libraries = 'auto_explain';
ALTER ROLE app_user SET auto_explain.log_min_duration = '50ms';
ALTER ROLE app_user SET auto_explain.log_analyze = on;
ALTER ROLE app_user SET auto_explain.sample_rate = 0.05;
```

In the log, a generic plan shows `$1` where the region value would be (for example `Index Cond: ((region)::text = $1)`). A custom plan shows the literal. `log_analyze` adds per-execution overhead, so keep the sample rate low and turn it off again once you're done.

### 2. Reproduce the prepared-statement path in psql

This mimics what pgjdbc does. Use the same role as the app, and set the session to match the app's settings, at least the time zone:

```sql
SET ROLE app_user;                 -- or log in as it, so role-level settings apply
SET TimeZone = 'UTC';              -- whatever the JVM zone is

PREPARE q(varchar) AS              -- varchar = what setString() sends
SELECT id, status, total
FROM orders
WHERE region = $1 AND status IN ('NEW','PAID') AND created_at > now() - interval '1 day'
ORDER BY created_at DESC
LIMIT 50;

-- Force the generic plan the app may end up with:
SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q('EU');

-- Compare with the custom plan:
SET plan_cache_mode = force_custom_plan;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE q('EU');

DEALLOCATE q;
```

Without forcing, you can run `EXECUTE q('EU');` six or more times and then `EXPLAIN EXECUTE q('EU');`. That shows whichever plan the auto mode settled on. (`EXPLAIN (GENERIC_PLAN)` only exists from Postgres 16, so on 13 use the `PREPARE`/`plan_cache_mode` approach.) If the generic and custom plans both use the new index, you're done. With 12 similarly sized regions they should.

If the generic plan turns out bad for some skewed region, you can pin the behaviour for the app role with `ALTER ROLE app_user SET plan_cache_mode = force_custom_plan;`. You'd pay planning cost on every execution, so only do this if you actually see the problem. Alternatively, set `prepareThreshold=0` in the JDBC URL to stop server-side preparing for those connections.

### 3. Confirm with statistics after deploying

```sql
-- Is the new index being used, and the old ones not?
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes WHERE relname = 'orders';

-- Latency of the app's statement (pg_stat_statements; on PG13, set
-- pg_stat_statements.track_planning = on to also see plan time)
SELECT calls, mean_exec_time, rows / calls AS avg_rows, shared_blks_hit, shared_blks_read
FROM pg_stat_statements
WHERE query LIKE '%FROM orders%ORDER BY created_at DESC%LIMIT%';
```

`pg_stat_statements` doesn't store plans, but before and after the index you should see `mean_exec_time` drop to sub-millisecond or low milliseconds, and shared blocks per call drop sharply.

## Checklist

1. Check the HOT rate on `orders` now, for comparison.
2. `CREATE INDEX CONCURRENTLY ... (region, created_at DESC) WHERE status IN ('NEW','PAID')`, then `ANALYZE orders`.
3. Check both the generic and custom plans with `PREPARE` + `plan_cache_mode` as the app role.
4. Turn on sampled `auto_explain` for the app role to confirm the plan in production, then turn it off.
5. After a representative period, drop `idx_orders_region` if `idx_scan` shows nothing else uses it. Watch bloat and autovacuum on `orders`.
6. Keep the status values as SQL literals in the Java code. Otherwise the generic plan can't use the partial index.
