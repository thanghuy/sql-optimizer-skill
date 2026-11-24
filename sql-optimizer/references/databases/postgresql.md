# PostgreSQL

## Official docs to check (replace `current` with the major version, e.g. `16`)
- Indexes overview: https://www.postgresql.org/docs/current/indexes.html
- Multicolumn indexes: https://www.postgresql.org/docs/current/indexes-multicolumn.html
- Indexes and ORDER BY: https://www.postgresql.org/docs/current/indexes-ordering.html
- Index-only scans & covering (INCLUDE): https://www.postgresql.org/docs/current/indexes-index-only-scans.html
- Partial / expression indexes: https://www.postgresql.org/docs/current/indexes-partial.html , https://www.postgresql.org/docs/current/indexes-expressional.html
- Using EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- Planner statistics / extended stats: https://www.postgresql.org/docs/current/planner-stats.html
- CREATE INDEX (CONCURRENTLY): https://www.postgresql.org/docs/current/sql-createindex.html
- Release notes for version-specific planner changes: https://www.postgresql.org/docs/release/

Version notes: 11 INCLUDE; 13 incremental sort, B-tree dedup; 14 `pg_stat_statements` improvements; 16 `EXPLAIN (GENERIC_PLAN)`; 17 faster multi-value `IN`/`= ANY` B-tree scans, `EXPLAIN (SERIALIZE, MEMORY)`; 18 B-tree skip scan, `EXPLAIN ANALYZE` shows BUFFERS by default.

## Version
```sql
SELECT version();
```

## Table structure and indexes
```sql
-- psql
\d+ schema.table_name

-- plain SQL
SELECT indexname, indexdef FROM pg_indexes WHERE schemaname = 'public' AND tablename = 'orders';

-- index usage and size (find unused / redundant indexes)
SELECT s.indexrelname, s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s WHERE s.relname = 'orders' ORDER BY s.idx_scan;
```

## Data volume and cardinality
```sql
SELECT relname, n_live_tup, pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       n_tup_ins, n_tup_upd, n_tup_del, last_analyze, last_autoanalyze
FROM pg_stat_user_tables WHERE relname = 'orders';

-- distinct values and most common values (negative n_distinct = fraction of rows)
SELECT attname, n_distinct, null_frac, most_common_vals, most_common_freqs, correlation
FROM pg_stats WHERE tablename = 'orders' AND attname IN ('customer_id','status','created_at');
```

## Traffic
```sql
-- needs the pg_stat_statements extension
SELECT calls, round(mean_exec_time::numeric,2) AS mean_ms, round(total_exec_time::numeric) AS total_ms,
       rows, shared_blks_hit, shared_blks_read, query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
```
Writes: `n_tup_ins/upd/del` above, sampled twice to get a rate. Note `n_tup_hot_upd`: adding an index on a frequently updated column disables HOT updates for that column and can increase bloat.

## Index features relevant to ESR
- B-tree supports ASC/DESC and NULLS FIRST/LAST per column; can be scanned backwards.
- `INCLUDE (...)` for covering without affecting order.
- Partial indexes `WHERE deleted_at IS NULL` remove constant equality columns from the key.
- Expression indexes `ON (lower(email))`.
- BRIN for huge append-only tables filtered by time; GIN for jsonb/arrays/full-text; `pg_trgm` GIN/GiST for `LIKE '%x%'`.
- Index-only scans need the visibility map to be fresh → `VACUUM` matters; check "Heap Fetches".
- Build without blocking writes: `CREATE INDEX CONCURRENTLY` (cannot run inside a transaction; check for `INVALID` index if it fails).
- Hypothetical indexes for testing: `hypopg` extension.

## Get the plan
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) SELECT ...;
-- JSON for tools such as explain.dalibo.com: add FORMAT JSON
-- for UPDATE/DELETE:
BEGIN; EXPLAIN (ANALYZE, BUFFERS) UPDATE ...; ROLLBACK;
-- plan as the app sees it with parameters (16+):
EXPLAIN (GENERIC_PLAN) SELECT ... WHERE id = $1;
```
For app-side plans enable `auto_explain` (`auto_explain.log_min_duration`, `log_analyze`).

## Plan red flags
- `Seq Scan` on a big table with a selective filter.
- `Rows Removed by Filter` large relative to rows returned → column order or missing column.
- `Sort` (check `Sort Method: external merge Disk` = spill → `work_mem`) or `Incremental Sort` (partial ESR match).
- `Index Scan` instead of `Index Only Scan` on hot queries; `Heap Fetches` high → vacuum or not covering.
- `Bitmap Heap Scan` with `Recheck Cond` and `lossy` blocks → `work_mem`.
- `rows=` estimate vs `actual rows=` off by 10×+ → `ANALYZE`, `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 1000`, `CREATE STATISTICS` for correlated columns.
- `Nested Loop` with `loops=` in the hundreds of thousands.

## Gotchas
- After 5 executions a prepared statement may switch to a generic plan (`plan_cache_mode`).
- `OFFSET` large values read and discard rows → keyset pagination.
- `NOT IN (subquery)` with NULLs behaves oddly and plans badly → `NOT EXISTS`.
- Types: `WHERE uuid_col = 'text'` is fine (literal is cast), but `WHERE int_col::text = ?` kills the index.
