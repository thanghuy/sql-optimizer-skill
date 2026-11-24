# MySQL (InnoDB)

## Official docs to check (replace `8.4` with the user's version, e.g. `8.0`, `9.x`)
- Optimization overview: https://dev.mysql.com/doc/refman/8.4/en/optimization.html
- How MySQL uses indexes: https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html
- Multiple-column indexes: https://dev.mysql.com/doc/refman/8.4/en/multiple-column-indexes.html
- ORDER BY optimization: https://dev.mysql.com/doc/refman/8.4/en/order-by-optimization.html
- Range optimization (IN, skip scan): https://dev.mysql.com/doc/refman/8.4/en/range-optimization.html
- EXPLAIN output format: https://dev.mysql.com/doc/refman/8.4/en/explain-output.html
- EXPLAIN ANALYZE: https://dev.mysql.com/doc/refman/8.4/en/explain.html
- Invisible indexes: https://dev.mysql.com/doc/refman/8.4/en/invisible-indexes.html
- Descending indexes: https://dev.mysql.com/doc/refman/8.4/en/descending-indexes.html
- Online DDL: https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html
- Histograms: https://dev.mysql.com/doc/refman/8.4/en/optimizer-statistics.html

Version notes: 5.7 is EOL; 8.0 adds descending indexes, invisible indexes, histograms, functional indexes (8.0.13), skip scan (8.0.13), `EXPLAIN ANALYZE` (8.0.18), hash join (8.0.18); 8.4 is LTS.

## Version
```sql
SELECT VERSION();
```

## Table structure and indexes
```sql
SHOW CREATE TABLE orders\G
SHOW INDEX FROM orders;   -- Seq_in_index = column order, Cardinality = estimated distinct values
-- unused / redundant indexes (sys schema)
SELECT * FROM sys.schema_unused_indexes WHERE object_name = 'orders';
SELECT * FROM sys.schema_redundant_indexes WHERE table_name = 'orders';
```

## Data volume and cardinality
```sql
SELECT table_rows, ROUND(data_length/1024/1024) AS data_mb, ROUND(index_length/1024/1024) AS index_mb
FROM information_schema.tables WHERE table_schema = DATABASE() AND table_name = 'orders';
-- table_rows is an estimate; exact: SELECT COUNT(*) (slow on big tables)

SELECT COUNT(DISTINCT customer_id), COUNT(DISTINCT status), COUNT(*) FROM orders;  -- run on replica / sample
SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY 2 DESC;              -- skew
```

## Traffic
```sql
-- top statements by total time
SELECT digest_text, count_star, ROUND(avg_timer_wait/1e9,2) AS avg_ms,
       ROUND(sum_timer_wait/1e12,1) AS total_s, sum_rows_examined, sum_rows_sent,
       sum_no_index_used, sum_created_tmp_disk_tables, sum_sort_merge_passes
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;

-- writes per table
SELECT object_name, count_insert, count_update, count_delete
FROM performance_schema.table_io_waits_summary_by_table WHERE object_schema = DATABASE();
```
Also the slow query log (`long_query_time`, `log_queries_not_using_indexes`) and `pt-query-digest`.

## Index features relevant to ESR
- InnoDB clusters rows by PRIMARY KEY; every secondary index implicitly ends with the PK columns → do not append the PK, and a short PK keeps indexes small.
- No `INCLUDE`: make covering indexes by appending columns after the R columns.
- Leftmost prefix rule applies strictly (skip scan only in narrow cases).
- Multi-value `IN` on a column before the ORDER BY column usually causes `Using filesort`.
- Before 8.0 `DESC` in index definitions was ignored → mixed-direction ORDER BY always filesorts.
- Functional index: `CREATE INDEX idx ON t ((LOWER(email)))`; or a generated column + index.
- Prefix index on long VARCHAR: `col(20)` — cannot be covering and cannot be used for ORDER BY.
- Test safely: `ALTER TABLE t ALTER INDEX idx INVISIBLE;` before dropping.
- Build online: `ALTER TABLE t ADD INDEX idx (...), ALGORITHM=INPLACE, LOCK=NONE;` (or gh-ost / pt-online-schema-change for very large tables; watch replication lag).

## Get the plan
```sql
EXPLAIN FORMAT=TREE SELECT ...;     -- readable estimated plan (8.0.16+)
EXPLAIN ANALYZE SELECT ...;         -- actual rows/time; EXECUTES the query (8.0.18+)
EXPLAIN FORMAT=JSON SELECT ...;     -- cost details, used_columns, "using_filesort"
-- session handler counters, before/after:
FLUSH STATUS; SELECT ...; SHOW SESSION STATUS LIKE 'Handler_read%';
-- optimizer reasoning (why an index was rejected):
SET optimizer_trace='enabled=on'; SELECT ...; SELECT * FROM information_schema.optimizer_trace\G
```

## Plan red flags (classic EXPLAIN columns)
- `type = ALL` (full scan) or `index` (full index scan) on large tables. Good: `const`, `eq_ref`, `ref`, `range`.
- `key = NULL` while `possible_keys` lists indexes → optimizer rejected them (stats, low selectivity, cast).
- `key_len` shorter than expected → only a prefix of the composite index is used (range stopped it).
- `rows × filtered` much larger than the result.
- `Extra: Using filesort` / `Using temporary` → ORDER BY / GROUP BY not served by index (S of ESR).
- `Extra: Using index` = covering (good); `Using index condition` = ICP filter inside index (OK); `Using where` alone on big rows = filtering after reading rows.
- `Handler_read_rnd_next` high → full scans.

## Gotchas
- `WHERE varchar_col = 123` casts the column → no index. Quote the value.
- JOIN columns with different charset/collation → no index on the join.
- `ORDER BY ... LIMIT` with large OFFSET → keyset pagination.
- `ANALYZE TABLE` refreshes index stats; `innodb_stats_persistent_sample_pages` for accuracy; histograms (`ANALYZE TABLE t UPDATE HISTOGRAM ON col`) for skewed non-indexed columns.
- `OR` across columns → index_merge sometimes; `UNION ALL` is more predictable.
