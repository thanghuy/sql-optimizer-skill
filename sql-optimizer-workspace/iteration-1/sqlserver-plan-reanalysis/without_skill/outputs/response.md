Your plan has three problems. They all come from one cause: the index's key order doesn't match the query's `ORDER BY`.

## What's wrong

**1. The index can't return rows in the order the query wants, so `TOP (20)` can't stop early.**

The index is keyed `(AssigneeId, CreatedAt, Priority)`. After the seek on `AssigneeId = @AssigneeId AND CreatedAt >= @Since`, rows come out ordered by `CreatedAt`. The query wants them ordered by `Priority DESC, CreatedAt DESC`. Having `Priority` as the third key doesn't help, because it is only sorted *within* each identical `CreatedAt` value, which is almost useless for a datetime column.

So SQL Server has to read **every** matching row (41,233), sort them all, and then keep 20. The `TOP` saves nothing.

**2. It does 41,233 key lookups to fetch `Subject` for rows it will throw away.**

`Subject` isn't in the index, so each of the 41k rows gets a Key Lookup into the clustered index. Each lookup costs about 3 page reads (an 18M-row B-tree is about 3 levels deep). 41,233 × ~3 ≈ 124k, which accounts for nearly all of your **131,904 logical reads**. The index seek itself is only a few hundred pages. In other words, about 95% of the I/O goes to fetching `Subject` for 41,213 rows that never reach the client.

**3. A bad row estimate leads to a memory grant that is too small, so the Sort spills to tempdb.**

The optimizer estimated 1,850 rows and got 41,233, about 22x more. The Sort's memory grant was sized for 1,850 rows that each carry an `nvarchar(400)`, so it spilled to tempdb. That spill is a big part of why elapsed time (655 ms) is about double CPU time (312 ms). The usual causes:
- **Parameter sniffing:** the cached plan was compiled for a quiet agent or a short `@Since` window, then reused for a busy one. Check `ParameterCompiledValue` against `ParameterRuntimeValue` in the actual plan XML (right-click the plan → Show Execution Plan XML, near the bottom).
- **Stale or low-sample statistics** on an 18M-row table where one agent owns 400k rows.

Even with a perfect estimate, this plan shape would still be slow because of #1 and #2. Fixing the estimate only removes the spill. **Fix the index first, and the spill and sniffing issues go away**, because the new plan has no Sort and no memory grant.

At 300 executions/min, the current plan costs about 40M logical reads per minute and burns a full CPU core (312 ms × 5/sec ≈ 1.5 s of CPU per second).

## What to do

### Step 1: An index whose key order matches the ORDER BY, covering `Subject`

```sql
CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Priority_Created
ON dbo.Tickets (AssigneeId, Priority DESC, CreatedAt DESC)
INCLUDE (Subject)
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);   -- ONLINE needs Enterprise edition
```

Why this works:
- With equality on `AssigneeId`, the index delivers rows already in `Priority DESC, CreatedAt DESC` order. The plan becomes `Top → Index Seek (ordered)`, with no Sort, no memory grant and no spill.
- `TicketId` is the clustering key, so it is already in every nonclustered index. `INCLUDE (Subject)` removes the Key Lookup.
- The `CreatedAt >= @Since` test becomes a residual predicate. Within each `Priority` value, rows are in `CreatedAt DESC` order, so recent tickets are read first.

**One caveat:** the residual predicate is cheap only if the highest-priority bucket has at least 20 tickets since `@Since`. If an agent has thousands of *old* top-priority tickets and few recent ones, the scan walks through all of those old rows before it reaches the next priority (SQL Server 2019 has no skip-scan). If that can happen in your data, use Step 2. It is safe regardless of the data.

### Step 2 (most robust): Seek each priority separately, then merge

`Priority` is almost certainly a small set of values. If so, do one tight seek per priority and merge them. The total read is at most 20 × (number of priorities) index rows, whatever the data looks like:

```sql
-- index: (AssigneeId, Priority, CreatedAt) INCLUDE (Subject), with key direction not important here
SELECT TOP (20) x.TicketId, x.Subject, x.Priority, x.CreatedAt
FROM (VALUES (4),(3),(2),(1)) AS p(Priority)      -- or SELECT Priority FROM dbo.Priorities
CROSS APPLY (
    SELECT TOP (20) t.TicketId, t.Subject, t.Priority, t.CreatedAt
    FROM dbo.Tickets AS t
    WHERE t.AssigneeId = @AssigneeId
      AND t.Priority   = p.Priority
      AND t.CreatedAt >= @Since
    ORDER BY t.CreatedAt DESC
) AS x
ORDER BY x.Priority DESC, x.CreatedAt DESC, x.TicketId DESC;
```

Each inner query is a seek on `(AssigneeId, Priority, CreatedAt >= @Since)` that reads backward and stops after 20 rows. The outer Sort handles at most 80 rows, a trivial grant that can't realistically spill. Expect **tens of logical reads** instead of 131,904. The index from Step 1 also serves this query; only the leading equality columns matter for the seek.

Two things to check:
- **NULL priorities:** if `Priority` is nullable, the `VALUES` list misses NULL rows. Add a branch for `Priority IS NULL` if those tickets should appear (they sort last under `DESC`).
- **Unknown priority values:** if the set of priorities isn't fixed, drive it from a lookup table rather than hard-coding it.

### Optional: Keep the index narrow and look up `Subject` only for the final 20

`Subject` is up to 800 bytes. If adding it to the index is too costly (index size, or write overhead from Subject edits), leave it out and fetch it after the `TOP`:

```sql
WITH top20 AS (
    SELECT TOP (20) TicketId, Priority, CreatedAt
    FROM dbo.Tickets
    WHERE AssigneeId = @AssigneeId AND CreatedAt >= @Since
    ORDER BY Priority DESC, CreatedAt DESC, TicketId DESC
)
SELECT t.TicketId, t.Subject, top20.Priority, top20.CreatedAt
FROM top20
JOIN dbo.Tickets AS t ON t.TicketId = top20.TicketId
ORDER BY top20.Priority DESC, top20.CreatedAt DESC, top20.TicketId DESC;
```

That is 20 key lookups (about 60 reads) instead of 41k. The same pattern works with the per-priority `CROSS APPLY`.

### Step 3: Decide what to do with the old index

Before dropping `IX_Tickets_Assignee_Created`, check whether anything else uses it, such as queries that filter by assignee and sort by `CreatedAt`:

```sql
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates, s.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
  ON s.object_id = i.object_id AND s.index_id = i.index_id AND s.database_id = DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.Tickets');
```

You can also search Query Store for plans that reference it. If only this query used it, drop it so you don't pay write overhead on two similar indexes.

### Step 4: Verify

Run the query for your busiest agent with `SET STATISTICS IO, TIME ON` and the actual plan. You should see:
- No Sort operator and no Key Lookup (Step 1), or a Sort of 80 rows or fewer (Step 2).
- Logical reads in the tens, not hundreds of thousands.
- CPU time near 0 ms and no spill warning.

Also test a quiet agent and a long `@Since` window. The point of these shapes is that the plan stays good for any parameter values, so you don't need `OPTION (RECOMPILE)` or plan forcing.

### Things to keep in mind

- **Write cost:** an index keyed on `Priority` moves a row whenever its priority changes, and an included `Subject` is rewritten whenever the subject is edited. On a helpdesk table both changes are infrequent compared with 300 reads/min, so this is usually a good trade, but check your update rate.
- **Ties:** `ORDER BY Priority DESC, CreatedAt DESC` isn't deterministic when two tickets share both values. I added `TicketId DESC` as a tiebreaker so paging and repeated calls give stable results. It costs nothing, because `TicketId` is already in the index.
- **Statistics:** worth a one-off `UPDATE STATISTICS dbo.Tickets WITH FULLSCAN` (or a higher sample rate) if the estimates still look far off. With the new plan shape this matters far less.

**Summary:** the advice you followed put `CreatedAt` before `Priority`. That suits the `WHERE` clause but not the `ORDER BY`, and with `TOP` the `ORDER BY` is what decides the cost. Put `Priority` right after `AssigneeId`, cover `Subject`, and if old high-priority tickets can pile up, use the per-priority `CROSS APPLY` so the cost stays small however the data is distributed.
