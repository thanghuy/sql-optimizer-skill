**Short answer: no, don't add a composite index. Fix the statistics.** Your indexes are fine. The planner badly underestimates how many customers match `city = 'Hanoi' AND country = 'VN'`, so it picks a nested loop that runs 480k times. A composite index would not change that estimate.

(I checked the PostgreSQL 16 docs on extended statistics, https://www.postgresql.org/docs/16/planner-stats.html. Everything below is valid on 16.)

## Diagnosis

The plan shows what happened:

| Node | Estimated rows | Actual rows | Gap |
|---|---|---|---|
| Bitmap Heap Scan on customers (city AND country) | **112** | **479,812** | ~4,300× |
| Nested Loop output | 112 | 7,398,120 | ~66,000× |
| Inner Index Scan on orders_customer_id_idx | 15 per loop | 15 per loop, **479,812 loops** | per-row estimate is right |

1. **Each column's estimate is accurate.** `city = 'Hanoi'` is estimated at 480,100 (actual 479,812) and `country = 'VN'` at 2,100,500 (actual 2,103,377). The problem is how the planner combines them.
2. **Without multi-column statistics, PostgreSQL assumes the columns are independent** and multiplies their selectivities. City and country are not independent: Hanoi implies VN, so the `country` condition removes nothing. The real result is about P(city), but the planner calculates P(city) × P(country), which is far too small.
3. **The cheap estimate leads to a nested loop.** With 112 outer rows, "112 index probes into orders" costs 2,051, which beats hashing anything. In reality it made 479,812 probes. Those probes touched 1.2M + 6.2M buffers. The 6,211,040 blocks *read* are about **47 GB of mostly random 8 KB I/O**, and that is your 9 minutes. The 2-second version was almost certainly a hash join (hash the ~480k Hanoi customers, scan orders once, possibly in parallel), which is the right shape for a query that returns 7.4M join rows.
4. **Re-running ANALYZE cannot fix this.** Autoanalyze only refreshes per-column statistics, and those are already correct. Nothing currently tells the planner that city and country are correlated.

**One number doesn't add up, and I'd like you to check it.** Pure independence gives 9M × (480,100 / 9M) × (2,100,500 / 9M) ≈ **112,000**, not 112. An estimate of 112k would be about 4× low. That is bad, but it would not usually cause a 480k-loop nested loop. An estimate of 112 is another 1,000× lower. So one of these is true:
- (a) the excerpt was trimmed or retyped (112k shown as 112), or
- (b) something else is pushing the estimate down: an existing extended-statistics object with a stale MCV list, a very wrong `reltuples` for `customers`, or a different version of the query (a generic plan with parameters, extra predicates).

The fix below is the same in every case, but please run the checks in "Verify, step 0" so we know which one it is.

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `WHERE c.city = 'Hanoi' AND c.country = 'VN'` | Correlated columns, and no extended statistics on `(city, country)` | **High** | Estimate is 112 against 479,812 actual, so the planner picks a nested loop with 480k index probes into a 140M-row table (about 47 GB of reads) instead of a hash join | `CREATE STATISTICS ... (dependencies, mcv) ON city, country FROM customers; ANALYZE customers;` |
| 2 | `AND c.country = 'VN'` with `customers_country_idx` | A redundant predicate in practice: it filters nothing, but it adds a second bitmap index scan over 2.1M entries and causes the multiplied estimate | Medium | `BitmapAnd` builds and intersects a 2.1M-entry bitmap for no benefit, and the predicate is the source of the misestimate | Keep it if `city` alone is not guaranteed unique to VN (other countries can have a city with the same name). Once the statistics from fix #1 exist, the planner usually stops using the country index here by itself. Only remove the predicate if your data model guarantees city implies country |
| 3 | `GROUP BY c.id, c.name` | Not a problem: `c.id` is the PK, so `c.name` is functionally dependent and costs almost nothing | Low (optional) | — | You can write `GROUP BY c.id`. The output is the same; it is purely cosmetic |

The SQL itself is otherwise fine. It has no anti-pattern that an index or rewrite needs to fix.

## Rewritten query

No rewrite is needed. Keep the query as it is:

```sql
SELECT c.id, c.name, SUM(o.total)
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.city = 'Hanoi' AND c.country = 'VN'
GROUP BY c.id, c.name;
```

## Recommendation

### 1. Create extended statistics on (city, country). This is the actual fix

```sql
CREATE STATISTICS customers_city_country_stx (dependencies, mcv)
  ON city, country FROM customers;

ANALYZE customers;   -- needed once now; autoanalyze will maintain it afterwards
```

- **`dependencies`** records that city determines country, so the planner stops multiplying their selectivities. The PG 16 docs say functional dependencies apply to "simple equality conditions that compare columns to constant values, and `IN` clauses with constant values", which is exactly your WHERE clause.
- **`mcv`** stores the most common (city, country) *pairs* with their real frequency. Hanoi/VN is about 5% of the table, so it will be in the MCV list, and the estimate for this exact pair becomes close to the true 480k. It is more accurate than dependencies alone and also covers other combinations (for example `city = 'Hanoi' AND country = 'US'` will no longer be estimated as 480k).
- **Expected effect:** the customers estimate rises to about 480k and the join estimate to about 7.4M. The planner then goes back to a **Hash Join** (customers hashed, orders scanned once, likely with Parallel Seq Scan) plus HashAggregate, back near your 2 s baseline.
- **Cost:** almost nothing. It is a catalog object plus a little extra ANALYZE work on `customers`, with no write overhead on INSERT/UPDATE, no disk to speak of, and no locks beyond `ANALYZE`'s ShareUpdateExclusive lock (which does not block reads or writes).
- **Rollback:** `DROP STATISTICS customers_city_country_stx;`
- If country names or other columns are often filtered together with city elsewhere (for example `district`), add them to the same statistics object or create another one. Keep each object to the columns that are actually queried together.

### 2. Why not a composite index (city, country)?

- **It does not fix the estimate.** The PostgreSQL planner does not get joint selectivity from a multi-column B-tree index. (Only *expression* indexes get their own statistics.) It would still multiply per-column selectivities, still estimate too few rows, and still choose the nested loop. You would pay write overhead on a 9M-row table and get the same 9-minute plan.
- **It does not help the access path much.** Since every Hanoi row is VN, `(city, country)` returns exactly the same rows as `customers_city_idx`. The 480k-row bitmap scan on customers is not the bottleneck; the 480k probes into orders are.
- **ESR view, for completeness:** `city` and `country` are both E columns and there is no S or R. A useful index would just be `(city)`, which you already have.

### 3. Optional, only if the report is still not fast enough after #1: a covering index on orders

```sql
CREATE INDEX CONCURRENTLY orders_customer_id_total_idx
  ON orders (customer_id) INCLUDE (total);
-- then, after checking pg_stat_user_indexes for a while:
-- DROP INDEX CONCURRENTLY orders_customer_id_idx;   -- the new one makes it redundant,
--   but check first that no FK or other query needs its exact definition (the new one also serves the FK)
```

- ESR mapping: E = `customer_id` (join key), with `total` as INCLUDE so the index covers the query.
- This lets a nested loop or merge join use an **index-only scan** without heap fetches. That helps only if the planner chooses an index path, and only if the visibility map is fresh (VACUUM). With the correct estimates, a hash join over `orders` will usually win, and this index won't be used for this query.
- **Cost:** a 140M-row index (on the order of 4–6 GB; check with `pg_relation_size`). Building it needs about the same amount of temp/disk space and takes a long time with `CONCURRENTLY`, and every INSERT into orders pays for it. Don't build it unless the plan after fix #1 shows a need, and unless your write rate on `orders` allows it.
- `CONCURRENTLY` cannot run inside a transaction. If the build fails, check for an `INVALID` index and drop it.

### Not recommended

- **`SET enable_nestloop = off` in application code, or `pg_hint_plan` hints.** They freeze one plan shape and hide the real problem. Use `enable_nestloop` only as the diagnostic in "Verify" below.

## Verify

**Step 0: explain the 112 (diagnostic, read-only):**

```sql
-- Is there already an extended-statistics object (possibly stale or wrong)?
SELECT statistics_name, attnames, kinds, dependencies, most_common_vals IS NOT NULL AS has_mcv
FROM pg_stats_ext WHERE tablename = 'customers';

-- Does the planner know the real table size?
SELECT reltuples::bigint, relpages FROM pg_class WHERE oid = 'customers'::regclass;
SELECT n_live_tup, last_analyze, last_autoanalyze FROM pg_stat_user_tables WHERE relname = 'customers';

-- The single-table estimate on its own, before the fix:
EXPLAIN SELECT 1 FROM customers WHERE city = 'Hanoi' AND country = 'VN';
```

**Step 1: after `CREATE STATISTICS` + `ANALYZE`, check the estimate first (cheap, does not run the query):**

```sql
EXPLAIN SELECT 1 FROM customers WHERE city = 'Hanoi' AND country = 'VN';
-- good: rows ≈ 480000 (instead of 112 / 112000)
```

**Step 2: the actual plan of the report (this runs the query, so use a replica or off-peak):**

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT c.id, c.name, SUM(o.total)
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.city = 'Hanoi' AND c.country = 'VN'
GROUP BY c.id, c.name;
```

A good plan looks like this:
- the customers node estimates about 480k (within 2× of actual)
- the join is a **Hash Join** (or a Parallel Hash Join under Gather), with `orders` read once by a (parallel) Seq Scan, **not** `loops=479812`
- `shared read` is roughly the size of `orders`, read sequentially, instead of 6.2M random reads
- the time is back to seconds

**If you want to confirm the diagnosis before changing anything**, run this in one session only:

```sql
BEGIN;
SET LOCAL enable_nestloop = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;   -- same query
ROLLBACK;
```

If this finishes in seconds, the estimate was the whole problem.

| Metric | Before | After (expected) |
|---|---|---|
| Execution time | ~541 s | ~2 s (your old baseline) |
| customers estimate vs actual | 112 vs 479,812 | ~480k vs 479,812 |
| Join | Nested Loop, 479,812 loops | Hash Join, orders read once |
| Buffers read | 6.2M (~47 GB random) | about the size of orders, sequential |

## Assumptions / open questions

Things I'd still like to know. None of them block the fix above:

1. **What changed between 2 s and 9 min?** For example: was `customers_country_idx` added recently, did a data load grow Hanoi/VN, was there an upgrade or `pg_upgrade` (which drops statistics until ANALYZE runs), or did someone drop a statistics object? This tells us whether other queries are at risk too.
2. **Is this exactly the SQL the app sends**, with literals, or a prepared statement with `$1/$2`? With parameters, PG 16 may switch to a generic plan after 5 executions (`plan_cache_mode`). Check with `EXPLAIN (GENERIC_PLAN) ...` (16+). Extended statistics help custom plans most.
3. **How often does this report run, and on primary or replica?** If it is frequent, consider a summary table or materialized view of `SUM(total)` per customer instead of re-aggregating 7.4M orders each time.
4. I assumed `total` is a plain numeric column and that `orders.customer_id` has no large skew. If a few customers have millions of orders, tell me and I'll revisit the join choice.
