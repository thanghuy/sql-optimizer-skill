Short answer: **don't add an index. Partition the table by day and cluster it by `project_id`.** In BigQuery, that's what fixes this.

## Why an index won't help

BigQuery has no B-tree indexes. It stores data by column and bills you for the bytes in every column a query touches, across every block it can't skip. Today the table has no partitions and no clustering, so BigQuery can't skip anything. Each run reads all ~9B rows of `project_id`, `event_ts`, `event_name` and `user_id`, even though you only want one project's last 7 days.

(BigQuery does have `CREATE SEARCH INDEX`, but it's built for needle-in-a-haystack text and JSON lookups, such as finding one `event_id`. It doesn't help a scan and aggregation like this one.)

The tools that cut bytes scanned are:

| Feature | What it skips | Fit here |
|---|---|---|
| **Partitioning** on `DATE(event_ts)` | Every day outside the time window | You always filter on a time window |
| **Clustering** on `project_id` (then `event_name`) | Storage blocks for other projects inside each day | You almost always filter on `project_id`, and there are about 300 of them |

Together they turn "scan the whole history for all 300 projects" into "scan about 8 days for one project."

## 1. Rebuild the table as partitioned and clustered

You can't add partitioning to an existing table, so create a new one and swap it in:

```sql
CREATE TABLE analytics.events_new
PARTITION BY DATE(event_ts)
CLUSTER BY project_id, event_name, user_id
OPTIONS (
  require_partition_filter = TRUE        -- stops anyone from scanning the full table again by accident
  -- , partition_expiration_days = 730   -- optional: if you don't need data forever
)
AS
SELECT * FROM analytics.events;
```

Notes:
- **Clustering order matters.** `project_id` goes first because it's the filter you nearly always use. `event_name` is next because you group by it. `user_id` last helps the `COUNT(DISTINCT user_id)` a little. Up to 4 clustering columns are allowed.
- **This rebuild is one full scan of the table** (properties JSON included, so it will be more than 2.3 TB). Do it once. If you're on on-demand pricing, that costs about the same as a few dashboard refreshes.
- **Repoint your ingestion** (streaming, Dataflow, load jobs, dbt model and so on) at the new table. Then swap the names: rename `events` to `events_old` and `events_new` to `events`, using `ALTER TABLE ... RENAME TO`. Backfill any rows that arrived during the cutover, and drop the old table once you've verified the new one.
- If the table is built by dbt, set `partition_by` and `cluster_by` in the model config rather than running hand-written DDL.
- Daily partitions are well within limits (BigQuery allows 10,000 partitions per table, roughly 27 years of daily partitions). If you have many years of history, `PARTITION BY TIMESTAMP_TRUNC(event_ts, MONTH)` is the alternative, but daily suits 7-day windows better.

## 2. Make the query prune cleanly (and fix a small bug)

```sql
SELECT
  event_name,
  COUNT(*)                AS n,
  COUNT(DISTINCT user_id) AS users
FROM analytics.events
WHERE project_id = 'prj_8812'
  AND event_ts >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
  AND event_ts <  TIMESTAMP(DATE_ADD(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY event_name
ORDER BY n DESC;
```

- `DATE(event_ts) BETWEEN ...` does prune partitions when the table is partitioned on `DATE(event_ts)`. A plain range on the raw timestamp, as above, is the most reliable way to get pruning, so make it your habit.
- **Off-by-one:** `BETWEEN today-7 AND today` covers **8** calendar days: 7 full days plus today so far. If you meant "last 7 days including today," use `INTERVAL 6 DAY`. Check what the dashboard is supposed to show.
- **Time zone:** `CURRENT_DATE()` and `DATE()` use UTC. If the dashboard should follow a local day, use `CURRENT_DATE('America/New_York')` and set the same zone in the bounds.
- Keep `properties` out of dashboard queries unless you actually need it. You already do, which is good: in a columnar store, columns you don't select cost nothing.

## Expected impact (rough, verify with a dry run)

- **Partitioning** cuts the scan to about 8 days divided by your total history. With 2 years of data, that's roughly 1% of the table.
- **Clustering on `project_id`** cuts it again by about this project's share of the rows in those days. With ~300 projects, a typical project might be well under 1% of the data. A very large project could be much more, since event data is usually skewed.
- Realistically, expect the 2.3 TB to fall to somewhere between **a few GB and a few tens of GB** per run.

For scale: 2.3 TB × 96 refreshes a day ≈ **220 TB/day**. At on-demand list price (about $6.25/TB in US multi-region) that's roughly **$1,400/day, or about $40k/month**, from this one tile.

One caveat on measuring: for clustered tables, the dry-run and "This query will process..." estimate is an **upper bound**. Clustering pruning happens at run time. Look at the actual *bytes billed* in the job details or `INFORMATION_SCHEMA.JOBS` after it runs.

## 3. Stop paying for every refresh

Even after the rebuild, this query is a good candidate to cache or pre-aggregate:

1. **Your query can't use BigQuery's free result cache.** Queries that call `CURRENT_DATE()` or `CURRENT_TIMESTAMP()` are never cached, so each of the 96 daily refreshes runs and bills in full. Two fixes:
   - **Looker caching:** set a `datagroup` with a `max_cache_age`, or a `sql_trigger` that checks `MAX(event_ts)` or the table's last-modified time. The dashboard then re-queries only when data has changed, or on a schedule you choose. Ask whether a 7-day trend really needs refreshing every 15 minutes; hourly is often fine.
   - Or have Looker inject literal dates instead of `CURRENT_DATE()`, so identical queries can hit BigQuery's cache. Streaming inserts into the table still invalidate that cache.
2. **Pre-aggregate.** Build a small daily rollup by project, day and event name, and point the dashboard at it:
   ```sql
   CREATE TABLE analytics.events_daily
   PARTITION BY day
   CLUSTER BY project_id, event_name AS
   SELECT
     DATE(event_ts)              AS day,
     project_id,
     event_name,
     COUNT(*)                    AS n,
     HLL_COUNT.INIT(user_id)     AS users_sketch   -- distinct users can't be summed across days; sketches can be merged
   FROM analytics.events
   WHERE event_ts >= TIMESTAMP('2000-01-01')        -- needed because of require_partition_filter
   GROUP BY 1, 2, 3;
   ```
   The dashboard query becomes:
   ```sql
   SELECT event_name, SUM(n) AS n, HLL_COUNT.MERGE(users_sketch) AS users
   FROM analytics.events_daily
   WHERE project_id = 'prj_8812'
     AND day BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
   GROUP BY event_name
   ORDER BY n DESC;
   ```
   That reads kilobytes to megabytes. Keep the rollup current with a scheduled query, or with an incremental dbt model that rebuilds today and yesterday. You could also use a **materialized view**: BigQuery refreshes it incrementally and can rewrite matching queries to use it automatically.
   - **Trade-off:** `HLL_COUNT` gives an *approximate* distinct count, typically within about 1% at the default precision. If you need exact distinct users, roll up to `(day, project_id, event_name, user_id)` instead and count distinct over that. It's bigger, but still far smaller than the raw table.
3. **Optional:** if you have many dashboards like this, BI Engine reservations or a move from on-demand to capacity (slot) pricing can cap the bill. Partitioning and clustering are the fix you need first, though.

## Order of operations

1. Rebuild `analytics.events` partitioned by `DATE(event_ts)` and clustered by `project_id, event_name, user_id`, with `require_partition_filter`.
2. Rewrite the filter as a raw `event_ts` range and fix the 7-versus-8-day window.
3. Set a Looker datagroup so the dashboard isn't billed 96 times a day.
4. For dashboard tiles that hit this table constantly, add a daily rollup or materialized view.

Step 1 alone should cut this query's cost by about two orders of magnitude or more. Steps 3 and 4 bring the rest close to zero. Look across your other ~300-project queries too: they filter the same way, so they all benefit from step 1 automatically.
