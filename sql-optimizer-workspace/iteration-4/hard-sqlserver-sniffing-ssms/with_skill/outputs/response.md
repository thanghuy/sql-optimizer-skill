**Short answer: no, you don't need another index.** Your plan-cache evidence shows the cause: **parameter sniffing on a skewed column.** SSMS stays fast because it uses its own plan-cache entry. At most you need to *extend* the index you already have by one INCLUDE column (see check A below). A second index won't help.

(I checked the SQL Server 2019 docs for your edition. That matters here because **Standard edition has no online index builds and no row-mode memory grant feedback**, and Parameter Sensitive Plan optimization needs SQL Server 2022 / compat 160. So nothing built in will fix this for you on 2019 Standard.)

---

## Diagnosis

1. **The cached plan was compiled for `'Closed'` and is reused for `'Open'`.**
   `ParameterCompiledValue = 'Closed'`, Estimated rows 11,402,113 (that's the 'Closed' count, so your statistics are accurate), `ParameterRuntimeValue = 'Open'`, Actual rows 9,041.
   The plan cached for 'Closed' expected ~95% of the table, so it chose *Clustered Index Scan + Sort* with a memory grant sized for sorting 11.4M rows. Every API call with 'Open' now reads all 12M rows and asks for that huge grant, about 200 times a minute. When several calls overlap, they also queue for memory (`RESOURCE_SEMAPHORE` waits). That fits "sometimes 20+ seconds".

2. **Why it's only *sometimes*.** A stored procedure gets one plan per cache entry, compiled from the first parameter value after a recompile. Several things trigger a recompile: an auto-update of statistics on `Tickets`, index or stats maintenance, plan eviction, a restart, or `sp_recompile`. If the nightly export with `'Closed'` is the first call after one of these, the API inherits the 'Closed' plan until the next recompile. If an 'Open' call comes first, the API is fast. Worth checking: does a stats or index maintenance job run just before the export?

3. **Why SSMS is fast and you can't reproduce it.**
   - SSMS connects with `SET ARITHABORT ON`. .NET SqlClient leaves it `OFF`. ARITHABORT is a plan-cache key, so your SSMS session gets **its own cache entry**, compiled fresh with `'Open'`, and never touches the app's 'Closed' plan.
   - Your test (`DECLARE @s ...; EXEC dbo.GetTicketsByStatus @s;`) is fine as far as the value goes. Passing a variable *to a procedure* still lets the procedure sniff 'Open'. (The "local variables give density estimates" problem only applies when the variable sits directly in the query's WHERE clause, not when it's passed to a proc.) The only difference is the separate cache entry.
   - Don't "fix" this by changing ARITHABORT in the app. That only changes which cache entry gets the bad plan.

4. **Something in the 'Closed' plan points to a second problem.** Your index `(Status, CreatedAt DESC) INCLUDE (Subject)` matches the query exactly by ESR (E = `Status`, S = `CreatedAt DESC`). If it is also **covering**, then even 'Closed' should get an *ordered Index Seek with no Sort and no memory grant*. That beats scanning the wider clustered index and then sorting. The optimizer picked Clustered Index Scan + Sort instead. That usually means **the index is not covering**, most likely because `TicketId` is not the clustered key (or the table is a heap). Then the seek would need 11.4M Key Lookups for 'Closed'. If so, fixing that removes most of the sensitivity to the parameter value (see A below).

---

## Issues found

| # | Where | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `WHERE Status = @Status` (data: 'Closed' 95%, 'Open' 0.075%) | Parameter sniffing: one cached plan serves two very different row counts | **High** | The plan compiled for 'Closed' scans 12M rows and sorts, while 'Open' needs only ~9k rows from an index seek | Make the plan shape the same for every value (A), and move the export to its own procedure (B) |
| 2 | `SELECT TicketId, Subject, CreatedAt` vs `INCLUDE (Subject)` | The index is probably not covering, if `TicketId` is not the clustered key | **High** (if confirmed) | Each matching row needs a Key Lookup. For 'Closed' that's 11.4M lookups, which pushes the optimizer to scan + sort | Extend the existing index: `INCLUDE (Subject, TicketId)` (A) |
| 3 | `ORDER BY CreatedAt DESC` in the 'Closed' plan | Sort with a large memory grant | **Medium** | The grant is sized for 11.4M rows and reused by every 'Open' call. Concurrent calls wait for memory. 2019 Standard has no memory grant feedback to shrink it | Goes away once the plan is an ordered seek on the index (no Sort operator) |
| 4 | Procedure body | No `SET NOCOUNT ON` | Low | One extra "rows affected" message per call. Minor, but free to fix | Add it |
| 5 | API path returns all ~9,000 'Open' rows, 200×/min | Unbounded result for an API | Low (question) | That's 1.8M rows/min sent to the app. Fine if the app really needs them all | Consider `TOP (@n)` / keyset paging if the UI pages |

Things that look suspicious but are fine here:
- **.NET `AddWithValue` sending `nvarchar`.** The procedure parameter is `varchar(20)`, so the value is converted once on the parameter side. There's no `CONVERT_IMPLICIT` on the column, assuming `Status` is `varchar`.
- **Statistics are not the problem.** The estimate exactly matches the compiled value ('Closed').

---

## Rewritten procedure

```sql
-- API path: only one statement, same logic
CREATE OR ALTER PROCEDURE dbo.GetTicketsByStatus
    @Status varchar(20)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TicketId, Subject, CreatedAt
    FROM dbo.Tickets
    WHERE Status = @Status
    ORDER BY CreatedAt DESC;
END;
GO

-- Nightly export: own procedure, so it never compiles the plan the API reuses
CREATE OR ALTER PROCEDURE dbo.ExportTicketsByStatus
    @Status varchar(20)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TicketId, Subject, CreatedAt
    FROM dbo.Tickets
    WHERE Status = @Status
    ORDER BY CreatedAt DESC
    OPTION (RECOMPILE);   -- runs once a night, compile cost is irrelevant
END;
GO
```

`CREATE OR ALTER` works on SQL Server 2016 SP1 and later, so it's fine on 2019.

---

## Recommendation (in order)

### A. First check whether the index is really covering

```sql
-- Which index is clustered, and on which columns?
SELECT i.name, i.type_desc, c.name AS col, ic.key_ordinal, ic.is_included_column
FROM sys.indexes i
JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
JOIN sys.columns c ON c.object_id = ic.object_id AND c.column_id = ic.column_id
WHERE i.object_id = OBJECT_ID('dbo.Tickets')
  AND (i.index_id <= 1 OR i.name = 'IX_Tickets_Status_CreatedAt')
ORDER BY i.index_id, ic.key_ordinal;

-- Column types (Status must be varchar, not nvarchar/char, for the seek to be clean)
EXEC sp_help 'dbo.Tickets';
```

- **If `TicketId` is the clustered key:** the index is already covering. Don't change it, and skip to B. (In that case, also capture the 'Closed' plan with `OPTION (RECOMPILE)` in a test, because a covering index should give an ordered seek. If it still scans, send me the plan.)
- **If it's not (clustered on something else, or a heap):** extend the existing index. Don't add a new one:

```sql
CREATE NONCLUSTERED INDEX IX_Tickets_Status_CreatedAt
ON dbo.Tickets (Status, CreatedAt DESC)
INCLUDE (Subject, TicketId)
WITH (DROP_EXISTING = ON, SORT_IN_TEMPDB = ON);
-- ONLINE = ON / RESUMABLE are Enterprise-only on SQL Server 2019 -> not available on Standard
```

- **ESR mapping:** E = `Status`, S = `CreatedAt DESC` (matches the ORDER BY, so no Sort), R = none. Covering via `INCLUDE (Subject, TicketId)`.
- **Expected effect:** every value gets the same plan: *Index Seek (Ordered, Backward = False) → no Sort, no Key Lookup, no memory grant.* 'Open' should be on the order of tens or hundreds of logical reads, and 'Closed' is a single ordered range read with no 11M-row sort. Sniffing stops mattering because one plan is good for every value.
- **Cost:** about the size of one extra `int`/`bigint` per row in an index you already maintain, so a few percent more space. Writes don't change in kind (it's the same index). **The build is offline on Standard edition, so it blocks writes to `Tickets` while it runs.** Do it in a quiet window and check free space in tempdb and the data file (roughly the size of the index again).
- **Rollback:** rerun the same statement with the original `INCLUDE (Subject)` and `DROP_EXISTING = ON`.

### B. Separate the export from the API path (do this either way)

Deploy the two procedures above and point the nightly job at `dbo.ExportTicketsByStatus`. The API procedure's plan is then compiled only from API calls, which are almost always 'Open'. That's the fix without hints: no `OPTIMIZE FOR`, no forced plans.

If the API ever calls with 'Pending' (~600k rows), test that value too after A. With a covering index the plan shape is the same, so it's fine.

Why not just put `OPTION (RECOMPILE)` on the API procedure? It would work: the query is trivial and compiles in milliseconds, even at 200/min. But it treats the symptom and costs CPU on every call. Keep it as a fallback if you can't do A and B.

### C. Immediate relief, before you deploy anything

Remove only the bad plan, not the whole cache:

```sql
EXEC sp_recompile N'dbo.GetTicketsByStatus';   -- next call recompiles (with 'Open' during the day)
```

If Query Store is on (it's available on Standard), you can also **force the good 'Open' plan** as a stopgap until A and B are deployed. Unforce it afterwards, because forced plans freeze a choice that can go wrong as data changes:

```sql
-- find query_id / plan_id
SELECT q.query_id, p.plan_id, p.is_forced_plan, rs.avg_duration/1000.0 AS avg_ms, rs.count_executions
FROM sys.query_store_query q
JOIN sys.query_store_plan p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
WHERE q.object_id = OBJECT_ID('dbo.GetTicketsByStatus');

EXEC sp_query_store_force_plan @query_id = <id>, @plan_id = <good plan id>;
-- later: EXEC sp_query_store_unforce_plan @query_id = <id>, @plan_id = <id>;
```

---

## Verify

**1. Reproduce the app's behavior in SSMS.** Compare the cache entries first:

```sql
SELECT ps.plan_handle, ps.cached_time, ps.execution_count,
       ps.last_elapsed_time/1000 AS last_ms, ps.max_elapsed_time/1000 AS max_ms,
       pa.value AS set_options      -- ARITHABORT ON = bit 4096 set (SSMS); app entry won't have it
FROM sys.dm_exec_procedure_stats ps
CROSS APPLY sys.dm_exec_plan_attributes(ps.plan_handle) pa
WHERE ps.database_id = DB_ID()
  AND ps.object_id = OBJECT_ID('dbo.GetTicketsByStatus')
  AND pa.attribute = 'set_options';
```

You should see two rows. `cached_time` on the app's row tells you whether the plan was compiled around the time of the nightly export. Then run it in SSMS with the app's settings so you reuse the app's plan:

```sql
SET ARITHABORT OFF;            -- match .NET SqlClient (check other set_options bits match too)
SET STATISTICS IO, TIME ON;
-- Ctrl+M (Include Actual Execution Plan)
EXEC dbo.GetTicketsByStatus @Status = 'Open';
```

To test the worst case on staging: `EXEC sp_recompile N'dbo.GetTicketsByStatus';`, then run `'Closed'` first and `'Open'` second, and the reverse order too. Before the fix, 'Open'-after-'Closed' is slow. After A and B, both orders should be fast.

**2. What a good plan looks like after the fix**, for 'Open', 'Pending' and 'Closed' in either compile order:
- `Index Seek` on `IX_Tickets_Status_CreatedAt`, with `Seek Predicate: Status = @Status` and `Ordered = True`
- **no** Sort operator, **no** Key Lookup, **no** memory grant (or a tiny one) in the plan root properties
- 'Open': estimated rows ≈ actual (~9k), logical reads in the low hundreds at most, not hundreds of thousands
- Send me the `.sqlplan` (or the STATISTICS IO output) before and after, and I'll check it.

---

## Assumptions / open questions

You don't need to answer these to start. A and B are safe either way, but the answers would confirm the diagnosis:

1. **Is `TicketId` the clustered key** of `dbo.Tickets`? Run the first query in A. This decides whether you need to extend the index.
2. **Data type of `Status`:** `varchar(20)`, `nvarchar`, or `char`?
3. **Is Query Store enabled** on this database? (`SELECT actual_state_desc FROM sys.database_query_store_options;`)
4. **Does a stats or index maintenance job run before the nightly export?** That's the likely trigger for the bad compile.
5. **When the API is slow, do you see `RESOURCE_SEMAPHORE` waits?** (`sys.dm_exec_query_memory_grants` during an incident.) That would confirm the memory-grant queueing.
6. Does the API really need all ~9,000 'Open' rows per call, or does it page?
