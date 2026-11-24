## Short answer: no, you don't need another index

Your plan-cache numbers already show the cause. This is **parameter sniffing**. The plan the API is using was compiled for `'Closed'` (`ParameterCompiledValue = 'Closed'`, estimated 11.4M rows), and it keeps getting reused for `'Open'` (`ParameterRuntimeValue = 'Open'`, actual 9,041 rows). The nightly export, or any recompile that happens while `'Closed'` is the value being passed, left a plan in cache that was built for 95% of the table. Every `'Open'` call after that pays for it:

- **Clustered Index Scan:** it reads all 12M rows to find 9k.
- **Sort:** it sorts rows that `IX_Tickets_Status_CreatedAt` already returns in `CreatedAt DESC` order.
- **A memory grant sized for sorting ~11.4M rows:** at ~200 calls/min, many calls ask for a large grant at the same time. They queue on `RESOURCE_SEMAPHORE` waits. That queueing is probably where much of your 20+ seconds goes, more than the scan itself.

With a plan compiled for `'Open'` you get an Index Seek on `IX_Tickets_Status_CreatedAt`, no Sort, a tiny or zero memory grant, and a few milliseconds per call.

## Why SSMS is fast: you get a different plan

SSMS connects with `SET ARITHABORT ON` by default. ADO.NET / SqlClient connects with `ARITHABORT OFF`. `SET` options are part of the plan cache key, so your SSMS session can't reuse the API's cached plan. It compiles its own, sniffs `'Open'`, and gets the good seek plan. The proc and the data are the same, but the cache entry is not.

(Your `DECLARE @s ...; EXEC dbo.GetTicketsByStatus @s;` test is fine. The value is still passed as a real parameter. The difference is the `SET` options, not the variable.)

### How to reproduce the slow behaviour in SSMS

```sql
SET ARITHABORT OFF;   -- match the .NET connection so you hit the API's cached plan
EXEC dbo.GetTicketsByStatus @Status = 'Open';
SET ARITHABORT ON;
```

You can also look at the cache directly. You should see two plans for the proc, with different `set_options`:

```sql
SELECT  ps.plan_handle, ps.execution_count, ps.last_elapsed_time, ps.max_elapsed_time,
        pa.value AS set_options, qp.query_plan
FROM    sys.dm_exec_procedure_stats AS ps
CROSS APPLY sys.dm_exec_plan_attributes(ps.plan_handle) AS pa
CROSS APPLY sys.dm_exec_query_plan(ps.plan_handle) AS qp
WHERE   ps.object_id = OBJECT_ID('dbo.GetTicketsByStatus')
  AND   pa.attribute = 'set_options';
```

Don't "fix" this by turning `ARITHABORT ON` in the app's connection string. It only gives the app a fresh cache entry, and it goes bad again the next time `'Closed'` is the value compiled into it.

## Immediate relief

Evict the bad plan so the next API call compiles with `'Open'`:

```sql
EXEC sp_recompile N'dbo.GetTicketsByStatus';
-- or, more surgically:  DBCC FREEPROCCACHE (<plan_handle from the query above>);
```

This is temporary. The problem comes back the next time a recompile happens to land on the export call. Stats updates on 12M rows, failovers, restarts and memory pressure can all trigger a recompile at the wrong moment.

## Permanent fix (pick one; I'd do #1)

**1. Give the nightly export its own proc (recommended).** The two workloads really are different queries: one returns 9k rows 200 times a minute, the other returns 11.4M rows once a night. Keep `dbo.GetTicketsByStatus` for the API and pin its plan to the common case:

```sql
ALTER PROCEDURE dbo.GetTicketsByStatus @Status varchar(20)
AS
SELECT TicketId, Subject, CreatedAt
FROM dbo.Tickets
WHERE Status = @Status
ORDER BY CreatedAt DESC
OPTION (OPTIMIZE FOR (@Status = 'Open'));
```

Point the export at a separate proc, for example `dbo.ExportTicketsByStatus`, with the same query and either no hint or `OPTION (RECOMPILE)`. It runs once a night, so compile cost doesn't matter. The two procs have separate cache entries and can't poison each other.

If the export must keep calling the same proc, use a small wrapper that branches to two sub-procs: `IF @Status = 'Closed' EXEC dbo.GetTickets_Large ... ELSE EXEC dbo.GetTickets_Small ...`. The branching has to happen across separate procs (or separate dynamic-SQL statements). An `IF` with two copies of the `SELECT` inside one proc does not work, because both statements are compiled together with the same sniffed value.

**2. `OPTIMIZE FOR (@Status = 'Open')` alone, if the export is OK with the seek plan.** If `IX_Tickets_Status_CreatedAt` really covers the query (see the check below), the `'Open'` plan is a seek plus an ordered range scan with no Sort. That is also a perfectly good plan for `'Closed'`: an ordered read of 11.4M index rows with no sort and no big grant. In that case the one-line hint is enough. If the index does not cover, the `'Open'` plan has Key Lookups, and 11.4M lookups at night would hurt. In that case use #1.

**3. `OPTION (RECOMPILE)`.** This is correct for every value, and for a single-table query like this the compile cost is small. Still, 200 compiles/min is CPU you don't need to spend when one value dominates. It's acceptable, but #1 or #2 is cleaner.

**4. Query Store plan forcing.** Query Store is available in 2019 Standard. Once the good seek plan is in Query Store, you can force it with `sp_query_store_force_plan` without changing any code. This is a good stopgap, or the right fix if you can't deploy a proc change quickly.

**Don't use** `OPTIMIZE FOR UNKNOWN` or the "copy the parameter into a local variable" trick here. Both make the optimizer use the average density: 12M rows / 3 values, about 4M rows. That estimate still picks the scan + sort + big grant for every call. Parameter Sensitive Plan optimization, which would handle this automatically, needs SQL Server 2022 at compat level 160. You don't get it on 2019 / 150.

## One thing to double-check about the index

With `(Status, CreatedAt DESC) INCLUDE (Subject)`, the optimizer can only use the index without lookups if `TicketId` is the **clustered index key**. Nonclustered indexes carry the clustering key automatically. Even for `'Closed'` it would normally prefer an ordered scan of this narrower index over a Clustered Index Scan + Sort. The fact that it chose scan + sort makes me want you to confirm:

- `TicketId` is the clustering key. If the table is clustered on something else, add `TicketId` to the `INCLUDE` list. That is the only index change I'd consider.
- `Status` really is `varchar`, not `nvarchar`, and the parameter type matches it. A type or collation mismatch with an implicit conversion can stop the seek.
- The index isn't disabled or filtered.

If all three are fine, no index change is needed. Apply the recompile now, then fix #1 (or #2) so the export can't poison the API plan again.
