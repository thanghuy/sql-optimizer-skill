# Google BigQuery

BigQuery has **no B-tree indexes**. Cost and speed are driven by bytes scanned and slot time. The tools are partitioning, clustering (up to 4 columns), search indexes, vector indexes, materialized views, and query rewrites. Translate ESR into partition + clustering order (see `esr-rule.md` §9). BigQuery is versionless (continuously updated), so always check current docs for limits.

## Official docs to check
- Optimize query computation (best practices): https://cloud.google.com/bigquery/docs/best-practices-performance-compute
- Reduce data processed: https://cloud.google.com/bigquery/docs/best-practices-performance-input
- Partitioned tables: https://cloud.google.com/bigquery/docs/partitioned-tables
- Clustered tables: https://cloud.google.com/bigquery/docs/clustered-tables
- Query plan and timeline: https://cloud.google.com/bigquery/docs/query-plan-explanation
- Search indexes: https://cloud.google.com/bigquery/docs/search-index
- Materialized views: https://cloud.google.com/bigquery/docs/materialized-views-intro
- INFORMATION_SCHEMA.JOBS: https://cloud.google.com/bigquery/docs/information-schema-jobs
- Pricing (on-demand vs capacity/editions): https://cloud.google.com/bigquery/pricing

## Table structure
```sql
SELECT ddl FROM `project.dataset.INFORMATION_SCHEMA.TABLES` WHERE table_name = 'orders';
SELECT column_name, clustering_ordinal_position, is_partitioning_column
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS` WHERE table_name = 'orders';
SELECT * FROM `project.dataset.INFORMATION_SCHEMA.SEARCH_INDEXES` WHERE table_name = 'orders';
```

## Data volume
```sql
SELECT table_name, total_rows, total_logical_bytes, total_physical_bytes
FROM `project.region-us.INFORMATION_SCHEMA.TABLE_STORAGE` WHERE table_name = 'orders';
SELECT partition_id, total_rows, total_logical_bytes
FROM `project.dataset.INFORMATION_SCHEMA.PARTITIONS` WHERE table_name = 'orders' ORDER BY partition_id DESC LIMIT 30;
SELECT APPROX_COUNT_DISTINCT(customer_id), APPROX_COUNT_DISTINCT(country) FROM `project.dataset.orders`;
```

## Traffic and cost
```sql
SELECT query_info.query_hashes.normalized_literals AS query_hash, COUNT(*) AS runs,
       AVG(total_slot_ms) AS avg_slot_ms, AVG(total_bytes_processed)/POW(1024,3) AS avg_gb,
       ANY_VALUE(query) AS sample_query
FROM `project.region-us.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY) AND job_type = 'QUERY'
GROUP BY query_hash ORDER BY SUM(total_slot_ms) DESC LIMIT 20;
```
Ask: on-demand (pay per TB scanned) or capacity/editions (pay for slots)? Streaming/DML frequency on the table? Dashboards (BI Engine) or batch?

## ESR translated
- **Range** filter that almost every query has (usually a date/timestamp) → **partition** column. Require it with `require_partition_filter = TRUE` on big tables.
- **Equality** filters → **clustering** columns, most frequently filtered first (order matters: pruning is best on the first column, and later columns help only together with earlier ones).
- Clustering also helps `GROUP BY` / `JOIN` on those columns; ORDER BY with LIMIT still sorts, but on fewer bytes.
- High-cardinality text lookups / `SEARCH()` / `IN` on IDs → search index.
- Repeated aggregations → materialized view (automatic smart tuning can rewrite queries to use it).
- Partitions should not be tiny (aim for roughly ≥1 GB per partition on average); otherwise cluster by date instead of partitioning.

## Get the plan
- Dry run for bytes estimate: `bq query --dry_run --use_legacy_sql=false 'SELECT ...'` (or the "This query will process X GB" hint in the console).
- After running: console **Execution details** / **Execution graph** tab, or
```sql
SELECT stage.name, stage.records_read, stage.records_written, stage.slot_ms, stage.shuffle_output_bytes_spilled
FROM `project.region-us.INFORMATION_SCHEMA.JOBS`, UNNEST(job_stages) AS stage
WHERE job_id = '<job_id>';
```
- `total_bytes_processed` vs `total_bytes_billed`, and whether clustering pruned: compare bytes processed with and without the filter.

## Plan red flags
- Bytes processed ≈ whole table with a selective filter → filter not on partition/cluster column, or wrapped in a function/cast (e.g. `DATE(ts_col)` on an ingestion-time partition is fine, `CAST(col AS STRING)` is not).
- One stage with far higher `slot_ms` than others; `compute_ratio_max` ≫ `compute_ratio_avg` → data skew on a join/group key.
- `shuffle_output_bytes_spilled` > 0 → huge shuffles; pre-aggregate or filter earlier.
- Join output ≫ inputs → duplicate join keys.
- `ORDER BY` without `LIMIT` on large results → runs on a single worker.

## Gotchas
- `SELECT *` scans every column; `LIMIT` does not reduce bytes scanned on non-clustered tables.
- Cached results are free and fast; disable cache (`--nouse_cache`) when benchmarking.
- Wildcard tables / sharded tables (`events_2025*`) are slower than one partitioned table.
- Clustering on a table that receives heavy streaming/DML is re-clustered automatically in the background (no cost), but freshly written data may be less clustered.
