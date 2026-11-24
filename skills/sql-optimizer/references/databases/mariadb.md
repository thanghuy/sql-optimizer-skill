# MariaDB

MariaDB forked from MySQL 5.5 and has diverged: the optimizer, EXPLAIN tooling and several index features differ. Do not assume MySQL 8 behavior.

## Official docs to check
MariaDB moved its knowledge base to `mariadb.com/docs` (older `mariadb.com/kb/en/...` links usually redirect). If a link fails, search `site:mariadb.com <topic>`.
- Optimization and indexes: https://mariadb.com/kb/en/optimization-and-indexes/
- Building the best INDEX for a given SELECT: https://mariadb.com/kb/en/building-the-best-index-for-a-given-select/
- ANALYZE statement (actual plan): https://mariadb.com/kb/en/analyze-statement/
- EXPLAIN / EXPLAIN FORMAT=JSON: https://mariadb.com/kb/en/explain/
- Optimizer trace: https://mariadb.com/kb/en/optimizer-trace-overview/
- Engine-independent statistics / histograms: https://mariadb.com/kb/en/engine-independent-table-statistics/
- Ignored indexes: https://mariadb.com/kb/en/ignored-indexes/
- Descending indexes (added in 10.8, MDEV-13756): https://mariadb.com/kb/en/what-is-mariadb-108/ (the old `/kb/en/descending-indexes/` link now lands on a generic page)
- Release notes for optimizer changes (11.0 cost model rewrite): https://mariadb.com/kb/en/release-notes/

Version notes: 10.6 / 10.11 / 11.4 / 11.8 are LTS; 10.6 ignored indexes; 10.8 descending indexes; 11.0 new cost-based optimizer (plans can change after upgrade from 10.x); `ANALYZE` statement since 10.1.

## Version
```sql
SELECT VERSION();
```

## Table structure and indexes
```sql
SHOW CREATE TABLE orders\G
SHOW INDEX FROM orders;
SELECT * FROM information_schema.INDEX_STATISTICS WHERE TABLE_NAME = 'orders';  -- needs userstat=1
```

## Data volume and cardinality
```sql
SELECT table_rows, ROUND(data_length/1024/1024) AS data_mb, ROUND(index_length/1024/1024) AS index_mb
FROM information_schema.tables WHERE table_schema = DATABASE() AND table_name = 'orders';
ANALYZE TABLE orders PERSISTENT FOR ALL;   -- collect engine-independent stats + histograms
SELECT * FROM mysql.column_stats WHERE table_name = 'orders';
```

## Traffic
- `performance_schema` (disabled by default in MariaDB; `performance_schema=ON` needs a restart), same digest query as MySQL.
- Slow query log with `log_slow_verbosity=query_plan,explain` writes plans into the log.
- `userstat=1` enables `TABLE_STATISTICS` / `INDEX_STATISTICS` (rows read/changed per table/index).

## Index features relevant to ESR
- Same leftmost-prefix and InnoDB clustered-PK rules as MySQL (PK appended to secondary indexes; no INCLUDE).
- Descending index parts only from 10.8; before that mixed ASC/DESC ORDER BY filesorts.
- No functional indexes like MySQL 8: use a **virtual/persistent generated column** and index it.
- `ALTER TABLE t ALTER INDEX idx IGNORED;` to test removing an index (10.6+).
- Rowid filter (10.4+) and condition pushdown can change which index wins — check `ANALYZE` output.
- Online build: `ALTER TABLE ... ADD INDEX ..., ALGORITHM=INPLACE, LOCK=NONE;`.

## Get the plan
```sql
EXPLAIN SELECT ...;
ANALYZE FORMAT=JSON SELECT ...;   -- executes; shows r_rows (actual), r_filtered, r_total_time_ms
SHOW ANALYZE FORMAT=JSON FOR <connection_id>;   -- plan of a running query
SET optimizer_trace='enabled=on'; SELECT ...; SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

## Plan red flags
- Same as MySQL: `type=ALL`, `Using filesort`, `Using temporary`, short `key_len`.
- In `ANALYZE` output: `rows` (estimate) vs `r_rows` (actual) far apart; `r_filtered` low → many rows read then discarded.
- `Rowid-ordered scan` / `Using rowid filter` appearing or disappearing between versions.

## Gotchas
- Plans may change significantly after upgrading to 11.x due to the new cost model — re-verify hot queries.
- Engine-independent stats are not auto-collected unless `use_stat_tables` / `histogram_size` are set; old stats mislead the optimizer.
- Aria/MyISAM tables do not cluster by PK; covering logic differs from InnoDB.
