**Short answer:** only **one** of the five can be dropped based on what you've shown: `idx_order_items_order_id`, because `idx_order_items_order_product` already covers it. Even for that one, do two quick checks first (below). The other four should stay. Two of them would break correctness or make your nightly purge very slow, and the numbers for the other two don't prove they're unused.

The main problem is that **these counters don't support "unused" yet**:

1. **The window is 3 days.** `stats_reset` is 3 days ago because the instance change restarted the server. In PG 15, statistics survive a clean shutdown, but *"when starting from an unclean shutdown (e.g., after an immediate shutdown, a server crash, …) all statistics counters are reset"* ([PG 15 docs, Cumulative Statistics](https://www.postgresql.org/docs/15/monitoring-stats.html)). Three days misses every weekly, month-end and quarterly job. Most teams wait for at least one full business cycle, usually 30+ days, before calling an index unused.
2. **You only looked at the primary.** Each server keeps its own counters, so a hot standby's index scans never appear in the primary's `pg_stat_user_indexes`. Your reporting dashboards run on the replica. But dropping an index on the primary also drops it on the replica (physical replication replays the DROP). So an index can show "40 scans" on the primary and still be the main thing the dashboards rely on.

## Diagnosis

| Index | Primary `idx_scan` (3 days) | Verdict | Why |
|---|---|---|---|
| `idx_order_items_order_id` (3.4 GB) | 12 | **Drop (after checks)** | Redundant. `(order_id)` is the leftmost prefix of `idx_order_items_order_product (order_id, product_id)`. Anything that uses it, including FK and cascade lookups on `order_items.order_id`, can use the composite index instead. This saves 3.4 GB and one index write on every `order_items` INSERT/DELETE. |
| `idx_payments_order_id` (2.1 GB) | 0 | **Keep** | It is the only index on the FK column `payments.order_id`, which has `ON DELETE CASCADE` and a nightly purge. For every deleted `orders` row, PostgreSQL runs `DELETE FROM payments WHERE order_id = $1`. Without this index, each of those becomes a **sequential scan of `payments`**, so a purge of N orders reads the whole table N times. The purge would go from minutes to hours, hold locks longer and produce far more I/O, which would make your write latency worse. (Zero scans after three nightly purges is odd in itself, since the cascade lookups normally show up in `idx_scan`. Check that the purge actually ran since the reset and what it deleted. Either way, the index stays.) |
| `users_email_key` (410 MB) | 3 | **Keep (it can't be dropped anyway)** | It enforces `UNIQUE (users.email)`. The scan count doesn't matter: the index does its job on every INSERT/UPDATE whether or not anything reads it. `DROP INDEX` fails because the constraint needs it. The only way to remove it is `ALTER TABLE users DROP CONSTRAINT users_email_key`, which would allow duplicate emails. It's also not on an orders table, so dropping it wouldn't help your problem. |
| `idx_orders_created_at` (1.2 GB) | 40 | **Keep for now** | 40 scans in 3 days is real use. It's probably your nightly purge (`WHERE created_at < …`) or some dashboard-style query on the primary. Dashboards on the replica almost certainly filter or sort `orders` by `created_at`, and those scans aren't in your numbers. Dropping it could turn the purge and the reports into full scans of `orders`. |
| `idx_order_items_order_product` (4.0 GB) | 981,233 | **Keep** | This is your hot index, and it also stands in for the dropped `(order_id)` index. |

## Issues found

| # | Where | Problem | Severity | Why it matters | Fix |
|---|---|---|---|---|---|
| 1 | `idx_payments_order_id`, `idx_scan = 0` | The FK + `ON DELETE CASCADE` index looks unused | High | Dropping it makes every cascaded delete scan all of `payments`, and the nightly purge falls apart | Keep it. Don't let FK child indexes be chosen on `idx_scan` alone |
| 2 | `users_email_key` | The index backs a UNIQUE constraint | High (correctness) | Removing it allows duplicate emails, and `DROP INDEX` fails anyway | Keep it |
| 3 | `stats_reset` 3 days ago | The observation window is too short | High | Misses weekly and monthly jobs, so "unused" can't be shown | Re-check after 30+ days or a full business cycle |
| 4 | Query run on the primary only | Replica usage isn't counted | High | Dashboard indexes look unused on the primary, and a DROP there also removes them from the replica | Run the same query on the replica |
| 5 | `idx_order_items_order_id` vs `(order_id, product_id)` | Leftmost-prefix duplicate | Medium | Costs 3.4 GB of storage and cache, and slows every `order_items` write | Drop it (below) |

## Recommendation

### 1. Before dropping: two checks (read-only)

```sql
-- a) Is it backing a constraint? Expect 0 rows.
SELECT conname, contype FROM pg_constraint
WHERE conindid = 'idx_order_items_order_id'::regclass;

-- b) Are both indexes valid? Is the composite a plain btree with order_id first? Save the definitions for rollback.
SELECT i.indexrelid::regclass AS index, i.indisvalid, i.indisunique, pg_get_indexdef(i.indexrelid)
FROM pg_index i
WHERE i.indrelid = 'order_items'::regclass;
```

Run this on **the replica** too, to see what the dashboards use:

```sql
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE relname IN ('orders','order_items','payments','users')
ORDER BY relname, idx_scan;
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
```

If `idx_order_items_order_product` is valid, is a btree, and has `order_id` as its first column, then `idx_order_items_order_id` is safe to drop even if the replica uses it. The composite index serves the same lookups and is only about 15% larger, so the plans are practically the same.

### 2. The DROP statement

```sql
-- Run in a plain session (psql autocommit), NOT inside BEGIN ... COMMIT.
-- CONCURRENTLY avoids blocking reads/writes on order_items while it waits.
SET lock_timeout = '5s';   -- give up instead of queueing behind long transactions
DROP INDEX CONCURRENTLY IF EXISTS public.idx_order_items_order_id;
```

Notes for PG 15 ([DROP INDEX docs](https://www.postgresql.org/docs/15/sql-dropindex.html)): `CONCURRENTLY` can't run inside a transaction block, takes only one index name, and doesn't support `CASCADE`. If it times out, retry. If it gets interrupted, the index may be left `INVALID`. Check `indisvalid` and run the `DROP INDEX CONCURRENTLY` again.

- **Expected effect:** one less index to maintain on every `order_items` INSERT and DELETE (and on UPDATEs of `order_id`), 3.4 GB freed, and less shared_buffers and WAL pressure.
- **Cost:** queries that used it now use the 4.0 GB composite index, which is slightly bigger. In practice the difference is negligible.
- **Rollback:** PostgreSQL has no "invisible index", so the only way back is a rebuild: `CREATE INDEX CONCURRENTLY idx_order_items_order_id ON public.order_items (order_id);`. On a 3.4 GB index that takes a while and needs spare I/O and disk. Do it off-peak if you ever need it. (Don't use the `UPDATE pg_index SET indisvalid = false` trick. It isn't supported, and RDS doesn't allow it.)

### 3. Statements I would *not* run (shown so the outcome is clear)

```sql
-- DO NOT RUN: FK cascade index, needed by the nightly purge
-- DROP INDEX CONCURRENTLY idx_payments_order_id;

-- DO NOT RUN: fails ("cannot drop index ... because constraint ... requires it");
-- dropping the constraint instead would allow duplicate emails
-- DROP INDEX users_email_key;

-- NOT YET: re-evaluate after 30+ days with replica stats + purge query checked
-- DROP INDEX CONCURRENTLY idx_orders_created_at;
```

## Verify

After the drop, look at the plans for the queries that used the dropped index. Make sure they now use `idx_order_items_order_product`, not a `Seq Scan`:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM order_items WHERE order_id = 12345;
-- expected: Index Scan using idx_order_items_order_product on order_items
--           Index Cond: (order_id = 12345)
```

Also confirm the next nightly purge takes about as long as before, and compare `order_items` write latency before and after.

## About the write latency itself

Dropping one redundant index will help, but it may not be the main cause. Check these too:

- `pg_stat_statements`: which INSERT/UPDATE/DELETE statements have the highest `mean_exec_time`?
- **The nightly purge.** A big cascading `DELETE` causes WAL spikes, lock waits and vacuum debt. Deleting in batches (for example, 5–10k orders per transaction) usually smooths this out a lot.
- `n_tup_hot_upd` vs `n_tup_upd` in `pg_stat_user_tables`. If updates touch indexed columns, HOT updates are disabled and every index gets written.
- Lock waits (`pg_stat_activity.wait_event_type = 'Lock'`) and RDS storage I/O / IOPS credits.
- `hot_standby_feedback` or long-running replica queries holding back vacuum on the primary, which leads to bloat and slower writes.

## Assumptions / open questions

- I assumed `idx_order_items_order_product` is a valid, plain btree with `order_id` first, and that `idx_order_items_order_id` doesn't back a constraint. Check (a) and (b) above confirm both.
- I assumed the tables aren't partitioned. `DROP INDEX CONCURRENTLY` doesn't work on partitioned-table indexes.
- Please send the **replica's** `pg_stat_user_indexes` output, the nightly purge statement, and the top write statements from `pg_stat_statements`. With those I can tell you whether `idx_orders_created_at` can go later, and what else is driving the write latency.
