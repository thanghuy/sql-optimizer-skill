Your plan shows exactly where the time goes. The index is being used, but its column order serves the `WHERE` clause and not the `ORDER BY`. That means SQL Server has to fetch and sort every matching row just to return 20.

## Diagnosis

| Plan evidence | What it means |
|---|---|
| Index Seek returns **41,233 rows** for a `TOP (20)` | The index is in **E-R-S** order: `AssigneeId` (Equality), then `CreatedAt` (Range), then `Priority`. Rows come out ordered by `CreatedAt ASC`, but you need `Priority DESC, CreatedAt DESC`. So the engine has to read **every** match before it can pick 20. |
| **Sort** of 41,233 rows, **spill to tempdb** (level 1) | This is the "S" of ESR not being met. The spill happens because the memory grant was sized for the **estimated 1,850 rows**, while 41,233 arrived (22× more). |
| **Key Lookup × 41,233** | `Subject` is not in the index, so each of the 41k rows costs a clustered-index lookup of about 3 pages. That is roughly **124k of your 131,904 logical reads (~94%)**. The lookups happen *before* the sort, so they are paid for rows you then throw away. |
| Estimated 1,850 vs actual 41,233 | This is a cardinality problem: parameter sniffing (the plan was compiled for a smaller assignee), or a test run with `DECLARE`d local variables, which uses the density average. It turns into a spill only because there is a big sort. Remove the sort and the estimate barely matters. |

At 300 executions/min this works out to about 40M logical reads/min, plus a tempdb spill on every run for busy agents.

**Why `(AssigneeId, CreatedAt, Priority)` can't fix it:** after a range on `CreatedAt`, the rows are no longer ordered by `Priority`. A column placed after a range column can only filter; it can't provide order.

## Recommendation

### 1. Index in ESR order (the main fix)

```sql
CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Priority_Created
ON dbo.Tickets (AssigneeId ASC, Priority DESC, CreatedAt DESC)
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);   -- ONLINE needs Enterprise; on Standard build off-peak (it blocks writes)
-- 2019 Enterprise can also add RESUMABLE = ON (with ONLINE = ON) for an 18M-row build.
```

**ESR mapping:**
- **E** = `AssigneeId`: the seek starts here.
- **S** = `Priority DESC, CreatedAt DESC`: matches the `ORDER BY` exactly, so there is no Sort operator, no memory grant and no spill. `TOP (20)` can stop early.
- **R** = `CreatedAt >= @Since`: `CreatedAt` is already a key column (as S), so the range is checked inside the index as a residual predicate, without touching the table.

**Why I'm not adding `INCLUDE (Subject)` yet:** with an ordered index, the Key Lookup runs only for the ~20 rows that are returned, not 41k. That costs about 60 reads. Including `Subject` (nvarchar(400)) would save those ~60 reads but add roughly `18M × avg Subject bytes`, likely 2–4 GB. It would also make every Subject edit maintain this index. Add it later only if the new plan shows the lookup still matters.

**One weakness of the plain index:** within each priority the newest tickets come first, so recent rows are read first, which is good. But if the highest priority has **fewer than 20 tickets since @Since**, the scan keeps reading that priority's *old* tickets, which don't match, before it moves on to the next priority. For an agent with 400k tickets, this can occasionally mean reading many thousands of narrow index rows. That is still far cheaper than today, but it is unpredictable. Rewrite 2 removes this weakness.

### 2. Query rewrite so `CreatedAt` is seekable too (recommended if `Priority` has a small fixed set of values)

This rewrite runs one tiny seek per priority on `(AssigneeId = ?, Priority = ?, CreatedAt >= ?)` and merges the results:

```sql
SELECT TOP (20) t.TicketId, t.Subject, t.Priority, t.CreatedAt
FROM (VALUES (CAST(4 AS tinyint)), (3), (2), (1)) AS p(Priority)   -- use your real values/type, or a dbo.Priorities lookup table
CROSS APPLY (
    SELECT TOP (20) x.TicketId, x.Subject, x.Priority, x.CreatedAt
    FROM dbo.Tickets AS x
    WHERE x.AssigneeId = @AssigneeId
      AND x.Priority   = p.Priority
      AND x.CreatedAt >= @Since
    ORDER BY x.CreatedAt DESC
) AS t
ORDER BY t.Priority DESC, t.CreatedAt DESC;
```

- Each branch is a pure seek (all three columns are in the Seek Predicate) and reads at most 20 rows. The total is ≤ 20 × (number of priorities), e.g. ≤ 80 index rows and ≤ 80 lookups.
- The final sort handles ≤ 80 rows, so its memory grant is tiny and it won't spill.
- It gives the same result as your query, with two conditions. **(a)** The list must contain every priority value. Use a lookup table if new priorities can appear. **(b)** If `Priority` is nullable, add a branch for `Priority IS NULL`. NULLs sort last under `DESC`, so they would only appear when fewer than 20 non-NULL rows exist.
- Ties on `(Priority, CreatedAt)` are returned in no guaranteed order in both versions. If paging must be stable, add `TicketId DESC` to the `ORDER BY`.

**Expected effect (both options):** logical reads drop from **131,904 to roughly 70–300**, with no Sort, no spill and CPU in the low single-digit ms. Option 2 gives the more predictable worst case.

**Cost:** one new narrow index. It holds about 17 bytes of key plus the clustering key per row, roughly 400–600 MB for 18M rows. Every INSERT, and every UPDATE of `AssigneeId`, `Priority` or `CreatedAt`, maintains it. Helpdesks reassign and re-prioritize a lot, so check the write rate (questions below).

**Rollback:** `DROP INDEX IX_Tickets_Assignee_Priority_Created ON dbo.Tickets;`, and revert the query text.

### 3. Then deal with the old index (don't drop it blindly)

`IX_Tickets_Assignee_Created` may become redundant. But it is still the right index for queries such as "tickets of an assignee in a date range, ordered by date". Check before removing it:

```sql
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates, s.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
  ON s.object_id = i.object_id AND s.index_id = i.index_id AND s.database_id = DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.Tickets');
```

If it shows no seeks after a week or two of normal traffic, and Query Store shows no other query using it, run `ALTER INDEX IX_Tickets_Assignee_Created ON dbo.Tickets DISABLE;` first. Drop it later.

### 4. The estimate gap (secondary, but check it)

- Check how you produced the plan. If you ran it in SSMS with `DECLARE @AssigneeId ...`, the optimizer can't sniff local variables. It uses the average density, which may be where the 1,850 comes from, and the app's plan may differ. Reproduce the app with `sp_executesql` and real parameters, or pull the app's actual plan from Query Store or the plan cache.
- Check stats freshness: `SELECT * FROM sys.dm_db_stats_properties(OBJECT_ID('dbo.Tickets'), <stats_id>)`, looking at `last_updated` and `modification_counter`. Recent `CreatedAt` values falling beyond the last histogram step is a classic cause of underestimates.
- At compat 150 you do get **row-mode memory grant feedback**. But it [disables itself for parameter-sensitive queries whose memory needs oscillate](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing-memory-grant-feedback). Busy versus quiet agents is exactly that case. The persistent/percentile version and Parameter Sensitive Plan optimization are SQL Server 2022 only, so on 2019 the real fix is removing the sort.
- Use `OPTION (RECOMPILE)` only as a diagnostic, not as a permanent fix at 300 executions/min.

## Verify

Build the index on staging with production-sized data, then run both a **typical agent** and the **busiest agent** the way the app does:

```sql
SET STATISTICS IO, TIME ON;
EXEC sp_executesql
  N'<query text exactly as the app sends it>',
  N'@AssigneeId int, @Since datetime2',      -- match the app's parameter types
  @AssigneeId = <busy agent>, @Since = <30 days ago>;
SET STATISTICS IO, TIME OFF;
```

Use "Include Actual Execution Plan" and save the plan as a `.sqlplan` file if you can.

What a good plan looks like:
- `Index Seek` on `IX_Tickets_Assignee_Priority_Created`. For option 2, the Seek Predicate shows `AssigneeId`, `Priority` **and** `CreatedAt`.
- **No Sort operator** for option 1, or only a small Top N Sort of ≤ 80 rows for option 2. No tempdb spill warning.
- *Number of Rows Read* close to *Actual Number of Rows*: about 20 for option 1 in the good case, ≤ 80 for option 2.
- Key Lookup executions ≤ 20–80.
- `Tickets` logical reads in the low hundreds or below, and elapsed time in single-digit ms.

| Metric | Before | After (fill in) |
|---|---|---|
| Logical reads | 131,904 | |
| CPU / elapsed | 312 / 655 ms | |
| Rows read from index | 41,233 | |
| Key Lookups | 41,233 | |
| Sort / spill | yes / yes | |

Send me the new plan and these numbers. If option 1 still reads thousands of rows for busy agents, that is the low-count-top-priority case, and option 2 fixes it.

## Assumptions / open questions

1. **`Priority`:** what is its data type, how many distinct values does it have, and is it nullable? Option 2 assumes a small fixed set like 1–4.
2. How are recent tickets for a busy agent spread across priorities? For example, are there usually ≥ 20 at the top priority? This decides whether option 1 alone is enough.
3. **Edition:** Enterprise or Standard? That decides whether `ONLINE = ON` is available or the build needs a maintenance window.
4. **Write rate on `Tickets`:** inserts/sec, and how often `AssigneeId` or `Priority` change.
5. Average `DATALENGTH(Subject)`, only if we later consider `INCLUDE (Subject)`.
6. Does the app send this query as a parameterized `sp_executesql` or stored procedure, and with which parameter types? An `nvarchar` or `datetime` parameter against a different column type can cause implicit conversions.
7. Are other hot queries using `IX_Tickets_Assignee_Created`? This decides whether it can be dropped.

I haven't run anything against your database. All DDL above is for you to run, staging first.
