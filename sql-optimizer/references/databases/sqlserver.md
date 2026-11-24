# Microsoft SQL Server / Azure SQL

## Official docs to check
- Index architecture and design guide: https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide
- Create indexes with included columns: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-indexes-with-included-columns
- Filtered indexes: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes
- Display an actual execution plan: https://learn.microsoft.com/en-us/sql/relational-databases/performance/display-an-actual-execution-plan
- Showplan logical and physical operators: https://learn.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference
- Query Store: https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store
- Intelligent query processing (PSP, memory grant feedback): https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing
- Statistics: https://learn.microsoft.com/en-us/sql/relational-databases/statistics/statistics
- Columnstore indexes: https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview

Version notes: compatibility level matters as much as version (`SELECT compatibility_level FROM sys.databases`). 2016 Query Store; 2017 adaptive joins; 2019 batch mode on rowstore, table variable deferred compilation; 2022 Parameter Sensitive Plan optimization, DOP feedback; 2025 optional parameter plan optimization.

## Version
```sql
SELECT @@VERSION, SERVERPROPERTY('ProductVersion'), SERVERPROPERTY('Edition');
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();
```

## Table structure and indexes
```sql
EXEC sp_help 'dbo.Orders';
-- indexes with key and included columns, filters
SELECT i.name, i.type_desc, i.is_unique, i.filter_definition,
       STRING_AGG(CASE WHEN ic.is_included_column = 0 THEN c.name + IIF(ic.is_descending_key=1,' DESC','') END, ', ')
         WITHIN GROUP (ORDER BY ic.key_ordinal) AS key_cols,
       STRING_AGG(CASE WHEN ic.is_included_column = 1 THEN c.name END, ', ') AS included_cols
FROM sys.indexes i
JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
JOIN sys.columns c ON c.object_id = ic.object_id AND c.column_id = ic.column_id
WHERE i.object_id = OBJECT_ID('dbo.Orders')
GROUP BY i.name, i.type_desc, i.is_unique, i.filter_definition;

-- usage (reads vs writes per index) since last restart
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s JOIN sys.indexes i ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE s.database_id = DB_ID() AND s.object_id = OBJECT_ID('dbo.Orders');
```

## Data volume and cardinality
```sql
EXEC sp_spaceused 'dbo.Orders';
DBCC SHOW_STATISTICS ('dbo.Orders', IX_Orders_CustomerId);   -- density, histogram
SELECT s.name, sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM sys.stats s CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders');
```

## Traffic
```sql
-- Query Store: top queries by total duration
SELECT TOP 20 qt.query_sql_text, SUM(rs.count_executions) AS execs,
       AVG(rs.avg_duration)/1000.0 AS avg_ms, AVG(rs.avg_logical_io_reads) AS avg_reads
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
GROUP BY qt.query_sql_text ORDER BY SUM(rs.avg_duration * rs.count_executions) DESC;
-- without Query Store: sys.dm_exec_query_stats + sys.dm_exec_sql_text
```
Writes: `user_updates` above, `sys.dm_db_index_operational_stats` (leaf inserts/updates/deletes).
Missing index DMVs (`sys.dm_db_missing_index_details`) are hints only: they ignore ESR ordering of sort columns and suggest overlapping indexes — validate with ESR before applying.

## Index features relevant to ESR
- Clustered index = table order; nonclustered indexes carry the clustering key as row locator.
- `INCLUDE (...)` for covering columns; key columns limited to 16 / 1700 bytes.
- ASC/DESC per key column.
- Filtered index `WHERE IsDeleted = 0` (beware: parameterized queries may not match the filter; `SET` options must be correct).
- Columnstore (clustered or nonclustered) for analytics on large tables.
- Online build: `CREATE INDEX ... WITH (ONLINE = ON, RESUMABLE = ON)` — Enterprise edition / Azure SQL.
- Disable instead of drop to test: `ALTER INDEX ... DISABLE` (for nonclustered).

## Get the plan
```sql
SET STATISTICS IO, TIME ON;
SET STATISTICS XML ON;     -- actual plan as XML; or "Include Actual Execution Plan" (Ctrl+M) in SSMS
SELECT ...;
SET STATISTICS XML OFF;
-- plan from the cache / Query Store for the app's real parameters:
SELECT qp.query_plan FROM sys.dm_exec_query_stats qs CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp ...;
-- live: sys.dm_exec_query_statistics_xml(session_id)
```
Ask the user to save the plan as `.sqlplan` (XML) — it contains estimated vs actual rows, warnings and memory grants.

## Plan red flags
- `Clustered Index Scan` / `Table Scan` on a big table with selective predicate.
- `Index Seek` with a residual `Predicate` (not `Seek Predicate`) → column order; check "Number of Rows Read" vs "Actual Number of Rows".
- `Key Lookup` / `RID Lookup` executed many times → add INCLUDE columns.
- `Sort` operator (S of ESR not met); yellow warning "Operator used tempdb to spill data".
- Implicit conversion warning (`CONVERT_IMPLICIT`), e.g. NVARCHAR parameter vs VARCHAR column → scan.
- Estimated vs actual rows far apart; "Parameter sniffing" when plan fits one value only.
- Excessive memory grant warning.
- `Logical reads` in STATISTICS IO — best before/after metric.

## Gotchas
- Parameter sniffing: try `OPTION (RECOMPILE)` as a diagnostic, not a permanent fix; on 2022 check PSP.
- Non-SARGable: `WHERE YEAR(OrderDate) = 2025`, `ISNULL(col, 0) = 0`.
- Table variables / multi-statement TVFs have bad estimates on older compat levels.
- `SELECT TOP n ... ORDER BY` benefits massively from ESR because the seek can stop early.
