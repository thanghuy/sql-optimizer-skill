# IBM Db2 (LUW; notes for z/OS where different)

## Official docs to check
Db2 docs live in IBM Documentation and deep links change often. Start from the version root and search inside it, or search `site:ibm.com/docs db2 <version> <topic>`.
- Db2 11.5 / 12.1 for Linux, UNIX and Windows: https://www.ibm.com/docs/en/db2/11.5 , https://www.ibm.com/docs/en/db2/12.1
- Topics to look up: "Index planning", "CREATE INDEX statement", "Explain facility", "db2exfmt", "Optimizer statistics / RUNSTATS", "Jump scans", "Column-organized tables", "Multidimensional clustering (MDC)", "Monitoring table functions MON_GET_PKG_CACHE_STMT".
- Db2 for z/OS: https://www.ibm.com/docs/en/db2-for-zos

## Version
```sql
SELECT service_level, fixpack_num FROM TABLE(sysproc.env_get_inst_info());
-- or: db2level (command line)
```

## Table structure and indexes
```bash
db2look -d MYDB -z APP -t ORDERS -e      # DDL including indexes
```
```sql
SELECT indname, uniquerule, indextype, colnames, fullkeycard, firstkeycard, clusterratio, clusterfactor, lastused
FROM syscat.indexes WHERE tabschema = 'APP' AND tabname = 'ORDERS';
-- colnames shows +COL (ASC), -COL (DESC); INCLUDE columns appear via syscat.indexcoluse colorder = 'I'
SELECT indname, colname, colseq, colorder FROM syscat.indexcoluse WHERE indschema = 'APP' ORDER BY indname, colseq;
```

## Data volume and cardinality
```sql
SELECT card, npages, fpages, stats_time FROM syscat.tables WHERE tabschema='APP' AND tabname='ORDERS';
SELECT colname, colcard, numnulls, high2key, low2key FROM syscat.columns WHERE tabschema='APP' AND tabname='ORDERS';
-- refresh stats (with distribution and index details)
CALL SYSPROC.ADMIN_CMD('RUNSTATS ON TABLE APP.ORDERS WITH DISTRIBUTION AND SAMPLED DETAILED INDEXES ALL');
```

## Traffic
```sql
SELECT num_executions, total_act_time, rows_read, rows_returned, pool_data_l_reads + pool_index_l_reads AS logical_reads,
       SUBSTR(stmt_text, 1, 200) AS stmt
FROM TABLE(MON_GET_PKG_CACHE_STMT(NULL, NULL, NULL, -2)) ORDER BY total_act_time DESC FETCH FIRST 20 ROWS ONLY;
SELECT tabname, rows_read, rows_inserted, rows_updated, rows_deleted
FROM TABLE(MON_GET_TABLE('APP', 'ORDERS', -2));
```

## Index features relevant to ESR
- B-tree with ASC/DESC per column; `ALLOW REVERSE SCANS` (default on newer versions) lets one index serve both directions.
- `INCLUDE (...)` is allowed **only on UNIQUE indexes** on LUW; otherwise append columns for covering ("index-only access").
- `CLUSTER` index keeps table rows roughly in index order (check `clusterratio`); MDC tables cluster by dimensions.
- Jump scan (11.1+) can skip gaps in leading columns — do not rely on it.
- Expression-based indexes (10.5+): `CREATE INDEX ... ON t (UPPER(email))`.
- Column-organized (BLU) tables: no regular secondary indexes for analytics; rely on synopsis tables (data skipping).
- `CREATE INDEX` with `ALLOW WRITE ACCESS` on recent versions for online builds; check version.

## Get the plan
```sql
-- once per database: create explain tables
CALL SYSPROC.SYSINSTALLOBJECTS('EXPLAIN', 'C', NULL, CURRENT SCHEMA);
EXPLAIN PLAN FOR SELECT ...;
```
```bash
db2exfmt -d MYDB -1 -o plan.txt        # formatted plan of the latest explained statement
db2expln -d MYDB -q "SELECT ..." -t -g # quick graph
```
Actual row counts: enable section actuals (`db2 update db cfg using SECTION_ACTUALS BASE`, or `WLM_SET_CONN_ENV`), capture with an activity event monitor, then `EXPLAIN_FROM_ACTIVITY` and `db2exfmt`.

## Plan red flags
- `TBSCAN` on a large table with a selective predicate (vs `IXSCAN` + `FETCH`, or `IXSCAN` alone = index-only).
- `IXSCAN` with many "Sargable Predicates" applied after start/stop keys → column order; look at "Start Key / Stop Key" lines.
- `SORT` operators (check "spilled" / temporary table) → S of ESR.
- `FETCH` many times with low clusterratio → covering or cluster index.
- Estimated cardinality vs actuals off → `RUNSTATS` with distribution, column group stats (`ON COLUMNS ((a, b))`).
- `NLJOIN` with a `TBSCAN` inner.

## Gotchas
- Static SQL packages keep their plan until `REBIND` — new indexes are not used until rebind.
- `REOPT ONCE/ALWAYS` bind option for skewed parameters.
- Index `lastused` helps find unused indexes.
