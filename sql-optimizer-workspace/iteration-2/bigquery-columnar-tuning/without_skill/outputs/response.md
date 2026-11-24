## Short answer: BigQuery has no index that fixes this. Partition and cluster the table instead.

BigQuery doesn't have B-tree indexes. It does have `CREATE SEARCH INDEX` and vector indexes, but those serve `SEARCH()` text lookups and embeddings, not range-and-equality filters like yours. BigQuery is columnar, and the bill is **bytes scanned**. Today every run reads the *entire* `project_id`, `event_ts`, `event_name` and `user_id` columns for all ~9B rows. Your `WHERE` clause cuts down the rows you get back, but not the bytes you pay for.

Two table settings make your filters cut the bytes scanned:

| Your filter | Feature | What it skips |
|---|---|---|
| `event_ts` time window | **Partitioning** by `DATE(event_ts)` | Every day outside the window. 8 days out of all your history. |
| `project_id = '...'` | **Clustering** on `project_id` (first) | Storage blocks of the other ~299 projects inside those days. |

(The `properties` JSON column is already not being read, because you don't reference it. Columnar storage only scans the columns a query touches. Keep it that way and never `SELECT *` from this table in dashboards.)

### Why this matters at your refresh rate

2.3 TB × 96 refreshes/day ≈ **220 TB/day** for one tile. At on-demand list price (~$6.25/TiB in US multi-region) that's roughly **$1,300+/day, or about $40k/month**. After the change, a 7-day, single-project query should typically scan **a few GB to tens of GB**, not TB. That's often a 100–1000× reduction, depending on how many days of history you have and how skewed the projects are.

---

## Step 1: Rebuild the table partitioned + clustered

You can't add partitioning to an existing table in place, so create a new one and swap.

```sql
CREATE TABLE analytics.events_v2
PARTITION BY DATE(event_ts)
CLUSTER BY project_id, event_name, user_id
OPTIONS (
  require_partition_filter = TRUE,        -- blocks accidental full scans
  partition_expiration_days = NULL        -- set this if you have a retention policy
)
AS
SELECT * FROM analytics.events;
```

Notes:
- **This one-time copy scans the whole table, `properties` included.** That's likely your largest single bill. Run it once, preferably on a reservation/flat-rate slot if you have one. Still, it costs about the same as a handful of dashboard refreshes today.
- **Cluster order matters.** `project_id` goes first because nearly every query filters on it. `event_name` is second, which helps queries that filter or group by event. `user_id` is third and helps the `COUNT(DISTINCT user_id)` work and per-user lookups. You can list up to 4 columns.
- **Partition granularity.** Daily is right here. The 10,000-partitions-per-table limit gives ~27 years of daily partitions. Pick hourly only if most queries use windows of a few hours.
- **`require_partition_filter`** makes any query without a time filter fail rather than scan everything. Strongly recommended for a 9B-row table that Looker users can explore. Turn it on only after checking that your existing Looker explores always pass a date filter.
- **Swap:** stop the writers briefly (or dual-write), backfill any rows that landed during the copy, then:
  ```sql
  ALTER TABLE analytics.events RENAME TO events_old;
  ALTER TABLE analytics.events_v2 RENAME TO events;
  ```
  Then point ingestion at the new table. Keep `events_old` for a few days, then drop it to stop paying its storage.

New rows are clustered automatically. BigQuery re-clusters in the background for free.

## Step 2: Write the filter so pruning is guaranteed

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

- A plain range filter on the raw partitioning column (`event_ts >= ... AND event_ts < ...`) is the form BigQuery is guaranteed to prune on. `DATE(event_ts) BETWEEN ...` does generally prune when the table is partitioned by `DATE(event_ts)`, but the raw-column range is the safer habit and also works if you ever switch to `TIMESTAMP_TRUNC` / hourly partitioning.
- Keep the `project_id` filter a literal or query parameter, not something derived from a join or subquery. Clustering pruning works best with constant predicates.
- Heads up on semantics: your `BETWEEN today-7 AND today` covers **8 calendar days**, including the partial current day. If you meant "last 7 full days," use `>= today-7` and `< today`. Also, `CURRENT_DATE()` is UTC unless you pass a time zone, e.g. `CURRENT_DATE('America/New_York')`.
- In Looker, make sure the generated SQL keeps this shape. Use a `dimension_group` on `event_ts` with `datatype: timestamp`, plus an `always_filter` / `sql_always_where` on the date. Check the SQL tab of the generated query.

## Step 3: Verify with real numbers

- **Dry run before and after:** `bq query --dry_run --use_legacy_sql=false '...'`. The dry-run estimate reflects **partition** pruning but *not* clustering pruning, so it will still look high-ish.
- **Actual bytes billed** is the real number. After a refresh, check:
  ```sql
  SELECT creation_time, total_bytes_processed, total_bytes_billed, query
  FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
    AND query LIKE '%analytics.events%'
  ORDER BY total_bytes_billed DESC
  LIMIT 50;
  ```
  Use the same view to find the *other* expensive dashboards hitting this table. One 15-minute tile is rarely the only one.

---

## Optional, but big wins for a dashboard refreshed every 15 minutes

Partitioning + clustering is the essential fix. On top of it, consider:

1. **Refresh less often, or cache in Looker.** Does this tile really need 15-minute freshness over a 7-day window? A Looker `datagroup` with `max_cache_age` and a `sql_trigger` (e.g. `SELECT MAX(event_ts) ...` on today's partition only) avoids re-running when nothing changed. BigQuery's own 24h result cache won't save you here: queries using `CURRENT_DATE()`/`CURRENT_TIMESTAMP()`, and tables receiving streaming inserts, generally aren't served from cache.

2. **Pre-aggregate.** Most dashboard tiles only need daily counts per project. Build a small rollup table with a scheduled query, or a Looker aggregate table / PDT:
   ```sql
   -- daily rollup, partitioned by day, clustered by project
   CREATE TABLE analytics.events_daily
   PARTITION BY day
   CLUSTER BY project_id, event_name AS
   SELECT
     DATE(event_ts)                   AS day,
     project_id,
     event_name,
     COUNT(*)                         AS n,
     HLL_COUNT.INIT(user_id)          AS users_sketch
   FROM analytics.events
   WHERE event_ts >= TIMESTAMP('2000-01-01')   -- satisfies require_partition_filter
   GROUP BY 1, 2, 3;
   ```
   Dashboard query:
   ```sql
   SELECT event_name,
          SUM(n) AS n,
          HLL_COUNT.MERGE(users_sketch) AS users
   FROM analytics.events_daily
   WHERE project_id = 'prj_8812'
     AND day BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
   GROUP BY event_name
   ORDER BY n DESC;
   ```
   This scans kilobytes to megabytes. **Trade-off:** distinct users across days can't be summed exactly from daily rows, so `HLL_COUNT.MERGE` gives an approximate count, typically within ~1% error. If the dashboard needs exact distinct users, keep querying the partitioned/clustered base table, which is fine after Step 1. A BigQuery materialized view is the alternative. Note that incremental MVs support `APPROX_COUNT_DISTINCT`/HLL sketches but not exact `COUNT(DISTINCT)` over a moving window.

3. **Pricing model.** If many dashboards hit this data all day, compare on-demand against BigQuery Editions capacity (slot) pricing. With slots you pay for compute instead of bytes, which caps the bill. Do this *after* fixing the table layout, not instead of it.

4. **Consider pulling hot `properties` keys into typed columns.** If other dashboards do `JSON_VALUE(properties, '$.plan')` etc., every such query reads the whole (probably huge) JSON column. Frequently used keys are much cheaper as their own columns.

---

### TL;DR
- There's no index to add in BigQuery for this.
- Recreate `analytics.events` with `PARTITION BY DATE(event_ts)` and `CLUSTER BY project_id, event_name, user_id`, with `require_partition_filter = TRUE`.
- Filter on `event_ts` as a raw timestamp range plus `project_id = '...'`.
- Optionally, put a daily rollup and Looker caching in front of the dashboard.

That should take this tile from ~2.3 TB per run down to GBs or less.
