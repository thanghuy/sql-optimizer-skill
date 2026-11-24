Thanks for including the version, schema, indexes, volumes and traffic. That is enough to design the index now. I checked the PostgreSQL **16** docs (prepared statements / `plan_cache_mode`, index-only scans), and everything below works on 16.4 and on RDS.

## Diagnosis

None of your current indexes can return rows in `updated_at` order. So for each call PostgreSQL does one of these:

- uses `idx_products_tenant`, reads **every** product of the tenant (up to 6M rows for your biggest tenant, about 5.1M of them active), checks `is_active` and `price` against the heap, then runs a top-N sort to keep 24 rows; or
- bitmap-ANDs `idx_products_tenant` and `idx_products_price`, which is still about 1.5M matching rows for a big tenant (5.1M active × 30%), followed by the same sort.

Either way the work grows with tenant size, not with `LIMIT 24`. That is why p95 is about 900 ms: your large tenants decide the p95. (Please confirm this with the "before" plan in the Verify section.)

## Recommendation

### 1. One composite index in ESR order (main fix)

```sql
-- Run outside a transaction block, off-peak. It does not block reads or writes.
SET maintenance_work_mem = '2GB';          -- session only; size it to your RDS instance class
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated
    ON products (tenant_id, is_active, updated_at DESC, id DESC, price);
```

And make the ORDER BY deterministic (this is optional but recommended, and the index above already matches it):

```sql
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC, id DESC
LIMIT 24;
```

**ESR mapping**

| Column | Role | Why |
|---|---|---|
| `tenant_id` | **E** | `= $1`; the seek goes straight to the tenant |
| `is_active` | **E** | `= true`; low cardinality, but as an equality column it still narrows the slice, and the order stays sorted after it |
| `updated_at DESC` | **S** | matches `ORDER BY updated_at DESC`, so there is no Sort node and the scan stops after 24 matches |
| `id DESC` | **S** (tie-breaker) | many products can share an `updated_at` (bulk price updates). Without it the 24 rows can change between calls, and keyset pagination (`(updated_at, id) < ($4, $5)`) would be impossible. It must sit right after `updated_at` |
| `price` | **R** | `BETWEEN` goes last. As a **key** column it is checked inside the index (it shows as part of `Index Cond`), so non-matching rows are skipped without visiting the heap. As an `INCLUDE` column it could not guide the scan |

**Why ESR and not E-R-S `(tenant_id, is_active, price, updated_at)`:** here the range is not selective (30%).
- ESR: about 24 / 0.30 ≈ **80 index entries** read, then 24 heap fetches. No sort.
- E-R-S for the biggest tenant: about 5.1M × 0.30 ≈ **1.5M rows** read and sorted to return 24.

ESR wins by about 4 orders of magnitude. Expect well under 5 ms per call, a few dozen buffers, and latency that no longer depends on tenant size.

**Why not a partial index (`... WHERE is_active`)?** It would work, because your query uses the literal `true`, so the planner can match the predicate even in a prepared statement. But at 85% active it only saves about 15% of size. It also could not replace `idx_products_tenant`, because queries without `is_active = true` could not use it. With `is_active` in the key instead, the leftmost prefix `(tenant_id)` serves every query `idx_products_tenant` serves today. That lets you drop the old index (step 3), so your index count stays the same. That matters with 170 writes/sec.

**Why not covering (`INCLUDE (title)`)?**
- With LIMIT 24 there are only 24 heap lookups per call. At 1,200 calls/s that is about 29k mostly-cached buffer hits/s, which is cheap.
- `title` is unbounded `text`, so including it would inflate a 42M-entry index.
- With 150 updates/s the visibility map is constantly cleared on hot pages, so an index-only scan would still do heap fetches. The PG 16 docs say directly that payload columns help only if "the table changes slowly enough" that an index-only scan avoids the heap.

Revisit this only if the after-plan shows heap access is the bottleneck.

**Cost**
- **Size:** about 42M × ~50 bytes per entry ≈ **2–2.5 GB** (B-tree deduplication will not help much, because `id` makes every key unique). Have at least that much free storage again for the build's sort/temp space.
- **Build:** `CONCURRENTLY` scans the table twice. Expect tens of minutes on 42M rows, depending on instance class and IOPS. It waits for open transactions, so avoid running it next to long-running queries.
- **Writes:** +1 index insert per INSERT (20/s) and per non-HOT UPDATE. Your price updates are already non-HOT, because `price` is indexed by `idx_products_price`. Adding `updated_at` to an index therefore costs about one extra index tuple per update (about 170/s). That is small, and it is cancelled out once `idx_products_tenant` is dropped.

**Rollback:** `DROP INDEX CONCURRENTLY idx_products_tenant_active_updated;`

If the build fails, it leaves an `INVALID` index behind. Check with `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;`, then drop it and retry.

### 2. Make sure the Go prepared statement gets the good plan

Your Go service uses prepared statements (pgx caches them by default). In PG 16, after **5 executions** the server may switch to a **generic plan** that does not know the actual `$1`/`$2`/`$3`. With this index the generic plan should still choose it, because it serves both the seek and the ORDER BY whatever the parameters are. Your tenant sizes are very skewed (from a few rows to 6M), so check it anyway (see Verify).

If the generic plan picks something else, set it for the application role only, not globally:

```sql
ALTER ROLE your_app_role SET plan_cache_mode = force_custom_plan;
```

This adds a little planning time per call, about 0.1–0.3 ms for a query this simple.

### 3. Clean up the now-redundant index (after a week of monitoring)

`idx_products_tenant (tenant_id)` is a leftmost prefix of the new index. Once `pg_stat_user_indexes` shows the new index taking the traffic, and you have checked that no hot query relies only on the old one:

```sql
DROP INDEX CONCURRENTLY idx_products_tenant;
```

Leave `idx_products_price` alone unless its `idx_scan` stays at 0. Other queries (price searches across tenants) may use it.

### One edge case to know about

ESR is fast because 30% of rows match, so 24 matches are found within about 80 entries. If the UI allows **very narrow or empty price ranges** (e.g. a filter that matches 0.01% of a big tenant, or nothing), the scan walks far down the tenant's index before it finds 24 matches. In the no-match case it reads all ~5.1M entries (hundreds of MB of index pages, still no heap access).

If your p99 or the `pg_stat_statements` max time shows this, there are two options. One is to add a second index `(tenant_id, is_active, price)` and keep custom plans, so the planner can choose E-R-S for narrow ranges. The other is to limit the price filter in the API. Do not add that second index pre-emptively, given your write rate.

## Verify

**1. Before (now) and after, on staging with production-like data or on a replica, with your biggest tenant and a typical tenant:**

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = <biggest_tenant_id> AND is_active = true AND price BETWEEN 10 AND 50
ORDER BY updated_at DESC, id DESC
LIMIT 24;
```

**2. The plan the app actually gets** (generic plan, PG 16+):

```sql
EXPLAIN (GENERIC_PLAN)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC, id DESC
LIMIT 24;
```

**What a good plan looks like:**

```
Limit (actual rows=24)
  -> Index Scan using idx_products_tenant_active_updated on products
       Index Cond: ((tenant_id = ...) AND (is_active = true) AND (price >= ...) AND (price <= ...))
       (actual rows=24 loops=1)
       Buffers: shared hit=<a few dozen>
```

- **No `Sort` node** (and no `Incremental Sort`). If one appears, the ORDER BY and the index directions do not match.
- `price` appears **inside `Index Cond`**, not as a separate `Filter` with a large `Rows Removed by Filter`.
- Buffers in the tens, not tens of thousands. Execution time in low single-digit ms for the biggest tenant.
- An `Index Scan` (not `Index Only Scan`) is expected and fine here, because `title` is not covered.
- The `GENERIC_PLAN` output uses the same index. If it shows `idx_products_tenant`, a Bitmap AND, or a Sort, apply step 2.

**3. In production after rollout:**

```sql
-- p50/mean and buffer use for this statement (reset or note the values before rollout)
SELECT calls, round(mean_exec_time::numeric,2) AS mean_ms, round(max_exec_time::numeric,2) AS max_ms,
       shared_blks_hit / NULLIF(calls,0) AS hit_per_call, shared_blks_read / NULLIF(calls,0) AS read_per_call
FROM pg_stat_statements
WHERE query ILIKE '%FROM products%ORDER BY updated_at DESC%';

-- the new index is used, and the old one has stopped being used
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE relname = 'products';
```

Get p95 from your app metrics, or from RDS Performance Insights. For slow app-side plans, enable `auto_explain` through the RDS parameter group (`auto_explain.log_min_duration = '50ms'`).

Fill in this table and send it back if anything does not look right:

| Metric | Before | After |
|---|---|---|
| Execution time (biggest tenant) | | |
| Rows read (actual rows on scan node + Rows Removed) | | |
| Buffers (shared hit + read) | | |
| Sort node | yes | should be no |
| Index used | | idx_products_tenant_active_updated |
| App p95 | ~900 ms | target < 50 ms |

## Assumptions / open questions

- **Paging:** I assumed the endpoint shows only the first 24 rows, or that you can move to keyset paging. If it uses `OFFSET`, tell me: deep offsets would read and discard rows, and keyset paging on `(updated_at, id)` is the fix that this index already supports.
- **Other hot queries:** I assumed no other hot query on `products` needs a different leading column. Run the `pg_stat_statements` query sorted by `total_exec_time` and send me the other top `products` queries before you drop `idx_products_tenant`.
- **Price range distribution:** "30% of a tenant's active products" is the typical case. If narrow or empty ranges are common, see the edge case above.
- **Index size:** the estimate assumes default fillfactor. If `updated_at` changes are mostly "move to now", the right-most pages of each tenant get the inserts and the rest stays fairly compact. Watch for bloat with `pgstattuple` after a few weeks.
