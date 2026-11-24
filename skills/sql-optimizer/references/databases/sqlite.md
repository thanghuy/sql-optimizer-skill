# SQLite

## Official docs to check
- Query planner overview (how indexes are used, covering, ORDER BY): https://www.sqlite.org/queryplanner.html
- Optimizer overview (WHERE clause analysis, skip-scan, LIKE optimization): https://www.sqlite.org/optoverview.html
- EXPLAIN QUERY PLAN: https://www.sqlite.org/eqp.html
- CREATE INDEX (partial, expression, DESC): https://www.sqlite.org/lang_createindex.html
- Partial indexes: https://www.sqlite.org/partialindex.html
- Indexes on expressions: https://www.sqlite.org/expridx.html
- ANALYZE and sqlite_stat1/stat4: https://www.sqlite.org/lang_analyze.html
- PRAGMA optimize: https://www.sqlite.org/pragma.html#pragma_optimize
- WITHOUT ROWID tables: https://www.sqlite.org/withoutrowid.html
- Next-generation query planner notes: https://www.sqlite.org/queryplanner-ng.html

## Version
```sql
SELECT sqlite_version();
```
On mobile the version comes with the OS / bundled library (Android, iOS, Electron) — features like `->>` JSON operators (3.38) or `PRAGMA optimize` behavior differ.

## Table structure and indexes
```sql
.schema orders                       -- sqlite3 CLI
SELECT sql FROM sqlite_schema WHERE tbl_name = 'orders';
PRAGMA index_list('orders');
PRAGMA index_xinfo('idx_orders_customer');   -- column order, DESC, collation
```

## Data volume and cardinality
```sql
SELECT COUNT(*) FROM orders;
ANALYZE;                       -- populates sqlite_stat1 (and stat4 if compiled in)
SELECT * FROM sqlite_stat1 WHERE tbl = 'orders';  -- "rows avg_rows_per_key_col1 avg_rows_per_key_col1_2 ..."
-- file size
SELECT page_count * page_size FROM pragma_page_count(), pragma_page_size();
```

## Traffic
SQLite is embedded, so there is no server-side query log. Ask: how many queries per second per process, how many concurrent writers (SQLite allows one writer at a time), journal mode (`PRAGMA journal_mode;` — WAL recommended for concurrent reads), and measure in the app (`.timer on` in the CLI, or `sqlite3_profile` / ORM logging).

## Index features relevant to ESR
- B-tree indexes with ASC/DESC per column; leftmost-prefix rule.
- No INCLUDE: append columns to make it covering ("USING COVERING INDEX").
- Rowid tables: every index stores the rowid; `INTEGER PRIMARY KEY` is the rowid itself. `WITHOUT ROWID` clusters by PK.
- Partial indexes (`WHERE deleted = 0`) and expression indexes.
- Skip-scan only when `ANALYZE` data shows the leading column has few distinct values.
- `LIKE 'abc%'` uses an index only with case-sensitive collation rules (`PRAGMA case_sensitive_like` or `COLLATE NOCASE` matching) — see optoverview.
- Without `ANALYZE` the planner uses fixed heuristics; run `PRAGMA optimize;` periodically (e.g. on app close).
- Index creation locks the database for writes; on big files do it during migration.

## Get the plan
```sql
EXPLAIN QUERY PLAN SELECT ...;
.eqp on        -- CLI: show plan automatically for every statement
.scanstats on  -- CLI (3.42+ when compiled with SQLITE_ENABLE_STMT_SCANSTATUS): actual loop/row counts
.timer on
```

## Plan red flags
- `SCAN orders` (full scan) vs good `SEARCH orders USING INDEX idx (customer_id=? AND created_at>?)`.
- `USE TEMP B-TREE FOR ORDER BY` / `FOR GROUP BY` / `FOR DISTINCT` → index does not provide order (S of ESR).
- `SEARCH ... USING INDEX` without `COVERING` on hot queries → extra lookups.
- `AUTOMATIC INDEX` / `BLOOM FILTER` → the planner built a temporary index because one is missing for a join.
- The columns listed in `(...)` after USING INDEX show exactly how many index columns were used for the seek.

## Gotchas
- Type affinity: comparing a TEXT column to an integer may skip the index.
- `OR` across columns may use `MULTI-INDEX OR` — verify.
- Too many indexes slow inserts on mobile/IoT noticeably; batch writes in transactions.
