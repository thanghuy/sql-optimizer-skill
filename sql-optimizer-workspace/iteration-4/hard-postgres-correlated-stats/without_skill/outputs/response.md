**Short answer: no, don't start with a composite index.** Your indexes and your stats are fine on their own. The planner is badly wrong about how many customers match, and a new index won't fix that. Teach it that `city` and `country` are correlated, using extended statistics.

## What's going wrong

Look at the estimate on the customers scan: **112 rows estimated, 479,812 actual**. That's about 4,300x too low.

The planner estimates each column on its own (both are accurate: 480,100 for Hanoi and 2,100,500 for VN). It then assumes the two columns are **independent** and multiplies their selectivities:

```
9,000,000 × (480,100 / 9,000,000) × (2,100,500 / 9,000,000) ≈ 112
```

That assumption fails here. Every Hanoi customer is in VN, so `country = 'VN'` filters out nothing once `city = 'Hanoi'` holds. The real selectivity is just that of `city = 'Hanoi'`.

Because it expects only 112 customers (about 1,700 orders), the planner chooses a **nested loop with 112 index probes into `orders`**. That looks cheap at an estimated cost of about 2,051. What actually runs is **479,812 probes**, which return 7.4M order rows scattered across a 140M-row table:

```
Buffers: shared hit=1,203,411  read=6,211,040
```

That's about 6.2M random 8 KB page reads (around 47 GB of random I/O) on the inner side. Those reads are your 9 minutes. The 2-second version almost certainly used a hash join (or a merge join) driven by a correct-sized customer set.

The recent autoanalyze doesn't help. Plain `ANALYZE` only collects **per-column** statistics. PostgreSQL never learns about cross-column correlation unless you ask it to with `CREATE STATISTICS`. So the analyze kept producing the same accurate single-column stats and the same bad combined estimate. The regression probably came from a small change that pushed the costs across a threshold: data growth, a slightly different analyze sample, or the `country` predicate being added to the report.

## The fix: extended statistics

```sql
CREATE STATISTICS customers_city_country_stx (dependencies, mcv)
    ON city, country FROM customers;

ANALYZE customers;   -- extended stats are only populated by ANALYZE
```

- `dependencies` records that `city → country` is a functional dependency (close to 1.0 here). PostgreSQL then stops multiplying the two equality selectivities.
- `mcv` stores the most common `(city, country)` combinations with their actual frequencies. `('Hanoi','VN')` is a very common pair, so the estimate will be close to exact. MCV lists also handle cases that dependencies can't, such as `IN` lists and mixed conditions.

Check the stats and the new estimate:

```sql
SELECT statistics_name, dependencies, most_common_vals
FROM pg_stats_ext
WHERE statistics_name = 'customers_city_country_stx';

EXPLAIN (ANALYZE, BUFFERS) <your query>;
```

The customers node should now estimate roughly 480k rows. The plan should switch to a **Hash Join**, probably a hash on the customer set with `orders` read sequentially, possibly in parallel. A sequential read of 140M rows is far cheaper than 6M random page reads, so expect run time back near the old 2s range (depending on table size and hardware).

From now on, autovacuum's autoanalyze keeps these statistics up to date automatically.

## Why a composite index isn't the answer

An index on `(city, country)`:

- **does not fix the row estimate.** PostgreSQL doesn't use a multi-column B-tree index to estimate the selectivity of combined conditions (only expression indexes get their own stats). The estimate stays at about 112, so the planner still picks the same nested loop.
- **only speeds up the cheap part.** The `BitmapAnd` on customers is not where the time goes. The 479,812 index probes into `orders` are.

The planner would scan customers a little faster, then make exactly the same mistake.

## Other options

1. **Drop the redundant predicate.** If `city = 'Hanoi'` always implies `country = 'VN'` in your data model, `WHERE c.city = 'Hanoi'` alone gives the same result and the right estimate today. However, the stats object is the more robust fix: it also protects other queries that filter on both columns. If city names aren't globally unique (another country could have a "Hanoi"), keep the predicate and rely on the stats object.

2. **A covering index on orders.** This is optional and a separate improvement. If the report still has to probe `orders` per customer in some plans, this index allows index-only scans (with a well-vacuumed visibility map) and avoids heap fetches:
   ```sql
   CREATE INDEX CONCURRENTLY orders_customer_id_total_idx
       ON orders (customer_id) INCLUDE (total);
   ```
   It adds write overhead to a 140M-row table, so only add it if the hash-join plan isn't fast enough.

3. **Aggregate before joining.** Grouping `orders` first, then joining, sometimes produces a more stable plan for reports like this:
   ```sql
   SELECT c.id, c.name, o.sum_total
   FROM customers c
   JOIN (SELECT customer_id, SUM(total) AS sum_total
         FROM orders GROUP BY customer_id) o ON o.customer_id = c.id
   WHERE c.city = 'Hanoi' AND c.country = 'VN';
   ```
   This aggregates all 140M orders, not just Hanoi's 7.4M, so it can be slower. Treat it as a fallback, not the first choice.

4. **Avoid global planner knobs.** `SET enable_nestloop = off` will "fix" this one query, but it treats a symptom. Applied globally it can hurt other queries. Use it only as a quick test to confirm the diagnosis, inside a transaction with `SET LOCAL`.

## Summary

| Action | Fixes the 9 min? | Notes |
|---|---|---|
| `CREATE STATISTICS ... (dependencies, mcv) ON city, country` + `ANALYZE` | **Yes** | Fixes the root cause (independence assumption) |
| Remove `country = 'VN'` | Yes, for this query | Only if `city` alone fully identifies the location |
| Composite index `(city, country)` | **No** | Estimate stays ~112, same bad nested loop |
| `orders (customer_id) INCLUDE (total)` | Partial help | Optional; reduces heap reads on the orders side |
| `enable_nestloop = off` | Masks it | Use only for diagnosis |

If the plan still isn't right after you create the statistics and run `ANALYZE`, send me the new full `EXPLAIN (ANALYZE, BUFFERS)` along with `pg_size_pretty(pg_total_relation_size('orders'))`, `work_mem`, and `random_page_cost`. Those settings decide whether the planner prefers the hash join.
