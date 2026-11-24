You've given me almost everything I need (version, schema, indexes, volume, skew, traffic), so here is a concrete recommendation followed by a verification plan. I have not run anything against your database. Everything below is DDL or a command for you to run, ideally on staging with production-like data first.

## Diagnosis

The query is `WHERE tenant_id = ? AND is_active = true AND price BETWEEN ? AND ? ORDER BY updated_at DESC LIMIT 24`. None of the current indexes can return rows in `updated_at` order for a single tenant, so PostgreSQL is forced into one of two bad plans:

- **`idx_products_tenant`** (likely for big tenants): it reads every row of the tenant. For your biggest tenant that is about 6M rows, of which about 5.1M are active and about 1.5M fall in the price range. It filters `is_active` and `price` on the heap, then runs a top-N sort to keep 24 rows. That is millions of heap visits to return 24 rows.
- **`idx_products_price`**: the range covers about 30% of a tenant's rows, so across all tenants it is very unselective. It ends up as a bitmap scan of a big slice of the table plus a filter on `tenant_id` and a sort.

Either plan fits a p95 around 900 ms. Small tenants are probably fast, and the p95 comes from the large tenants.

There is also a prepared-statement factor. Your Go driver uses prepared statements, so after 5 executions PostgreSQL may switch to a **generic plan** that ignores the actual `tenant_id` and price values. That is a good reason to choose an index the planner prefers for every parameter value, not just for some of them.

## Recommendation

### 1. One ESR index (main fix)

```sql
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated_price
    ON products (tenant_id, is_active, updated_at DESC, price);
```

**ESR mapping**

| Column | Role | Why |
|---|---|---|
| `tenant_id` | **E** | `= $1`. It is the seek prefix. |
| `is_active` | **E** | `= true`. It is fixed to one value, so the next column stays sorted. |
| `updated_at DESC` | **S** | It matches `ORDER BY updated_at DESC`. Rows come out already ordered, there is no Sort node, and `LIMIT 24` stops the scan early. |
| `price` | **R** | `BETWEEN`. It does not bound the seek, but because it is a key column the B-tree checks it on the index tuple (it shows up in `Index Cond`). The heap is visited only for rows that match. |

**Why ESR rather than E‑R‑S (`tenant_id, is_active, price, updated_at`)**, using your numbers for the biggest tenant:

- Rows matching E: N_e ≈ 5.1M. Rows matching E and R: N_er ≈ 1.5M (30%).
- With ESR, the scan reads about `24 / 0.30 ≈ 80` index entries and visits about 24 heap rows. There is no sort.
- With E‑R‑S, the scan reads about 1.5M entries and then sorts them for a top‑24.
- ESR wins by roughly four orders of magnitude. It stays good unless price ranges get very narrow (see Assumptions).

**Design choices**

- **Keep `is_active` in the key rather than using a partial index `WHERE is_active`.** A partial index would only save about 15% of entries, because 85% of rows are active. Keeping the full index means its `(tenant_id)` prefix can serve every other tenant-scoped query, which lets you drop `idx_products_tenant` (step 3). Because of alignment, the boolean also adds almost nothing: `int4` + `bool` still pads to 8 bytes before the `timestamptz`.
- **No `INCLUDE (title)` and no attempt at an index-only scan.** An index-only scan skips the heap only for pages marked all-visible in the visibility map. The PG 16 docs say outright that `INCLUDE` doesn't help "if the table changes frequently" ([indexes-index-only-scans](https://www.postgresql.org/docs/16/indexes-index-only-scans.html)). Here the rows at the top of this index are the most recently updated ones, which are the least likely to be all-visible. You would pay to store every `title` in the index and still get heap fetches. Fetching about 24 heap rows, mostly from cache, is cheap.

**Expected effect:** about 80 index tuples plus about 24 heap fetches. That is typically tens of buffers and low single-digit milliseconds even for the 6M-product tenant, which is well under your 50 ms target.

**Cost**

- **Size:** roughly 40 bytes per entry × 42M ≈ **1.5–2.5 GB**. Run `pg_relation_size` after the build to get the real figure.
- **Build:** `CONCURRENTLY` does two table passes and waits for running transactions. Expect minutes to tens of minutes on 42M rows. It cannot run inside a transaction block.
  - Run it off-peak.
  - Raise `maintenance_work_mem` for the session, for example `SET maintenance_work_mem = '2GB';`.
  - Make sure the RDS instance has free storage for the index plus sort temp space, and watch `FreeStorageSpace`.
- **Writes:** each of the about 150 updates/s that changes `updated_at` now also writes a new entry in this index. HOT updates are already lost whenever `price` changes, because `idx_products_price` exists. Since your updates mostly change price and `updated_at` together, the extra HOT loss is small. Updates that change only `updated_at` will stop being HOT. Check this with the `n_tup_hot_upd` query under Verify.
- **Bloat:** the index will churn at the "hot" end. Autovacuum needs to keep up. Consider per-table settings such as `ALTER TABLE products SET (autovacuum_vacuum_scale_factor = 0.02);` if vacuum lags.

**Rollback:** `DROP INDEX CONCURRENTLY idx_products_tenant_active_updated_price;`. If the build fails, it leaves an `INVALID` index behind. Find it with `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;` and drop it the same way.

### 2. Optional: make the ordering deterministic

`ORDER BY updated_at DESC LIMIT 24` is not stable when two rows share the same `updated_at`. If this endpoint pages through results, or ever will, use a tiebreaker and keyset pagination instead of `OFFSET`:

```sql
-- index variant (use this one INSTEAD of #1 if you adopt the tiebreaker)
CREATE INDEX CONCURRENTLY idx_products_tenant_active_updated_id_price
    ON products (tenant_id, is_active, updated_at DESC, id DESC, price);

-- page 2+: pass the last row's (updated_at, id)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
  AND (updated_at, id) < ($4, $5)
ORDER BY updated_at DESC, id DESC
LIMIT 24;
```

### 3. Clean up the existing indexes (after verification, not on the same day)

- **`idx_products_tenant (tenant_id)`**: this is a leftmost prefix of the new index, so it becomes redundant. It is not tied to a FK or a uniqueness constraint. Once the new index has been in use for a while, run `DROP INDEX CONCURRENTLY idx_products_tenant;`. That removes write work and gets back roughly 0.3–1 GB.
- **`idx_products_price (price)`**: do not drop it blindly. Check `idx_scan` (query below) to see whether any other query uses it. If nothing does, dropping it has two benefits: it removes one index from every price update, and it gives the planner one less bad option for this query. Note that it does **not** bring back HOT updates for price changes, because `price` is also in the new index.

After the change you have 2–3 secondary indexes instead of 3, so the write budget on this table goes down rather than up.

## Verify

Run these on staging with production-like data if you can. Otherwise run them on a read replica, or on the primary off-peak. They are all read-only `SELECT`s, so `EXPLAIN ANALYZE` is safe here.

**0. Capture the baseline before creating the index**

```sql
-- pg_stat_statements (enabled by default on RDS)
SELECT queryid, calls, round(mean_exec_time::numeric,2) mean_ms,
       round(stddev_exec_time::numeric,2) stddev_ms, rows,
       shared_blks_hit, shared_blks_read
FROM pg_stat_statements
WHERE query ILIKE '%FROM products%ORDER BY updated_at DESC%';

-- write / HOT baseline (sample twice, a few minutes apart, to get rates)
SELECT n_tup_upd, n_tup_hot_upd, n_tup_ins,
       round(100.0 * n_tup_hot_upd / nullif(n_tup_upd,0), 1) AS hot_pct
FROM pg_stat_user_tables WHERE relname = 'products';
```

Then run the plan for **the biggest tenant** and for **a typical tenant**, each with a realistic price range:

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = <biggest_tenant_id> AND is_active = true
  AND price BETWEEN 20.00 AND 80.00
ORDER BY updated_at DESC
LIMIT 24;
```

**1. Create the index (DDL above), then confirm it is valid**

```sql
SELECT indexrelid::regclass, indisvalid, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_index WHERE indrelid = 'products'::regclass;
```

**2. Re-run the same `EXPLAIN (ANALYZE, BUFFERS)` for both tenants.** A good plan looks like this:

```
Limit  (actual time=0.05..0.4 rows=24 loops=1)
  Buffers: shared hit=~30-150
  ->  Index Scan using idx_products_tenant_active_updated_price on products
        Index Cond: ((tenant_id = ...) AND (is_active = true)
                     AND (price >= ...) AND (price <= ...))
        Buffers: shared hit=...
```

Check each of the following:
- The plan uses an **`Index Scan`** on the new index. It is *not* `Bitmap Heap Scan` and *not* `idx_products_tenant` or `idx_products_price`.
- There is **no `Sort` node** and no `Incremental Sort`. A Sort node means the S part isn't being used.
- **`price` appears in `Index Cond`**, with no big `Rows Removed by Filter`. If price shows up as `Filter:` with a cast, such as `(price)::double precision`, the driver is sending the parameters as `float8`. Make the parameters `numeric`, or cast the parameter in SQL: `$2::numeric`.
- Buffers are in the tens or low hundreds, against what is probably tens of thousands or more in the baseline.

**3. Check the generic plan, which is what your Go service may actually get**

```sql
-- what the generic plan looks like (PG 16; cannot be combined with ANALYZE)
EXPLAIN (GENERIC_PLAN)
SELECT id, title, price, updated_at FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC LIMIT 24;

-- and execute it as a generic plan for the biggest tenant
SET plan_cache_mode = force_generic_plan;
PREPARE p(int, numeric, numeric) AS
  SELECT id, title, price, updated_at FROM products
  WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
  ORDER BY updated_at DESC LIMIT 24;
EXPLAIN (ANALYZE, BUFFERS) EXECUTE p(<biggest_tenant_id>, 20.00, 80.00);
DEALLOCATE p; RESET plan_cache_mode;
```

The generic plan should also use the new index with no Sort. If the console plan is fast but the generic plan is not, the fallback is `ALTER ROLE <app_role> SET plan_cache_mode = force_custom_plan;`. It costs a little planning CPU per call at 1,200/s, so treat it as a fallback, not the first fix.

**4. Confirm in production what the app really runs**
- Reset the counters for just this query with `SELECT pg_stat_statements_reset(0, 0, <queryid>);`. After some peak traffic, compare `mean_exec_time` and `shared_blks_*` per call against the baseline.
- Optionally, enable `auto_explain` through the RDS parameter group (`shared_preload_libraries` needs a reboot). Use `auto_explain.log_min_duration = 50ms` and `log_buffers = on`, so any call still over target logs its real plan.
- Check p95 in your Go service metrics or Performance Insights. `pg_stat_statements` gives mean and stddev, not percentiles.

**5. Watch the write side for a few days**
- Rerun the `n_tup_hot_upd` query. A small drop in `hot_pct` is expected.
- Watch UPDATE latency in Performance Insights and the new index's size growth.
- Find out what is still using the old indexes:

  ```sql
  SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
  FROM pg_stat_user_indexes WHERE relname = 'products';
  ```

  Then drop the redundant ones as described in Recommendation step 3.

Summary of what to compare once you have numbers:

| Metric | Before | After (expected) |
|---|---|---|
| Execution time, biggest tenant | ~900 ms (p95) | < 5 ms |
| Rows read for 24 returned | ~1.5M–6M | ~80 |
| Buffers | very high | tens to low hundreds |
| Sort node | yes | no |
| Index used | `idx_products_tenant` / `idx_products_price` | `idx_products_tenant_active_updated_price` |

If you paste the before and after `EXPLAIN (ANALYZE, BUFFERS)` output, especially for the biggest tenant and the generic plan, I'll check it and adjust if needed.

## Assumptions / open questions

- **Price ranges are usually wide (about 30%).** ESR reads about `24 / selectivity` index entries. A 1% range on the biggest tenant still reads only about 2,400 entries, which is fine. A very narrow range like 0.01% has two problems:
  - It reads hundreds of thousands of entries.
  - It returns fewer than 24 rows, which forces a walk through the tenant's whole active range.

  If narrow ranges or "no results" searches are common, tell me their share. A second index on `(tenant_id, is_active, price)` would serve that case. The planner can only choose between the two with custom plans, though, and it adds write cost.
- **No correlation between price and `updated_at`.** I'm assuming recently updated products aren't systematically outside typical price ranges. If they are, the scan reads further before it finds 24 matches.
- **`title` is short.** It is not in the index either way, but if it is very long (TOASTed), the heap fetches cost slightly more.
- **Other hot queries.** I don't know the other hot queries on `products`. Run the `pg_stat_statements` top-20 query before you drop `idx_products_price`.
- **Driver parameter types.** I assumed the Go driver (pgx or lib/pq) sends `$2`/`$3` as numeric. The `Index Cond` check in Verify step 2 confirms this.
