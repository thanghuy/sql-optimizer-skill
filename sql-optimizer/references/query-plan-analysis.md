# Analyzing a query plan

Use this after Step 6, when the user sends back a plan. Engine-specific operator names and commands are in `references/databases/<db>.md`; this file is the common method.

## 1. Make sure it is the right plan

- **Actual, not only estimated.** Estimated plans hide the most important signal: estimated vs actual rows. Ask for the actual plan whenever it is safe.
- **Same parameters as production.** A plan for `customer_id = 1` (10 rows) can differ completely from one for the biggest customer (2M rows). Ask for both a typical and a worst-case value.
- **Prepared statements / ORMs** may get a generic or cached plan different from the one you see in a console (PG generic plans, SQL Server parameter sniffing, Oracle bind peeking/adaptive cursor sharing). If the console is fast but the app is slow, get the plan from the cache (Query Store, `DBMS_XPLAN.DISPLAY_CURSOR(sql_id)`, `auto_explain`, `performance_schema`).
- **Warm vs cold cache.** Run twice; compare the second run, but note if the first run is the real-world case.
- **Data-modifying statements** execute under EXPLAIN ANALYZE: wrap in `BEGIN; ... ROLLBACK;`.

## 2. Read it in this order

1. **Total time and where it goes.** Find the node(s) with the largest actual time / cost share. Ignore everything else at first.
2. **Access path per table.** Full scan, index range scan / seek, index-only scan, bitmap scan? Is the index you designed actually chosen?
3. **Rows: estimated vs actual.** Off by ≥10× → statistics, correlated columns, skew or parameter sniffing. Fix stats before touching indexes: `ANALYZE`, `UPDATE STATISTICS`, `DBMS_STATS.GATHER_TABLE_STATS`, `ANALYZE TABLE`, extended/multi-column statistics, histograms.
4. **Rows read vs rows returned.** Reading 500k index entries to return 20 rows means a column is in the wrong position (usually a range before the sort, or a missing equality column), or a filter happens after the index instead of inside it ("Rows Removed by Filter", "Using where", residual predicate).
5. **Sort / temp / hash spills.** A `Sort`, `Using filesort`, `Using temporary`, `SORT ORDER BY`, a spill to disk / tempdb warning → the S part of ESR is not satisfied, or memory (`work_mem`, `sort_buffer_size`, memory grant) is too small.
6. **Lookups back to the table.** Key Lookup (SQL Server), `TABLE ACCESS BY INDEX ROWID` (Oracle), heap fetches (PG), no "Using index" (MySQL), executed once per row → consider a covering index if the query is hot and the column list is small.
7. **Joins.** Nested loop with a big outer side and no index on the inner join column is the classic disaster. Hash join on two large inputs is fine for reports, bad for OLTP. Check join order — the optimizer should start from the most selective table.
8. **Buffers / logical reads / bytes scanned.** Better than time for comparing before/after, because they do not depend on cache warmness or machine load.

## 3. Red flags (any engine)

| Symptom in plan | Likely cause | Typical fix |
|---|---|---|
| Full scan on large table, selective WHERE | No usable index, function on column, implicit cast, stale stats | ESR index, rewrite predicate, cast parameter, refresh stats |
| Index used but huge rows read | Range column before sort column, or missing equality column | Reorder per ESR |
| Sort step with LIMIT | Index order does not match ORDER BY | Put sort columns right after equality columns |
| Estimate 1, actual 100,000 | Stats / correlation / skew / sniffing | Stats, extended stats, histograms, plan guides as last resort |
| Per-row lookups to table | Non-covering index | INCLUDE / trailing columns |
| Nested loop × millions | Missing index on inner join column, bad estimate | Index the FK, fix stats |
| Spill to disk | Too little memory for sort/hash | Avoid the sort (index) or raise memory for that session |
| Different plan in app vs console | Generic plan / sniffing / different session settings | Capture app plan, compare settings, adjust |
| Columnar: almost all partitions scanned | Filter not on partition/cluster key, or function on it | Filter on the raw partition column, re-cluster, change sort key |

## 4. Report the comparison

Always show before vs after in a small table:

| Metric | Before | After |
|---|---|---|
| Execution time | | |
| Rows read / examined | | |
| Buffers / logical reads / bytes scanned | | |
| Sort / temp step | yes/no | yes/no |
| Index used | | |

Then decide:
- Target met → done; remind the user to drop any index the new one made redundant (after checking usage stats for a while) and to monitor write latency.
- Not met → state the next bottleneck from the plan and loop back to Step 5 (query rewrite) or Step 6 (index).
- Inherent cost (the query must read a large share of the data) → suggest a structural change: summary/rollup table, materialized view, caching, read replica, async job, or moving to a columnar store.
