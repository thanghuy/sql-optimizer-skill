Short answer: **BigQuery has no `CREATE INDEX` that will fix this.** There are no B-tree indexes. You pay for the bytes of the columns a query reads, and the only way to read fewer bytes for this query is to lay the table out physically so BigQuery can skip storage blocks:

- **Partition** it by the time range you always filter on (`DATE(event_ts)`).
- **Cluster** it by the equality filter you always use (`project_id`), then by the `GROUP BY` column (`event_name`).

That is the BigQuery version of "index on (project_id, event_ts)". (BigQuery *search indexes* do exist, but they are built for `SEARCH()` and point lookups on high-cardinality text. They do not help a 7-day aggregate.)

Below are the diagnosis, the DDL, a migration plan, a cost estimate, and how to verify it. I could not see your table metadata, so the numbers marked *assumed* need your real values (the commands are at the end).

---

## Diagnosis

1. **No partition, no cluster means a full-column scan on every run.** The query reads 4 columns (`project_id`, `event_ts`, `event_name`, `user_id`) across all ~9 B rows, which is where the ~2.3 TB comes from. The `WHERE` clause cannot skip anything, because nothing in the storage layout matches it. `LIMIT` would not help either: BigQuery bills the bytes it scans, not the rows it returns.
2. **The bill comes from frequency times size.** A refresh every 15 min is 96 runs/day, so 96 × 2.3 TB ≈ **220 TB/day (~6.6 PB/month)**. On on-demand pricing (about $6.25 per TiB in US multi-region; check your region on the [pricing page](https://cloud.google.com/bigquery/pricing)) that is about **$1,250/day, or ~$37k/month, for this one tile.** If Looker sends the query once per project or per user filter, multiply by that too.
3. **The result cache cannot save you.** The query uses `CURRENT_DATE()`, and BigQuery does not cache results for queries with non-deterministic functions. So every refresh pays the full scan.
4. **The query is fine for pruning once the table is partitioned.** BigQuery's partition-pruning docs list `DATE(ts_col)` on a DAY-partitioned TIMESTAMP column as a form that prunes ([querying partitioned tables](https://cloud.google.com/bigquery/docs/querying-partitioned-tables)). `CURRENT_DATE()` counts as a constant for pruning. I still show a bare-column range form below, which is the safest form and easy to check with a dry run.
   - Side note: `BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()` covers **8 calendar days** (7 full days plus today so far), all in **UTC**. If the dashboard means "last 7 days" in another time zone, fix that now, or the numbers will differ slightly from what people expect.

## Recommendation

### 1. Recreate the table partitioned and clustered (this is "the index")

You cannot add partitioning to an existing table in place, so you create a new table and swap it in:

```sql
CREATE TABLE analytics.events_v2
PARTITION BY DATE(event_ts)
CLUSTER BY project_id, event_name
OPTIONS (
  require_partition_filter = TRUE,
  -- partition_expiration_days = 400,   -- optional: only if you have a retention policy
  description = 'events, partitioned by day, clustered by project_id, event_name'
)
AS
SELECT * FROM analytics.events;
```

How each part maps to the ESR rule (Equality → Sort → Range):

| Query part | ESR role | BigQuery feature | Why |
|---|---|---|---|
| `DATE(event_ts) BETWEEN …` | **R** (range on almost every query) | **Partition** by `DATE(event_ts)` | Whole days outside the window are never read. |
| `project_id = 'prj_8812'` | **E** (on almost every query) | **Cluster column #1** | Blocks are sorted by `project_id` inside each partition, so blocks for the other ~299 projects are skipped. The first cluster column prunes best, so it must be the one every query filters on. |
| `GROUP BY event_name` | **S** (group by counts as sort) | **Cluster column #2** | Rows for one project are stored grouped by `event_name`, which helps the aggregation and any `event_name = …` filters on other tiles. |
| `user_id` | (aggregated only) | not clustered | `COUNT(DISTINCT user_id)` gains nothing from clustering here. Add it as a 3rd column only if other queries filter on `user_id = …`. |

Why each option is there:
- **`require_partition_filter = TRUE`** makes any query without a usable `event_ts` filter fail instead of silently scanning ~2+ TB. With 300 projects and many ad-hoc Looker explores, this is your guardrail against the next exploding bill. It also applies to views built on the table. Any Looker explore without a date filter will start erroring, so add a default/required date filter to the explores (`always_filter` / `conditionally_filter` in LookML) before you swap.
- **DAY partitioning** is a reasonable granularity at ~9 B rows. Check the size of one day with the command at the end. Aim for partitions of roughly ≥ 1 GB on average. If days are much smaller, use `PARTITION BY TIMESTAMP_TRUNC(event_ts, MONTH)` and add `event_ts` to the clustering. Limits: 10,000 partitions per table, and a single job can modify at most 4,000 partitions. If your history is longer than ~11 years of days, run the CTAS in date chunks.
- **Clustering costs nothing extra to maintain.** BigQuery re-clusters new data in the background for free. Freshly streamed data may be less well clustered for a short while.

**Optional query tweak (bare column, safest pruning):**

```sql
SELECT event_name, COUNT(*) AS n, COUNT(DISTINCT user_id) AS users
FROM analytics.events
WHERE project_id = 'prj_8812'
  AND event_ts >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
  AND event_ts <  TIMESTAMP(DATE_ADD(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY event_name
ORDER BY n DESC;
```

This returns the same rows as your version (8 UTC days), but filters on the bare column. Your `DATE(event_ts)` form should also prune, so confirm with a dry run (see Verify) and keep whichever form Looker generates naturally. The `ORDER BY` without a `LIMIT` is fine here because the result is at most a few hundred `event_name` rows.

### 2. Migration path (zero/low downtime)

1. **Check size and cost first:**
   `bq query --dry_run --use_legacy_sql=false 'SELECT * FROM analytics.events'`
   The CTAS reads **every column of every row**, including `properties JSON`, so it will cost more than 2.3 TB once. That is a one-time charge, roughly what ~1–3 dashboard runs cost today.
2. **Create `analytics.events_v2`** with the DDL above, ideally outside peak hours. Record `max(event_ts)` (or your load watermark) at the time of the copy.
3. **Point the writers** (streaming / batch loads / Dataflow / etc.) at `events_v2`. Or pause them briefly, so you can backfill the gap:
   ```sql
   INSERT INTO analytics.events_v2
   SELECT * FROM analytics.events
   WHERE event_ts > TIMESTAMP '<watermark from step 2>';
   ```
   (If late-arriving events are possible, dedupe on `event_id`.)
4. **Validate:** compare row counts per day for the last ~30 days between old and new. Then run the dashboard query on both and compare the results.
5. **Swap names:**
   ```sql
   ALTER TABLE analytics.events RENAME TO events_old;
   ALTER TABLE analytics.events_v2 RENAME TO events;
   ```
   (Renaming can fail while a table has an active streaming buffer. Do it after the writers have switched and the buffer has drained.) Re-apply any IAM grants, authorized views or policy tags that were set on the old table.
6. **Rollback:** rename back (`events` → `events_v2`, `events_old` → `events`). Keep `events_old` for a week or two and drop it only once you are confident. Until you drop it, you pay storage for both tables.

### 3. Bigger win for a dashboard refreshed every 15 min: pre-aggregate

Even after partitioning, every refresh re-reads 8 days of raw rows. Dashboards like this are usually better served by a small daily rollup. One catch: **an incremental materialized view does not support exact `COUNT(DISTINCT)` or `CURRENT_DATE()`.** The docs list `APPROX_COUNT_DISTINCT` and `HLL_COUNT.INIT` as supported aggregates, and non-deterministic functions such as `CURRENT_DATE()` as unsupported ([create materialized views](https://cloud.google.com/bigquery/docs/materialized-views-create)). So pick one of these:

**Option A: materialized view with HLL sketches (approximate distinct users, ~1% error):**

```sql
CREATE MATERIALIZED VIEW analytics.mv_events_daily
PARTITION BY event_date
CLUSTER BY project_id, event_name
AS
SELECT
  DATE(event_ts)             AS event_date,
  project_id,
  event_name,
  COUNT(*)                   AS n,
  HLL_COUNT.INIT(user_id)    AS users_sketch
FROM analytics.events          -- the new partitioned table
GROUP BY event_date, project_id, event_name;
```

Dashboard query (reads a few MB instead of TB):

```sql
SELECT event_name,
       SUM(n)                          AS n,
       HLL_COUNT.MERGE(users_sketch)   AS users   -- distinct users across the 8 days, approx.
FROM analytics.mv_events_daily
WHERE project_id = 'prj_8812'
  AND event_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
GROUP BY event_name
ORDER BY n DESC;
```

Sketches merge correctly across days, so users counted on several days are not double-counted. BigQuery refreshes the view incrementally and also combines it with the base table's newest changes, so results stay fresh. Because the base table has `require_partition_filter`, test that creating and refreshing the view works in your project, since the requirement also applies to views on the table. If it does not, create the view before switching that option on, or keep the option off and enforce date filters in Looker.

**Option B: exact distinct users.** Use a scheduled query (e.g. hourly) that writes a rollup table at `(event_date, project_id, event_name, user_id)` grain, or a non-incremental materialized view (`allow_non_incremental_definition = TRUE` plus a `max_staleness`), which is fully recomputed on refresh. These are bigger and more expensive than Option A, but exact.

Also on the Looker side: a 15-minute refresh is very aggressive for a 7-day window. A datagroup / persistent derived table refreshed hourly (or when new data lands) divides the run count by 4 for free. BI Engine can also accelerate the small rollup table.

## Expected effect (estimate; replace with your real numbers)

*Assumed:* the table holds ~2 years (≈730 days) of fairly evenly spread data, and the 300 projects are not extremely skewed.

| Step | Bytes per run | Runs/day × bytes | Approx. on-demand cost/day |
|---|---|---|---|
| Today (no partition/cluster) | ~2.3 TB | ~220 TB | ~$1,250 |
| Partition by day (8 of ~730 days) | ~25 GB | ~2.4 TB | ~$14 |
| + cluster by `project_id` (1 of ~300 projects, realistically 1/50–1/300 because of block granularity and skew) | ~0.1–0.5 GB | ~10–50 GB | < $1 |
| + daily MV rollup (Option A) | a few MB (10 MB minimum billed) | < 1 GB | ~$0 |

So roughly **~99% from partitioning alone, and ≥ 99.9% with clustering.** The real factor depends on how many days the table holds and how large `prj_8812` is compared with the other projects. A big tenant that makes up 20% of events prunes much less than a small one. With a dry run you can see the exact *upper bound* (clustering pruning only shows up after the query runs).

## Verify

1. **Before/after, without the cache** (the `CURRENT_DATE()` form is never cached anyway):
   ```bash
   bq query --dry_run --use_legacy_sql=false '<dashboard query>'
   bq query --nouse_cache --use_legacy_sql=false '<dashboard query>'
   ```
   The dry run estimate after partitioning should be about 8/N of the table, where N is the number of days. The actual `total_bytes_processed` after the run should be much lower still, because the dry run does not account for cluster pruning.
2. **Measured from job history:**
   ```sql
   SELECT creation_time, total_bytes_processed/POW(1024,3) AS gb_processed,
          total_bytes_billed/POW(1024,3) AS gb_billed, total_slot_ms, cache_hit
   FROM `region-us`.INFORMATION_SCHEMA.JOBS
   WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
     AND job_type = 'QUERY'
     AND query LIKE '%FROM analytics.events%GROUP BY event_name%'
   ORDER BY creation_time DESC;
   ```
   A good result is GB, not TB, for `gb_processed`, and a big drop in `total_slot_ms`.
3. **Execution details / graph** in the console: the first stage's *records read* should be close to the number of `prj_8812` rows in the 8 days, not 9 B. If bytes are still near the whole table, the date filter is not pruning (for example, Looker wraps the column in a different function or casts it). Paste the generated SQL and the job's stage stats and I'll look again.

## Assumptions / open questions

Please send these, ideally together, and I'll firm up the numbers. Rough values are fine.

1. **Billing model:** on-demand (pay per TB scanned) or Editions/capacity (slots)? On Editions, the win shows up as slot time and concurrency rather than a per-TB bill, but it is just as large.
2. **History and partition size:**
   ```sql
   SELECT MIN(event_ts), MAX(event_ts), COUNT(*) FROM analytics.events;  -- ~one more full scan of event_ts only, cheap-ish
   SELECT total_rows, total_logical_bytes/POW(1024,4) AS tib
   FROM `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE
   WHERE table_schema = 'analytics' AND table_name = 'events';
   ```
3. **Project skew:** how big are the biggest projects?
   ```sql
   SELECT project_id, COUNT(*) AS n
   FROM analytics.events
   WHERE event_ts >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
   GROUP BY project_id ORDER BY n DESC LIMIT 20;
   ```
4. **Writes:** how do events arrive (streaming API / Storage Write API / batch loads, and how often)? Are there `UPDATE`/`DELETE`/`MERGE` jobs on the table? This decides the swap procedure.
5. **Exact vs approximate distinct users:** is ~1% error acceptable on the dashboard (MV Option A), or do you need exact counts (Option B)?
6. **Other heavy queries:** if many of them also filter on `event_name` or `country`, the cluster order `project_id, event_name, country` may fit better. Send the top 5 from:
   ```sql
   SELECT query_info.query_hashes.normalized_literals AS h, COUNT(*) runs,
          SUM(total_bytes_billed)/POW(1024,4) AS tib_billed, ANY_VALUE(query) AS sample
   FROM `region-us`.INFORMATION_SCHEMA.JOBS
   WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY) AND job_type = 'QUERY'
   GROUP BY h ORDER BY tib_billed DESC LIMIT 20;
   ```
   (Replace `region-us` with your dataset's region.)

I haven't run anything against your project. Please run the CTAS and the rename yourself, ideally after testing it on a copy or a recent slice of the table.
