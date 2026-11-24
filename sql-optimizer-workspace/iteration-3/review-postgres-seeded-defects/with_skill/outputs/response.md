I reviewed this against PostgreSQL 16 (I checked the PG 16 docs for `NOT IN` semantics; all DDL and EXPLAIN options below work on 16). You haven't said whether this is RDS, Aurora, Cloud SQL or self-hosted, but nothing below depends on that.

**Short version:** there's one correctness bug and four performance problems. The bug is that `NOT IN` against a `blacklist.customer_id` that contains NULLs returns **zero rows**, so this endpoint is currently giving you an empty report. The worst performance problems are two correlated subqueries on a 60M-row `orders` table that has no index on `customer_id`, and the fact that no index can serve `lower(c.email)`.

## Diagnosis

I don't have a plan yet, so this comes from the SQL text, your schema and the row counts. Please confirm it with the plan from the Verify section.

- **customers (4M):** no index on `email` or `lower(email)` is usable, so the planner has two bad choices:
  - a Seq Scan over 4M rows that evaluates `lower(email)` on each one;
  - walking `customers_created_idx` backwards and filtering every row until it finds 50,020 matches, which in practice can mean reading most of the table.

  Either way it has to produce 50,020 qualifying rows just to throw 50,000 of them away (`OFFSET`).
- **orders (60M):** each output row runs two scalar subqueries, and no index exists on `orders.customer_id`. Each subquery execution is therefore a scan of 60M rows. PostgreSQL computes target-list expressions before the Limit node discards the OFFSET rows. That means the cost is up to 2 × (rows the Limit reads) × 60M-row scans. This is what grows with your data.
- **blacklist (20k):** small. `NOT IN` is usually run as a hashed SubPlan here, which is cheap, but the result is wrong (next section).

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow / wrong | Fix |
|---|---|---|---|---|---|
| 1 | `c.id NOT IN (SELECT customer_id FROM blacklist)` | **Correctness bug.** `blacklist.customer_id` is nullable and you said it sometimes *is* NULL | **High (wrong results)** | PG 16 docs: if there's no match and at least one right-hand row is null, the result of `NOT IN` "will be null, not true". `WHERE` drops NULL, so **every customer is excluded and the query returns 0 rows** whenever one NULL exists. `NOT IN` also can't be planned as a real anti-join. If the list ever exceeds `work_mem`, it falls back to a plain SubPlan that rescans `blacklist` for every customer. | `NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = c.id)`. It ignores NULL rows and plans as an anti-join. |
| 2 | `(SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id)` and `(SELECT MAX(o.created_at) ... )` | Two correlated subqueries, and no index on `orders.customer_id` | **High** | Each runs once per row the Limit node reads, which is up to 50,020 rows, and each is a scan of 60M rows. Two subqueries means two passes per customer. | Index `orders (customer_id, created_at)`. Compute both values in **one** `LATERAL` aggregate, and only for the 20 rows on the page. |
| 3 | `lower(c.email) = lower($1)` | Function on the column, and no expression index | **High** | A plain index on `email` can't be used for `lower(email)`. The planner has to scan 4M rows or walk the `created_at` index while filtering. | Expression index on `lower(email)` (see the index below). Alternative: a `citext` column or a case-insensitive nondeterministic collation, but that's a bigger change. |
| 4 | `ORDER BY c.created_at DESC OFFSET 50000 LIMIT 20` | Large OFFSET | **High** | PG reads, sorts and computes the subqueries for 50,020 rows and then discards 50,000. Cost grows with the page number. | Keyset pagination: `WHERE (c.created_at, c.id) < ($2, $3)` plus `LIMIT 20`. |
| 5 | `ORDER BY c.created_at DESC` | No unique tie-breaker | Medium | Customers created at the same `created_at` can appear on two pages or on none, and keyset pagination needs a unique key. | `ORDER BY c.created_at DESC, c.id DESC` |
| 6 | `lower(c.email) = lower($1)` combined with `OFFSET 50000` | **Logic smell** | Medium (correctness of the feature) | If emails are unique or nearly unique, the filter matches 0–few rows and page 2,501 is always empty. Either the email filter is optional in the real app (e.g. `$1` is NULL for "all customers"), or the pagination is meaningless. | Please tell me which (see questions). If the filter is optional, the app should send a different query without it. A catch-all `($1 IS NULL OR …)` would plan badly. |
| 7 | `blacklist` has no index | Minor | Low | With 20k rows, a Hash Anti Join over the whole table is cheap. After the rewrite only about 20 probes happen, which an index makes trivial. | Optional: `CREATE INDEX CONCURRENTLY blacklist_customer_id_idx ON blacklist (customer_id) WHERE customer_id IS NOT NULL;` |

What I left out on purpose: `COUNT(*)` is fine (it's identical to `COUNT(1)`), and the `blacklist` table size isn't a problem in itself.

A related question about blacklist rows with a NULL `customer_id`: you said they stand for "blocked emails", but `blacklist` has no email column, so this query can never match them anyway. If blocked emails are meant to exclude customers by email, the table needs an email column (stored lowercased) and a second `NOT EXISTS`. Tell me how blocked emails are stored and I'll add it.

## Rewritten query

This version assumes the email filter really is required and uses keyset pagination. On the first page, leave out the `(created_at, id) <` line. On later pages, pass the `created_at` and `id` of the last row of the previous page as `$2` and `$3`.

```sql
SELECT p.id, p.name, os.order_count, os.last_order_at
FROM (
    SELECT c.id, c.name, c.created_at
    FROM customers c
    WHERE lower(c.email) = lower($1)
      AND NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = c.id)
      AND (c.created_at, c.id) < ($2, $3)          -- omit on the first page
    ORDER BY c.created_at DESC, c.id DESC
    LIMIT 20
) p
LEFT JOIN LATERAL (
    SELECT count(*) AS order_count, max(o.created_at) AS last_order_at
    FROM orders o
    WHERE o.customer_id = p.id
) os ON true
ORDER BY p.created_at DESC, p.id DESC;
```

Why it's written this way:
- **The inner subquery picks the 20 customers first.** The orders aggregate then runs exactly 20 times, whatever plan the planner picks for the outer part.
- **The LATERAL aggregate with no `GROUP BY` always returns one row.** Customers with no orders get `order_count = 0` and `last_order_at = NULL`, the same as the original.
- **Behaviour change you should know about:** customers who used to be wrongly excluded by the NULL in `blacklist` will now appear. That is the intended behaviour.
- **If the UI really needs "jump to page N", keyset pagination can't do that.** The cheaper fallback keeps OFFSET only in the inner subquery, which pages over the index without touching `orders`. It still reads N×20 index entries.

## Recommendation

**1. Index for the orders lookup (biggest win)**
```sql
CREATE INDEX CONCURRENTLY orders_customer_id_created_at_idx
    ON orders (customer_id, created_at);
```
- **ESR:** E = `customer_id` (the correlation / join column). Then `created_at`, so that `max(created_at)` is read from the end of the customer's slice and `count(*)` is answered from the same slice.
- **Expected effect:** both values come from one Index Only Scan per customer, touching about as many entries as that customer has orders (about 15 on average across 60M/4M), instead of 60M rows. Heap Fetches stay low only if `orders` is vacuumed regularly (the visibility map has to be current).
- **Cost:**
  - Disk: roughly 1.5–2.5 GB for 60M entries, plus about that much temporary space during the build.
  - Build time: tens of minutes on 60M rows. `CONCURRENTLY` doesn't block writes, but it takes longer and can't run inside a transaction.
  - Writes: every INSERT into `orders` maintains one more index. This index also serves the FK if `orders.customer_id` references `customers`, which currently has no supporting index (that also makes customer deletes slow).
- **Rollback:** `DROP INDEX CONCURRENTLY orders_customer_id_created_at_idx;`. If the build fails, check for an `INVALID` index in `\d orders` and drop it.

**2. Index for the email filter and the page order**
```sql
CREATE INDEX CONCURRENTLY customers_lower_email_created_id_idx
    ON customers (lower(email), created_at DESC, id DESC);
ANALYZE customers;   -- collects statistics for the lower(email) expression
```
- **ESR:** E = `lower(email)`, S = `created_at DESC, id DESC`. The sort columns match the rewritten ORDER BY exactly, including the tie-breaker. The keyset predicate `(created_at, id) < ($2, $3)` is a range on the sort columns, so the index serves both the seek and the order. No Sort node is needed, and `LIMIT 20` stops early.
- **No `INCLUDE (name)`:** only 20 heap lookups happen per page, so covering a `text` column isn't worth the extra size.
- **If emails are unique and the pagination turns out to be meaningless (issue 6):** the simpler `CREATE UNIQUE INDEX CONCURRENTLY … ON customers (lower(email))` is enough. Run it only after checking for case-variant duplicates, and note it also enforces uniqueness.
- **Cost:** about 150–300 MB on 4M rows. Writes to `customers` are probably low; please confirm.
- **Redundancy:** keep `customers_created_idx` unless `pg_stat_user_indexes.idx_scan` shows nothing else uses it.
- **Rollback:** `DROP INDEX CONCURRENTLY customers_lower_email_created_id_idx;`

**3. Optional:** the partial `blacklist (customer_id)` index from issue 7.

Build these on staging first, outside peak hours. I won't run anything against your database myself.

## Verify

On staging with production-like data, run this once with a typical email and once with the email that matches the most customers:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
<rewritten query with real parameter values>;

-- the plan the app gets as a prepared statement (PG 16+):
EXPLAIN (GENERIC_PLAN) <rewritten query with $1, $2, $3>;
```

A good plan looks like this:
- `Limit` → `Nested Loop Anti Join`, or a Hash Anti Join on `blacklist` (either is fine) → `Index Scan using customers_lower_email_created_id_idx`, with `Index Cond: lower(email) = …` and the keyset condition. There's **no `Sort` node** and no Seq Scan on `customers`.
- `Aggregate` → `Index Only Scan using orders_customer_id_created_at_idx` with `loops=20` (or fewer), and `Heap Fetches` close to 0. If Heap Fetches is high, `VACUUM orders`.
- Buffers in the hundreds, not millions. Latency in single-digit milliseconds.

Please send the plan from before and after the change and I'll compare execution time, buffers and rows read.

## Assumptions / open questions

I've given a complete answer above under these assumptions. The answers below would change specific parts:

1. **What `$1` really is.** Is the email filter always present, and are emails unique? If it's optional (an "all customers" report), the email index is the wrong lead column. You'd want `(created_at DESC, id DESC)` and keyset pagination, and I'd redesign around that.
2. **How "blocked emails" are stored.** `blacklist` has no email column, so how should a NULL-`customer_id` row block someone? Should this report exclude customers by email too?
3. **Does the UI need arbitrary page jumps?** Or is next/previous enough, which keyset pagination supports?
4. **Traffic.** How often does this endpoint run, what's the current and target latency, and how many inserts per second does `orders` get? Use `pg_stat_statements`, plus `n_tup_ins/upd/del` from `pg_stat_user_tables` sampled twice:
   ```sql
   SELECT calls, round(mean_exec_time::numeric,2) AS mean_ms, rows, query
   FROM pg_stat_statements WHERE query ILIKE '%blacklist%' ORDER BY total_exec_time DESC LIMIT 5;
   SELECT relname, n_live_tup, n_tup_ins, n_tup_upd, n_tup_del, last_autovacuum, last_autoanalyze
   FROM pg_stat_user_tables WHERE relname IN ('customers','orders','blacklist');
   ```
5. **Skew and existing indexes:**
   ```sql
   SELECT max(cnt), avg(cnt) FROM (SELECT customer_id, count(*) cnt FROM orders GROUP BY customer_id) s;  -- largest customer
   SELECT count(*) FROM blacklist WHERE customer_id IS NULL;   -- confirms the NOT IN bug
   \d+ customers
   \d+ orders
   ```
   I assumed no customer has millions of orders. If one does, `count(*)` for that customer is still a long index range, and a per-customer summary table maintained by trigger or batch job would be the next step.

Schemas, counts and plans are all I need. Please don't paste real customer data.
