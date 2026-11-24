Short answer: of these five, only `idx_order_items_order_id` is safe to drop, and only once you've checked the replica. Keep the other four. A low `idx_scan` number on the primary doesn't show an index is unused in your setup, for three reasons:

1. **Your stats cover only 3 days.** `idx_scan` counts from the last stats reset, which was the instance upgrade 3 days ago. That window misses anything that runs weekly or monthly: month-end reports, finance exports, quarterly jobs, rare admin screens.
2. **The primary doesn't count replica reads.** `pg_stat_user_indexes` is tracked separately on each server. Your reporting dashboards run on the read replica, so their index scans never show up in what you ran on the primary. But the replica is a physical copy of the primary: drop an index on the primary and it disappears from the replica too.
3. **Some indexes have a job besides speeding up `SELECT`s.** They enforce constraints or support foreign-key actions.

## Verdict per index

| Index | Verdict | Why |
|---|---|---|
| `idx_payments_order_id` (0 scans, 2.1 GB) | **Keep** | It supports the `ON DELETE CASCADE` foreign key. Every time your nightly purge deletes an `orders` row, Postgres has to find the matching `payments` rows by `order_id`, and this is the only index that can do that lookup. Without it, each deleted order means a sequential scan of `payments`. Deleting N orders then costs about N full scans of a multi-GB table, so the purge would run far longer and hold locks far longer. Zero scans most likely means the purge hasn't actually deleted anything since the reset (worth checking that it's running), not that the index is useless. |
| `users_email_key` (3 scans, 410 MB) | **Keep** | It enforces the UNIQUE constraint on `users.email`. Postgres checks it on every insert or update of `email`, but those checks don't increase `idx_scan`. You also can't `DROP INDEX` it; you would have to drop the constraint, which would allow duplicate emails. It's on `users`, so it has nothing to do with your orders write latency anyway. |
| `idx_orders_created_at` (40 scans, 1.2 GB) | **Keep** | 40 scans in 3 days on the primary means something is using it. That's probably the nightly purge (`WHERE created_at < ...`), and dropping it would turn that into a full scan of `orders`. Date-range dashboard queries on the replica almost certainly use it too, and you can't see those scans from the primary. |
| `idx_order_items_order_id` (12 scans, 3.4 GB) | **Drop, after the checks below** | It's redundant. `idx_order_items_order_product (order_id, product_id)` starts with `order_id`, so it can serve every lookup that needs only `order_id`: equality, `IN`, ranges, joins, and the FK/cascade lookup from `orders`. Its 981k scans show the planner already uses it heavily. Queries that now use the single-column index will switch to the composite one, which costs only a bit more I/O. Dropping it removes 3.4 GB of index that every `order_items` insert currently has to maintain. |

## Checks before dropping `idx_order_items_order_id`

**1. Run the stats query on the replica too**, and look at `order_items` in particular:

```sql
-- run on the READ REPLICA
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size,
       d.stats_reset
FROM pg_stat_user_indexes s
CROSS JOIN pg_stat_database d
WHERE d.datname = current_database()
  AND s.relname IN ('orders','order_items','payments')
ORDER BY s.relname, s.idx_scan;
```

Even if the replica shows heavy use of `idx_order_items_order_id`, you can still drop it: the composite index covers the same lookups. The replica numbers mainly confirm that the other indexes are in use.

**2. Confirm the composite index is a valid, non-partial btree with `order_id` first, and that the single-column index doesn't back a constraint:**

```sql
SELECT c.relname AS index_name,
       pg_get_indexdef(i.indexrelid) AS definition,
       i.indisvalid, i.indisready, i.indisunique,
       i.indpred IS NOT NULL AS is_partial,
       con.conname AS backs_constraint
FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid
LEFT JOIN pg_constraint con ON con.conindid = i.indexrelid
WHERE i.indrelid = 'order_items'::regclass;
```

You want to see:
- `idx_order_items_order_product`: `indisvalid = true`, `is_partial = false`, and a definition of `USING btree (order_id, product_id)`.
- `idx_order_items_order_id`: `backs_constraint` is NULL.

**3. (Optional) Look for hard-coded references** to the index name in your code: query hints (e.g. `pg_hint_plan`), migrations, or monitoring that would break if it disappeared.

## The DROP statement

Run it on the **primary**; the change replicates to the replica automatically. `CONCURRENTLY` avoids taking an exclusive lock that would block writes to `order_items`. It can't run inside a transaction block, so turn off autocommit or BEGIN wrapping in your client or migration tool.

```sql
-- On the primary, outside a transaction block
SET lock_timeout = '5s';   -- give up instead of queueing behind long transactions; retry if it times out
DROP INDEX CONCURRENTLY IF EXISTS public.idx_order_items_order_id;
RESET lock_timeout;
```

If it fails partway, the index can be left behind as INVALID. Just run the `DROP INDEX CONCURRENTLY` again.

**Rollback**, if some query gets noticeably slower afterwards:

```sql
CREATE INDEX CONCURRENTLY idx_order_items_order_id ON public.order_items (order_id);
```

Rebuilding a 3.4 GB index takes a while and adds I/O load, so do it off-peak.

**After dropping**, watch `pg_stat_statements` and the dashboard response times (primary and replica) for a few days. Look for queries on `order_items` whose mean time went up.

## What I would not do yet

- **Don't drop anything else based on 3 days of stats.** If you still want to review the rest, wait until the stats cover at least a full business cycle (ideally 30 days or more, including a month-end). Then check `idx_scan` on **both** primary and replica.
- **Don't drop FK-supporting or unique indexes based on `idx_scan`.** It doesn't measure their real job.

## Other things to check

Dropping one index on `order_items` cuts that table's insert cost a bit, but on its own it probably won't fix "bad write latency". Other places to look:

- **HOT updates.** Run `SELECT relname, n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname IN ('orders','order_items','payments');`. A low HOT ratio on `orders` means updates rewrite every index on the table. That happens when updated columns are indexed or pages have no free space; lowering `fillfactor` (e.g. 90) can help.
- **Index bloat.** 3.4 GB for a single-column index is large. The nightly purge probably leaves bloat in `idx_order_items_order_product`, `idx_payments_order_id` and the `orders` indexes. Running `REINDEX INDEX CONCURRENTLY` on them can shrink them a lot and lower the cost of maintaining them.
- **The purge itself.** A large nightly `DELETE` with cascades makes WAL spikes, lock contention and vacuum pressure. Deleting in smaller batches, or partitioning `orders` and `order_items` by `created_at` so you can drop old partitions, usually helps writes more than dropping indexes.
- **RDS-level causes.** Check EBS IOPS/throughput limits (gp2 burst credits, gp3 provisioned IOPS), `WAL`/commit latency, and whether the new instance class changed storage throughput.
