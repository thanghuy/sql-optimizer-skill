# ClickHouse

ClickHouse (MergeTree family) has a **sparse primary index** defined by the table's `ORDER BY` (one index mark per granule, default 8192 rows), plus data-skipping indexes and projections. There are no B-tree secondary indexes. ESR maps onto the choice of `ORDER BY` / `PRIMARY KEY` (see `esr-rule.md` §9). The sort key is fixed at table creation — changing it means creating a new table (or adding a projection).

## Official docs to check
- Primary indexes (sparse index guide): https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes
- Choosing a primary key (best practices): https://clickhouse.com/docs/best-practices/choosing-a-primary-key
- Data skipping indexes: https://clickhouse.com/docs/optimize/skipping-indexes
- Projections: https://clickhouse.com/docs/sql-reference/statements/alter/projection
- Partitioning key (and why not to over-partition): https://clickhouse.com/docs/engines/table-engines/mergetree-family/custom-partitioning-key
- EXPLAIN: https://clickhouse.com/docs/sql-reference/statements/explain
- Query optimization guide: https://clickhouse.com/docs/optimize/query-optimization
- system.query_log: https://clickhouse.com/docs/operations/system-tables/query_log
If a path moved (docs were reorganized in 2024–2025), search `site:clickhouse.com/docs <topic>`.

## Version
```sql
SELECT version();
```
Self-hosted vs ClickHouse Cloud (SharedMergeTree) — some settings differ.

## Table structure and indexes
```sql
SHOW CREATE TABLE db.events;   -- ENGINE, PARTITION BY, ORDER BY, PRIMARY KEY, INDEX ... TYPE ..., PROJECTION
SELECT name, type, expr, granularity FROM system.data_skipping_indices WHERE table = 'events';
```

## Data volume
```sql
SELECT table, sum(rows) AS rows, formatReadableSize(sum(data_compressed_bytes)) AS compressed,
       formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed, count() AS parts,
       uniqExact(partition) AS partitions
FROM system.parts WHERE active AND table = 'events' GROUP BY table;
SELECT uniq(tenant_id), uniq(event_type), uniq(user_id) FROM db.events;   -- cardinality of key candidates
```

## Traffic
```sql
SELECT normalized_query_hash, count() AS runs, avg(query_duration_ms) AS avg_ms,
       avg(read_rows) AS avg_read_rows, avg(result_rows) AS avg_result_rows,
       formatReadableSize(avg(memory_usage)) AS avg_mem, any(query) AS sample
FROM system.query_log
WHERE type = 'QueryFinish' AND event_time > now() - INTERVAL 1 DAY
GROUP BY normalized_query_hash ORDER BY sum(query_duration_ms) DESC LIMIT 20;
```
Ask also: insert rate and batch size (many small inserts → too many parts), and whether queries run concurrently from dashboards.

## ESR translated
- `ORDER BY (E1, E2, ..., R)`: columns filtered by equality in most queries first, generally **lower cardinality before higher cardinality** (it improves both pruning of later columns and compression), then the time/range column.
- If most queries are "last N hours for tenant X", `ORDER BY (tenant_id, toStartOfHour(ts), ...)` or `(tenant_id, ts)` is typical.
- Queries whose `ORDER BY` matches a prefix of the sort key can read in order (`optimize_read_in_order`) and stop early with LIMIT — this is the "S" benefit.
- Secondary access patterns → **projection** with a different ORDER BY, or a materialized view into another table.
- Selective filters on non-key columns → skip index (`minmax`, `set(N)`, `bloom_filter`, `ngrambf_v1`/`tokenbf_v1` for text). They only help when matching values are clustered in few granules.
- `PARTITION BY` is for data management (TTL, dropping old data), usually by month or day — not as a query index. Too many partitions hurts.
- `FINAL` on ReplacingMergeTree is expensive; check if it can be avoided.

## Get the plan
```sql
EXPLAIN indexes = 1 SELECT ...;     -- shows parts/granules selected by partition, primary key, skip indexes
EXPLAIN PIPELINE SELECT ...;        -- parallelism
EXPLAIN actions = 1 SELECT ...;
-- actual work of a run:
SELECT query_duration_ms, read_rows, read_bytes, result_rows, memory_usage
FROM system.query_log WHERE query_id = '<id>' AND type = 'QueryFinish';
-- clickhouse-client prints "Processed N rows, X MB" after every query; send_logs_level = 'trace' shows index analysis
```

## Plan red flags
- `EXPLAIN indexes = 1` shows `Granules: 12000/12000` for PrimaryKey → the filter does not use the sort key prefix.
- `read_rows` ≫ `result_rows` for a point-ish query.
- Skip index listed but dropping almost no granules → wrong type or data not clustered on that column; remove it (it costs on insert).
- High `memory_usage` in GROUP BY / JOIN → join order (smaller table on the right), `JOIN` → `IN` / dictionaries, or `max_bytes_before_external_group_by`.
- Many active parts per partition → insert batching problem, not query design.

## Gotchas
- Functions on key columns can still use the index if monotonic (e.g. `toDate(ts)`), but not arbitrary functions.
- `PRIMARY KEY` can be a prefix of `ORDER BY` to keep the index small in memory.
- Nullable columns in the sort key are discouraged.
- Benchmark with `SETTINGS use_query_cache = 0` and note page cache effects.
