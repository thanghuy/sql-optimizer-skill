## Short answer

The index you were given has its columns in the wrong order for this query. `(AssigneeId, CreatedAt, Priority)` lets SQL Server *find* the rows quickly, but it hands them back sorted by `CreatedAt`, and you asked for `Priority DESC, CreatedAt DESC`. So the engine has to:

1. read **every** matching row for the agent in the last 30 days (41,233 rows), not just 20,
2. do a **Key Lookup for each one** to get `Subject` (41,233 lookups at about 3 page reads each comes to about 124k of your 131,904 logical reads),
3. **sort all 41k rows** to find the top 20. The memory grant was sized for 1,850 rows, so the sort spilled to tempdb.

`TOP (20)` only saves work when rows already come out of the index in the final `ORDER BY` order. Here they don't, so the TOP does nothing until everything has been read, looked up and sorted.

At ~300 executions/min × 312 ms CPU, this one query keeps about **1.5 CPU cores busy all the time**, so fixing it is worth doing.

## What's wrong, item by item

| Symptom in the plan | Cause |
|---|---|
| Index Seek returns 41,233 rows for a TOP 20 | The index order (`CreatedAt` after `AssigneeId`) doesn't match `ORDER BY Priority, CreatedAt`, so there's no early exit |
| Key Lookup × 41,233 (most of the 131,904 reads) | `Subject` isn't in the index, and the lookups run *before* the Sort/Top, so every candidate row gets one |
| Sort spilling to tempdb (level 1) | The memory grant was based on 1,850 estimated rows against 41,233 actual |
| Estimated 1,850 against actual 41,233 | Either the plan was compiled (sniffed) for a quieter agent or an older `@Since`, or the parameters are local variables, so the optimizer used average density × a fixed 30% guess for `>=`. For example, 18M rows / ~2,900 assignees × 0.3 ≈ 1,850. Busy agents get a plan sized for an average one |

Fixing the estimate alone (for example with `OPTION (RECOMPILE)`) would stop the spill, but you'd still read 41k rows, do 41k lookups and sort them. The real fix is to make the index return rows in `ORDER BY` order.

## What to do

### Option 1: an index that matches the ORDER BY (simplest change)

```sql
CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Priority_Created
ON dbo.Tickets (AssigneeId, Priority DESC, CreatedAt DESC)
-- INCLUDE (Subject)   -- optional, see below
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);   -- ONLINE needs Enterprise edition
```

Now the plan is: seek on `AssigneeId`, read in `Priority DESC, CreatedAt DESC` order, apply `CreatedAt >= @Since` as a residual filter, and **stop after 20 qualifying rows**. There's no Sort, no spill, and at most ~20 lookups (none if you INCLUDE `Subject`).

**The catch:** `CreatedAt` is no longer a seek predicate. Within each priority the newest rows come first. But if the highest priority has fewer than 20 tickets in the window, the engine reads **all of that priority's older tickets** (possibly years of them, out of 400k) before moving to the next priority. Usually that's still far cheaper than now. For agents with many old high-priority tickets and few recent ones it can cost thousands of rows. Test with your worst-case agent.

### Option 2: per-priority seeks (bounded cost every time, recommended if Priority has few values)

If `Priority` has a small, fixed set of values (e.g. 1–5), you can make the cost independent of history. Do one tiny seek per priority, then merge:

```sql
-- Index: key order allows Assignee = , Priority = , CreatedAt >= as a full seek
CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Priority_Created
ON dbo.Tickets (AssigneeId, Priority, CreatedAt)
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);

SELECT TOP (20) x.TicketId, x.Subject, x.Priority, x.CreatedAt
FROM (VALUES (1),(2),(3),(4),(5)) AS p(Priority)      -- or SELECT Priority FROM dbo.Priorities
CROSS APPLY (
    SELECT TOP (20) t.TicketId, t.Subject, t.Priority, t.CreatedAt
    FROM dbo.Tickets AS t
    WHERE t.AssigneeId = @AssigneeId
      AND t.Priority   = p.Priority
      AND t.CreatedAt >= @Since
    ORDER BY t.CreatedAt DESC
) AS x
ORDER BY x.Priority DESC, x.CreatedAt DESC;
```

Each inner query is an exact seek (`AssigneeId =`, `Priority =`, `CreatedAt >=` range) read backwards. It touches at most 20 rows, so in total that's ≤ 100 rows, ≤ 100 lookups and a sort of ≤ 100 rows (tiny grant, no spill). Expect logical reads in the low hundreds instead of 131,904, and CPU in the low single-digit ms. The plan shape is the same for a quiet agent and a busy one, so parameter sniffing stops mattering.

(The same index also works for Option 1's query, in a backward scan. Choose Option 1 or 2 by how much you can change the app's SQL.)

### About INCLUDE (Subject)

`Subject` is `nvarchar(400)`. Including it in an 18M-row index can add several GB and more write cost on every insert/update of `Subject`. Once the plan touches only ≤ 20–100 rows, the remaining Key Lookups cost ~60–300 reads per call, which is cheap. I'd **leave it out** at first and add it only if the lookups still show up as significant after the change.

### About the existing index

Don't drop `IX_Tickets_Assignee_Created` yet. Check whether anything else uses it first:

```sql
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates, s.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
  ON s.object_id = i.object_id AND s.index_id = i.index_id AND s.database_id = DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.Tickets');
```

If only this query used it, replace it with the new index rather than keeping both (each extra index slows inserts into an 18M-row hot table).

## Also check

- **How the app sends the parameters.** If the query is built with `DECLARE @AssigneeId ...; SET ...` inside a batch (local variables), that's where the "average density" estimate comes from. Real parameters (`sp_executesql`, or parameterized commands from the driver) are better. With Option 2 the plan barely depends on the estimate either way.
- **Deterministic ordering.** `Priority DESC, CreatedAt DESC` can tie. If the app pages through results, add `TicketId DESC` as a last tiebreaker (and as the last index key if you want it covered by the order).

## How to verify

Run with `SET STATISTICS IO, TIME ON` and the actual plan, using **your busiest agent** and also an agent with few recent high-priority tickets (Option 1's worst case). You should see:

- no Sort operator (Option 1), or a Sort of ≤ 100 rows with no warning (Option 2),
- Key Lookup executions ≤ 20 (Option 1) or ≤ 100 (Option 2),
- logical reads in the tens or low hundreds, not 131,904,
- CPU time in single-digit ms.

Once it's deployed, compare the query's average CPU and reads before and after in Query Store.

## Questions that would sharpen this

1. What type is `Priority` and how many distinct values does it have? That decides whether Option 2 works and what goes in the `VALUES` list.
2. Can you change the query text in the app, or only the indexes? If only the indexes, use Option 1.
3. Which edition are you on (`ONLINE = ON` needs Enterprise), and do other queries rely on `IX_Tickets_Assignee_Created`?

My default if you can't answer yet: use Option 2 if Priority has a handful of fixed values and you can change the SQL, otherwise Option 1. Either way, leave out `INCLUDE (Subject)` at first.
