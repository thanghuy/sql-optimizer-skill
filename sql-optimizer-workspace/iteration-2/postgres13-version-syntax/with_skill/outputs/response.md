Everything below is checked against the **PostgreSQL 13** docs (`/docs/13/`). Two features you may see recommended elsewhere are **not available on 13**: `EXPLAIN (GENERIC_PLAN)` (added in 16) and B-tree skip scan (added in 18). So I won't rely on either. (One side note: community support for 13 ended in November 2025, so it no longer gets security fixes. That doesn't change the advice below.)

## Diagnosis

- **What the planner can do today.** `idx_orders_region` is almost useless here: 12 regions means about 6.7M rows per value. `idx_orders_created` can walk the last day (about 240k rows across all regions), but `region` and `status` have to be checked on every heap row. Either way, Postgres reads tens of thousands of rows (or sorts them) to return 50, and it does that 400 times per second.
- **Why `(region, status, created_at)` isn't the fix on 13.** `status IN ('NEW','PAID')` has two values. The index can seek both of them, but the rows come back as "all NEW, then all PAID", not in global `created_at DESC` order. So PG 13 must read every matching row in the last day and then run a top-N `Sort` before the `LIMIT` applies. Better handling of multi-value `= ANY` in B-tree scans arrived in 17, not 13.
- **What helps.** The status list is made of **literals**, not bind parameters. That means you can move it into a **partial index predicate**. Then the index has one equality column followed by the sort column, which is a clean ESR match.

## Recommendation

### 1. Partial index (main recommendation)

```sql
-- Run outside a transaction block; it does not block inserts or updates
CREATE INDEX CONCURRENTLY idx_orders_open_region_created
    ON orders (region, created_at DESC)
    WHERE status IN ('NEW', 'PAID');
```

**ESR mapping**
- **E** = `region`. The `status` equality is folded into the `WHERE` clause of the index, so it costs no key space.
- **S** = `created_at DESC`, matching the `ORDER BY`.
- **R** = `created_at > now() - interval '1 day'`. This is the same column as S, so it only sets where the backward scan stops. Order is kept, and there is no conflict between sort and range.

**Expected plan:** `Limit -> Index Scan using idx_orders_open_region_created`, with `Index Cond: (region = $1) AND (created_at > (now() - '1 day'))`. There is no `Sort` node and no `Rows Removed by Filter`. It reads 50 index entries plus 50 heap rows and stops, which typically means tens of buffers and well under 1 ms. The worst case is a region with fewer than 50 open orders in the last day: then it reads just those rows and stops at the day boundary.

**Why this works with JDBC prepared statements.** The planner can only use a partial index when it can prove the query's `WHERE` clause implies the index predicate. Your `status IN ('NEW','PAID')` is a constant in the SQL text, so that proof holds for **generic plans** too. `region` stays a `$1` parameter, and `now()` is evaluated at run time. **Caveat:** if the app ever binds the statuses as parameters (`status IN (?, ?)` or `= ANY(?)`), the generic plan can no longer use this index. Keep them as literals, or tell me and I'll give you a non-partial design.

**Why there is no `INCLUDE (id, status, total)` (INCLUDE exists since 11, so 13 supports it).** An index-only scan only skips the heap for pages marked all-visible in the visibility map. Your target rows are less than a day old and were just inserted or updated, so their pages are mostly *not* all-visible yet. You would still see about 50 heap fetches. With only 50 rows per call, the lookup costs little, and covering columns would make the index bigger and pull `total` into HOT considerations. Add `INCLUDE` later only if the plan shows heap fetches really hurt.

**Cost**
- **Size.** NEW+PAID is about 3% of 80M, so roughly 2.4M entries. Each entry holds `region` plus an 8-byte timestamptz, so expect around 70–120 MB. That is far smaller than a full index on 80M rows.
- **Writes.** Every insert of a NEW order adds one entry (about 300/s). Each status change into or out of NEW/PAID adds or removes an entry.
- **HOT updates (the real trade-off).** Because `status` now appears in an index predicate, an `UPDATE` that changes `status` can no longer be a HOT update. Each of the 1–2 status updates per order will then write new entries into *all* indexes on `orders` (pkey, region, created, and the new one), which means more WAL and more bloat. Check how many updates are HOT today:
  ```sql
  SELECT n_tup_upd, n_tup_hot_upd,
         round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 1) AS hot_pct
  FROM pg_stat_user_tables WHERE relname = 'orders';
  ```
  If `hot_pct` is already low (common with the default fillfactor 100 on an append-heavy table), you lose almost nothing. If it is high, look at option 2.
- **Build.** `CONCURRENTLY` scans the 80M-row table twice and waits for open transactions. Run it off-peak and make sure `maintenance_work_mem` is reasonable. If the build fails it leaves an `INVALID` index behind. Find it with:
  ```sql
  SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
  ```
  Then drop it and retry.

**Rollback:** `DROP INDEX CONCURRENTLY idx_orders_open_region_created;`

### 2. Alternative if HOT updates matter a lot: index on `(region, created_at DESC)` without `status`

```sql
CREATE INDEX CONCURRENTLY idx_orders_region_created
    ON orders (region, created_at DESC);
```

- **ESR:** E = `region`, S/R = `created_at`. `status` is checked as a heap filter.
- **Upside:** status updates stay HOT-eligible, and this index makes `idx_orders_region` redundant because `(region)` is its leftmost prefix.
- **Downside:** it reads rows newest-first until it finds 50 NEW/PAID rows. If most last-day orders are still NEW/PAID, that's about 50–100 rows and nearly as fast as option 1. If only a small share are, it can read up to all ~20k last-day rows of the region per call. That would be bad at 400 QPS.
- **How to decide:** pick using this number.
  ```sql
  SELECT region,
         count(*) FILTER (WHERE status IN ('NEW','PAID')) AS open_last_day,
         count(*) AS all_last_day
  FROM orders
  WHERE created_at > now() - interval '1 day'
  GROUP BY region ORDER BY region;
  ```

### 3. Cleanup after the new index has run for a while

`idx_orders_region` on a 12-value column is very likely dead weight that costs you on every insert. After a week or two, check whether anything still uses it, then drop it if `idx_scan` stays flat and no constraint or other query needs it:

```sql
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE relname = 'orders';
-- if unused:
DROP INDEX CONCURRENTLY idx_orders_region;
```

Keep `idx_orders_created` unless the same check shows it's unused; other queries probably need it.

## Verify: seeing the plan the app actually gets

**Why psql can differ from the app.** pgJDBC switches to a named server-side prepared statement after `prepareThreshold` executions (default 5). Then the server runs 5 more executions with custom plans, and after that it may switch to a **generic plan** with `$1` in place of the region value (`plan_cache_mode = auto`). A plain `EXPLAIN` in psql with a literal region always shows a *custom* plan, so it can look fine while the app runs something else.

### A. Reproduce the generic plan in psql (PG 13 syntax: `PREPARE` + `plan_cache_mode`, available since 12)

```sql
-- pgJDBC's setString binds as varchar by default, so use the same type
PREPARE q(varchar) AS
SELECT id, status, total
FROM orders
WHERE region = $1 AND status IN ('NEW','PAID') AND created_at > now() - interval '1 day'
ORDER BY created_at DESC
LIMIT 50;

SET plan_cache_mode = force_generic_plan;       -- what the app gets after ~10 executions
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) EXECUTE q('EU');   -- plan shows $1 => generic

SET plan_cache_mode = force_custom_plan;        -- what psql normally shows you
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) EXECUTE q('EU');

DEALLOCATE q; RESET plan_cache_mode;
```

This is a read-only `SELECT`, so `ANALYZE` is safe to run. Try it with your busiest and quietest region. If `region` is not `varchar`/`text` (for example an enum or `char(n)`), tell me the type, because the implicit cast can change the plan.

### B. Capture the plans the app really runs: `auto_explain`

This logs the plan that actually executed inside the app's own sessions, including whether it was generic.

```sql
-- as superuser; affects NEW connections of the app role only (recycle the pool afterwards)
ALTER ROLE app_user SET session_preload_libraries = 'auto_explain';
ALTER ROLE app_user SET auto_explain.log_min_duration = '20ms';  -- only log slow executions
ALTER ROLE app_user SET auto_explain.log_analyze = on;
ALTER ROLE app_user SET auto_explain.log_buffers = on;
ALTER ROLE app_user SET auto_explain.log_timing = off;           -- cuts most of the ANALYZE overhead
ALTER ROLE app_user SET auto_explain.sample_rate = 0.05;         -- instrument 5% of statements at 400 QPS
```

- **Where to set it.** You can put the same settings in `postgresql.conf` and reload. `session_preload_libraries` doesn't need a restart, but only new sessions pick it up.
- **Reading the log.** In the logged plan, `$1` in `Index Cond` means a generic plan.
- **Bind values (optional).** To see the values next to slow statements, set `log_parameter_max_length` (new in 13) together with `log_min_duration_statement`.
- **Turning it off.** Undo with `ALTER ROLE app_user RESET ALL;`, or reset each setting.
- **Overhead.** `log_analyze` adds instrumentation to every *sampled* statement, so keep `sample_rate` low and remove the settings once you have your evidence.

### C. Before/after numbers from `pg_stat_statements` (13 column names)

```sql
SELECT calls, round(mean_exec_time::numeric, 3) AS mean_ms,
       round(shared_blks_hit::numeric / calls, 1) AS blks_hit_per_call,
       round(shared_blks_read::numeric / calls, 1) AS blks_read_per_call
FROM pg_stat_statements
WHERE query ILIKE '%FROM orders%status IN%created_at DESC%LIMIT%';
```

Run `SELECT pg_stat_statements_reset();` right after the index build if you want a clean "after" window.

**A good plan looks like this**, for the generic plan and with a real region value:

```
Limit (actual rows=50)
  -> Index Scan using idx_orders_open_region_created on orders (actual rows=50)
       Index Cond: ((region = $1) AND (created_at > (now() - '1 day'::interval)))
       Buffers: shared hit=<~100>
```

There should be no `Sort` node, no `Rows Removed by Filter`, and no `Bitmap Heap Scan`. If you still see a `Sort`, or the generic plan picks `idx_orders_created`, send me both plans from step A and I'll analyze them.

## Assumptions / open questions

I made these assumptions so I could answer now. Please confirm or correct them:

1. **Exact minor version** (`SELECT version();`). Planner behavior doesn't depend on the minor release, but you should be on the final 13.x.
2. **Column types** of `region` and `status` (`\d orders`). I assumed `text`/`varchar` or an enum, and `created_at timestamptz`.
3. **Share of last-day orders that are still NEW/PAID**, per region (the query in option 2). This decides between option 1 and option 2.
4. **Current HOT ratio** (the `pg_stat_user_tables` query above), and whether write latency or WAL volume is already a concern at 300 inserts/s.
5. **Current and target latency** for this query (p95/p99), and whether it runs on the primary or a replica.
6. **Other hot queries on `orders`** (top entries in `pg_stat_statements`), so the new index or the dropped `idx_orders_region` doesn't hurt them.
7. **Whether the statuses stay literals in the app's SQL.** The partial index depends on it.

Please don't send real customer data; schemas, counts and plans are enough.
