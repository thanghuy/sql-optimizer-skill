# Oracle Database

## Official docs to check (replace `23` with `19`, `21` as needed)
- SQL Tuning Guide: https://docs.oracle.com/en/database/oracle/oracle-database/23/tgsql/
  - Optimizer access paths (index range scan, skip scan, fast full scan) — chapter "Optimizer Access Paths"
  - Generating and displaying execution plans / DBMS_XPLAN — chapters "Generating and Displaying Execution Plans", "Reading Execution Plans"
  - Optimizer statistics — chapters "Optimizer Statistics Concepts", "Gathering Optimizer Statistics"
- Database Concepts, Indexes and Index-Organized Tables: https://docs.oracle.com/en/database/oracle/oracle-database/23/cncpt/indexes-and-index-organized-tables.html
- CREATE INDEX: https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/CREATE-INDEX.html
- DBMS_XPLAN package: https://docs.oracle.com/en/database/oracle/oracle-database/23/arpls/DBMS_XPLAN.html
- Performance Tuning Guide (AWR, ASH): https://docs.oracle.com/en/database/oracle/oracle-database/23/tgdba/
If a deep link fails, search `site:docs.oracle.com <topic> <version>`.

Version notes: 12c adaptive plans, invisible columns; 18c/19c automatic indexing (Autonomous / Exadata), real-time statistics; 21c/23ai SQL analysis report in DBMS_XPLAN. Diagnostics Pack license is needed for AWR/ASH/SQL Monitor — ask before suggesting them.

## Version
```sql
SELECT banner_full FROM v$version;
```

## Table structure and indexes
```sql
SELECT DBMS_METADATA.GET_DDL('TABLE', 'ORDERS', 'APP') FROM dual;
SELECT index_name, index_type, uniqueness, visibility, status, num_rows, distinct_keys, clustering_factor, leaf_blocks
FROM all_indexes WHERE table_owner = 'APP' AND table_name = 'ORDERS';
SELECT index_name, column_name, column_position, descend
FROM all_ind_columns WHERE table_owner = 'APP' AND table_name = 'ORDERS' ORDER BY index_name, column_position;
SELECT * FROM all_ind_expressions WHERE table_owner = 'APP' AND table_name = 'ORDERS';  -- function-based
```

## Data volume and cardinality
```sql
SELECT num_rows, blocks, last_analyzed, stale_stats FROM all_tab_statistics WHERE owner='APP' AND table_name='ORDERS';
SELECT column_name, num_distinct, num_nulls, density, histogram FROM all_tab_col_statistics
WHERE owner='APP' AND table_name='ORDERS';
SELECT segment_name, ROUND(bytes/1024/1024) mb FROM dba_segments WHERE owner='APP' AND segment_name='ORDERS';
```
`clustering_factor` close to `num_rows` means index order is far from table order → range scans cost many block reads.

## Traffic
```sql
SELECT sql_id, executions, ROUND(elapsed_time/NULLIF(executions,0)/1000,2) AS avg_ms,
       buffer_gets, disk_reads, rows_processed, sql_text
FROM v$sql ORDER BY elapsed_time DESC FETCH FIRST 20 ROWS ONLY;
-- write activity per table
SELECT table_name, inserts, updates, deletes, timestamp FROM all_tab_modifications WHERE table_owner='APP';
```
AWR reports (`@?/rdbms/admin/awrrpt.sql`) and ASH for peaks — only if licensed.

## Index features relevant to ESR
- B-tree composite with ASC/DESC; index skip scan can use non-leading columns when the leading column has few distinct values.
- No INCLUDE: append columns for covering; or use an Index-Organized Table.
- Function-based indexes `ON (UPPER(email))`, `ON (TRUNC(created_at))`.
- "Partial" index trick: Oracle does not index rows where all key columns are NULL → `CREATE INDEX ON t (CASE WHEN status='OPEN' THEN id END)`.
- Bitmap indexes for low-cardinality columns in **read-mostly / DW** tables only — they lock badly under concurrent DML.
- Partitioned tables: local vs global indexes; partition pruning appears as `PARTITION RANGE SINGLE/ITERATOR`.
- Test safely: `CREATE INDEX ... INVISIBLE;` then `ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;`.
- Online build: `CREATE INDEX ... ONLINE;`.

## Get the plan
```sql
-- actual plan with row counts
SELECT /*+ GATHER_PLAN_STATISTICS */ ... ;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PEEKED_BINDS +PREDICATE'));
-- the app's real cursor
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('<sql_id>', NULL, 'ALLSTATS LAST'));
-- estimated only
EXPLAIN PLAN FOR SELECT ...; SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- long-running: SQL Monitor (licensed)
SELECT DBMS_SQL_MONITOR.REPORT_SQL_MONITOR(sql_id => '<sql_id>', type => 'TEXT') FROM dual;
```

## Plan red flags
- `TABLE ACCESS FULL` on a big table with a selective predicate.
- `E-Rows` vs `A-Rows` off by 10×+ (multiply E-Rows by `Starts` for comparison).
- `INDEX RANGE SCAN` followed by `TABLE ACCESS BY INDEX ROWID BATCHED` with high Buffers → not covering, or poor clustering factor.
- In the Predicate Information section: `access(...)` = used for seek (good); `filter(...)` on an index line = column read but not seeked → ESR order.
- `SORT ORDER BY` / `SORT GROUP BY` with `Used-Tmp` → spill.
- `INDEX FULL SCAN` / `INDEX SKIP SCAN` chosen unexpectedly.
- `filter(TO_NUMBER("COL")=...)` or `INTERNAL_FUNCTION` → implicit conversion.

## Gotchas
- Bind peeking / adaptive cursor sharing: one plan for all values unless the column has a histogram.
- Stale stats after bulk loads → `DBMS_STATS.GATHER_TABLE_STATS('APP','ORDERS', cascade => TRUE)`; extended stats for correlated columns: `DBMS_STATS.CREATE_EXTENDED_STATS`.
- Avoid hints as a fix; use SQL Plan Baselines if a plan must be stabilized.
- `ROWNUM` / `FETCH FIRST n ROWS` + ESR index gives `COUNT STOPKEY` — early stop, very fast.
