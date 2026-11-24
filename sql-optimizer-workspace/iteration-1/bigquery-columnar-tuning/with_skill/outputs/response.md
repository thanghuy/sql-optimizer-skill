Short answer: **BigQuery doesn't have B-tree indexes, so there's no index to add that fixes this.** What fixes it is the table's physical layout: **partition by `DATE(event_ts)` and cluster by `project_id`** (plus `event_name`), then make one small change to the query. With your access pattern ("almost every query filters on project_id and a time window"), this should cut the scan from ~2.3 TB to a few GB or less per run. A pre-aggregated table can bring it down to MB.

(I checked the current BigQuery docs on partition pruning and materialized views. Links are at the end.)

---

## Diagnosis

- **What it costs now.** BigQuery bills on-demand queries by bytes read from the columns you reference. This query reads 4 columns (`project_id`, `event_ts`, `event_name`, `user_id`) across **all 9B rows**. It has to, because the table has no partitioning or clustering, so neither filter can skip any storage. `properties` JSON isn't read, which is why you're at 2.3 TB and not more.
- **What it costs per month.** Looker runs it every 15 minutes, which is 96 runs/day × ~2.3 TB ≈ **~220 TB/day**. At the US on-demand list price (~$6.25/TiB, check your region and contract) that's roughly **$1.2–1.3k per day, or ~$38k per month, for this one tile.** If the dashboard runs once per project, or has several tiles like this, multiply accordingly.
- **Why an index is the wrong tool.** BigQuery's "search indexes" are for needle-in-haystack lookups (`SEARCH()`, `=` on high-cardinality strings, such as finding one `event_id`). They aren't for "aggregate one project over 7 days". The equivalent of an index here:
  - **Range filter (the time window) → partition column.**
  - **Equality filter (`project_id`) → first clustering column.**
  - The GROUP BY column (`event_name`) → second clustering column, which helps the aggregation a little.

## Recommendation

### 1. Rebuild the table as partitioned and clustered (the main fix)

You can't add partitioning to an existing table in place, so you create a new table and swap it in:

```sql
CREATE TABLE analytics.events_v2
PARTITION BY DATE(event_ts)
CLUSTER BY project_id, event_name
OPTIONS (
  require_partition_filter = TRUE        -- block accidental full scans
  -- , partition_expiration_days = 730   -- only if you have a retention policy
)
AS
SELECT * FROM analytics.events;
```

Then cut over during a quiet window:

1. Pause or point the writers (streaming / load jobs) at `events_v2`, or dual-write for a short time.
2. Backfill any rows that arrived after the CTAS ran (`INSERT ... SELECT ... WHERE event_ts > <ctas snapshot time>`).
3. `ALTER TABLE analytics.events RENAME TO events_old;` then `ALTER TABLE analytics.events_v2 RENAME TO events;`
4. Keep `events_old` for a few days as a rollback, then drop it.

**Layout mapping:**

| Column | Role in the query | Placement | Why |
|---|---|---|---|
| `event_ts` | Range (time window) | Partition: `DATE(event_ts)`, daily | Almost every query has it. Pruning partitions skips whole days of storage. |
| `project_id` | Equality | Cluster column #1 | Filtered on almost every query. Pruning works best on the first clustering column. With ~300 projects, a single project is a small slice of each day. |
| `event_name` | GROUP BY | Cluster column #2 | Rows are co-located by event name, so less shuffle work. Also helps queries that filter `project_id = ? AND event_name = ?`. |

Optional: if you often filter or join on `user_id` within a project, add it as cluster column #3. Clustering order matters, and later columns only prune when the earlier ones are also filtered.

**Expected effect.** This is an estimate, because I don't know your retention yet. Say the table holds about 2 years: 9B rows / ~730 days ≈ 12M rows/day.
- Partition pruning alone: 8 of ~730 days ≈ 1.1% of 2.3 TB ≈ **~25 GB**.
- Clustering on `project_id`: if projects were evenly sized, 1/300 of that would be **under 1 GB**. In practice, large projects read more and small ones read close to the minimum block size.
- Overall, roughly **100× to over 1000× fewer bytes**, and the dashboard also gets faster.

**Cost of the change:**
- The CTAS scans the whole table once (all columns, including `properties`). That's roughly what 1–2 dashboard refreshes cost today.
- Storage is duplicated until you drop `events_old`.
- Automatic re-clustering in the background is free. Freshly streamed data may be slightly less well clustered until it runs.
- `require_partition_filter = TRUE` makes every query without a filter on `event_ts` **fail**, including views and other dashboards. Run step 1 of "Verify" first to find those queries, or leave this option off at the start and enable it later.

**Check the partition size.** BigQuery works best when partitions average roughly 1 GB or more. If a day is much smaller than that, use `PARTITION BY TIMESTAMP_TRUNC(event_ts, MONTH)` and add `event_ts` as a clustering column, or keep daily partitions if your time windows are usually days. Daily partitioning also has a per-table partition count limit (currently 10,000, check the quotas page). That covers about 27 years of daily data, so it's fine here.

**Rollback:** rename `events_old` back to `events`.

### 2. Rewrite the filter as a plain range on the column

```sql
SELECT event_name, COUNT(*) AS n, COUNT(DISTINCT user_id) AS users
FROM analytics.events
WHERE project_id = 'prj_8812'
  AND event_ts >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
  AND event_ts <  TIMESTAMP(DATE_ADD(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY event_name
ORDER BY n DESC;
```

- The docs say pruning works for `DATE(col) = ...` on a column partitioned by `DATE(col)`, and recommend a dry run to confirm other forms. A bare range on `event_ts` is guaranteed to prune and also helps any clustering on `event_ts`.
- Check that the window is what you intended. `BETWEEN today-7 AND today` is **8 calendar days** (7 full days plus today so far), in **UTC**. If your dashboard is in another timezone, use `CURRENT_DATE('America/New_York')` and the matching `TIMESTAMP(..., 'tz')`.
- `ORDER BY n DESC` without a LIMIT is fine here. The result is only a few hundred event names.

### 3. For a dashboard that refreshes every 15 minutes, pre-aggregate it

Even after step 1, you're re-reading 8 days of raw events 96 times a day to recompute numbers that barely change. Keep a daily rollup instead.

`COUNT(DISTINCT user_id)` can't be summed across days, so you need one of these two:

**a) Approximate distinct users with an incremental materialized view.** `HLL_COUNT.INIT` and `COUNT` are supported in incremental MVs. Exact `COUNT(DISTINCT)` is not.

```sql
CREATE MATERIALIZED VIEW analytics.events_daily_mv
PARTITION BY event_date
CLUSTER BY project_id, event_name
AS
SELECT DATE(event_ts) AS event_date, project_id, event_name,
       COUNT(*) AS n,
       HLL_COUNT.INIT(user_id) AS users_sketch
FROM analytics.events
GROUP BY event_date, project_id, event_name;
```

The dashboard query then becomes:

```sql
SELECT event_name, SUM(n) AS n, HLL_COUNT.MERGE(users_sketch) AS users
FROM analytics.events_daily_mv
WHERE project_id = 'prj_8812'
  AND event_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
GROUP BY event_name
ORDER BY n DESC;
```

This reads KB to MB per run. Distinct counts are within about 1% at the default precision. The MV refreshes incrementally, and today's partition is combined with fresh base-table data automatically.

**b) If you need exact distinct users,** use a scheduled query (or an incremental Looker PDT/dbt model) that maintains a daily table of `(event_date, project_id, event_name, user_id)` with one row per distinct user per day. The dashboard then runs `COUNT(*)`-style sums plus `COUNT(DISTINCT user_id)` over that much smaller table, which should still be far smaller than raw events.

Also on the Looker side:
- Set a datagroup or caching policy so the tile doesn't re-query faster than the data actually needs to be fresh.
- BigQuery's own result cache won't help here, because queries using `CURRENT_DATE()` aren't cached.
- BI Engine is an option if you need sub-second response.

## Verify

1. **Find every query that will be affected** (and the next most expensive ones):
   ```sql
   SELECT query_info.query_hashes.normalized_literals AS qh, COUNT(*) AS runs,
          SUM(total_bytes_billed)/POW(1024,4) AS tib_billed, ANY_VALUE(query) AS sample
   FROM `region-us`.INFORMATION_SCHEMA.JOBS
   WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
     AND job_type = 'QUERY'
     AND REGEXP_CONTAINS(query, r'analytics\.events')
   GROUP BY qh ORDER BY tib_billed DESC LIMIT 30;
   ```
2. **Dry run before and after:** `bq query --dry_run --use_legacy_sql=false '<query>'`. After partitioning, the estimate should drop to about 8 days' worth of those 4 columns. Note that **dry run does not account for clustering pruning**, so it overstates the cost on clustered tables.
3. **Measure the actual bytes.** Run the query once with `--nouse_cache`, then check `total_bytes_processed` and `total_bytes_billed` in `INFORMATION_SCHEMA.JOBS` for that `job_id`. Good result: a few GB or less, versus 2.3 TB now. Also look at **Execution details**: the input stage's records read should be close to that project's 8-day row count, not 9B.
4. If bytes are still high, send me the `job_id` stage stats (`job_stages`: `records_read`, `slot_ms`, `shuffle_output_bytes_spilled`) and I'll look at what isn't pruning.

## Assumptions / open questions

These numbers would sharpen the estimates above. If you don't have them, the recommendation still stands.

1. **Size and retention:**
   ```sql
   SELECT total_rows, total_logical_bytes/POW(1024,4) AS tib
   FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE
   WHERE table_schema='analytics' AND table_name='events';
   SELECT MIN(event_ts), MAX(event_ts) FROM analytics.events;  -- reads one column only
   ```
2. **Project skew.** Are a few projects most of the data? (`SELECT project_id, COUNT(*) ... GROUP BY 1 ORDER BY 2 DESC LIMIT 20`)
3. **How data arrives:** streaming (Storage Write API or `insertAll`), batch loads, or frequent DML? This affects the cutover plan and how fresh the clustering stays.
4. **Pricing model:** on-demand (per TB) or editions/slots? On-demand makes the bytes savings direct money. On slots, the gain shows up as less slot time and less contention.
5. **Exact or approximate distinct users:** is about 1% error acceptable? That decides option 3a or 3b.
6. **Timezone and window:** is the 8-day UTC window intended?

Docs referenced:
- Querying partitioned tables / pruning: https://cloud.google.com/bigquery/docs/querying-partitioned-tables
- Clustered tables: https://cloud.google.com/bigquery/docs/clustered-tables
- Materialized views (supported aggregates): https://cloud.google.com/bigquery/docs/materialized-views-create
- Search indexes (and why they don't fit here): https://cloud.google.com/bigquery/docs/search-index
