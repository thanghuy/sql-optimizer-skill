Thanks. You sent the version (SQL Server 2019, compat 150), the index, the query, row counts, traffic and an **actual** plan with STATISTICS IO/TIME, which is everything needed to diagnose this. The plan shows three separate problems. One of them is the root cause.

## Diagnosis

Your index is `(AssigneeId, CreatedAt, Priority)`. In ESR terms that is **E-R-S** (Equality, Range, Sort), but the query needs **E-S-R**:

| Query part | Column | ESR role |
|---|---|---|
| `AssigneeId = @AssigneeId` | AssigneeId | **E** |
| `ORDER BY Priority DESC, CreatedAt DESC` | Priority, CreatedAt | **S** |
| `CreatedAt >= @Since` | CreatedAt | **R** (also the second sort column) |

Here is what each part of the plan tells us.

1. **Sort plus tempdb spill (root cause).** Inside one AssigneeId, your index stores rows ordered by `CreatedAt`, not by `Priority`. SQL Server therefore has to read **every** matching row (41,233), sort them all, and only then keep 20. `TOP (20)` cannot stop the scan early because the rows do not come out in `ORDER BY` order. ESR says the sort column belongs right after the equality column, and that is the S part your index is missing.

2. **41,233 Key Lookups make up nearly all the logical reads.** The index has no `Subject`, so each of the 41k rows triggers a Key Lookup into the clustered PK. `TicketId` is not the issue, because it is the clustering key and every nonclustered index already carries it. 41,233 lookups × about 3 pages per B-tree descent ≈ 124k pages, which matches almost all of your **131,904 logical reads**. The lookups run *before* the Sort and Top, so you pay for 41k lookups to return 20 rows.

3. **Estimated 1,850 vs actual 41,233 rows (22× off).** That gap is a cardinality problem, not an index problem. The Sort's memory grant was sized for about 1,850 rows, so it spilled to tempdb (spill level 1). Likely causes, from most to least likely:
   - **Parameter sniffing.** The cached plan was compiled for a quieter assignee, and a busy one (400k tickets) reuses it.
   - **`@AssigneeId` / `@Since` are local variables** (`DECLARE @x ...; SELECT ...`) rather than real parameters. SQL Server then uses the average density instead of the histogram, which gives roughly "average assignee" rows.
   - Stale statistics on `AssigneeId` / `CreatedAt`. The last 30 days are the part of the histogram that is always outdated (the ascending-key problem).

   Row-mode memory grant feedback (available on 2019 at compat 150) can grow the grant on later runs. With one cached plan shared by busy and quiet agents, though, the grant keeps flipping between too big and too small, and the feedback can switch itself off. Parameter Sensitive Plan optimization would help here, but it needs **SQL Server 2022 / compat 160**, so it is not available to you. ([Microsoft Learn: Intelligent query processing](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing))

The good news: if the index gives rows in `ORDER BY` order, there is **no Sort**. With no Sort there is no memory grant and no spill, and the bad estimate stops mattering for this query. So fix #1 first and #2 and #3 mostly go away.

## Recommendation

### 1. Replace the index with an ESR index (main fix)

```sql
CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Priority_Created
ON dbo.Tickets (AssigneeId, Priority DESC, CreatedAt DESC)
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);   -- ONLINE needs Enterprise/Developer edition; RESUMABLE = ON is also allowed for CREATE INDEX on 2019 (with ONLINE = ON)
```

**ESR mapping**
- **E** = `AssigneeId`: seek to one agent.
- **S** = `Priority DESC, CreatedAt DESC`: rows come out in exactly the `ORDER BY` order and direction, so there is no Sort operator and `TOP (20)` stops after 20 qualifying rows.
- **R** = `CreatedAt >= @Since`: it is no longer a seek predicate. It becomes a **residual predicate evaluated inside the index**, before any Key Lookup. Only rows that pass it get looked up.
- `TicketId` (PK) is included automatically as the row locator.

**Expected effect.** About 1 in 10 of a busy agent's tickets are inside the window (40k of 400k), so with ESR the seek reads roughly `20 × 400k / 40k ≈ 200` index rows (formula from the ESR rule: k × N_e / N_er). That means:
- **about 20 Key Lookups instead of 41,233**
- no Sort, no memory grant, no tempdb spill
- logical reads go from about 132k to **tens, maybe a few hundred**

**Worst case to know about.** Within each priority the index is ordered by `CreatedAt DESC`. If the top priority has fewer than 20 tickets in the window, SQL Server reads that priority's *older* tickets too (and discards them in the index) before moving to the next priority. Even then the reads are cheap. This index row is about 25–30 bytes (≈ 250–300 rows per page), so in the absolute worst case, the agent's full 400k-row history, it reads about 1,500 pages with **no** lookups. That is still about 90× fewer reads than today. If that worst case matters at 300/min, use the rewrite in #3.

**Why I do *not* INCLUDE `Subject`.** `Subject` is `nvarchar(400)` (up to 800 bytes). Adding it to an 18M-row index adds roughly (average subject bytes × 18M) to the index: about 2 GB at a 60-character average subject. It also widens every page the seek reads and makes every Subject edit write to the index too. With an ESR index only about 20 rows need a lookup per execution, so that is about 60 extra page reads to avoid a multi-GB covering index. It is not worth it. (Without ESR order, covering would have been the only way to make 41k lookups cheap, which is why the missing-index DMV/hint probably suggested it.)

**Cost**
- One extra 18M-row narrow index while both exist, roughly the size of your current index (a few hundred MB). Plan for about that much again in tempdb/log during the build.
- Write overhead is about the same as the current index, because it is a replacement and not an extra index. One difference: an `UPDATE` that changes `Priority` now moves the row inside this index (with the old index it changed only a trailing key column). If tickets are re-prioritized very often, tell me the rate.
- Build it outside the helpdesk peak.

**Replace, don't stack.** The old index `IX_Tickets_Assignee_Created` becomes redundant for this query. Before dropping it, check whether another query relies on its `(AssigneeId, CreatedAt)` order. One example would be "my tickets since X ordered by CreatedAt" without Priority.

```sql
-- usage since last restart
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE s.database_id = DB_ID() AND s.object_id = OBJECT_ID('dbo.Tickets');
-- and search Query Store for other queries on dbo.Tickets filtering by AssigneeId
```
If nothing else uses it, disable it first, watch for a few days, then drop it:
```sql
ALTER INDEX IX_Tickets_Assignee_Created ON dbo.Tickets DISABLE;   -- reversible: ALTER INDEX ... REBUILD
-- later
DROP INDEX IX_Tickets_Assignee_Created ON dbo.Tickets;
```
**Rollback for the new index:** `DROP INDEX IX_Tickets_Assignee_Priority_Created ON dbo.Tickets;` (rebuild the old one first if you already dropped it).

### 2. Check the estimate problem (cheap, do it together with #1)

- Confirm how the app sends the query. Check whether it is `sp_executesql` with typed parameters or a batch with `DECLARE`d local variables. With local variables you get density-based estimates. Use real parameters, or a stored procedure.
- Make sure the **parameter types match the columns**: `@AssigneeId` the same type as `AssigneeId`, and `@Since` the same type as `CreatedAt` (for example `datetime2(3)` vs `datetime`). Your seek predicate looks clean, but check the plan XML for `CONVERT_IMPLICIT`.
- Check statistics freshness:
  ```sql
  SELECT s.name, sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
  FROM sys.stats s CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
  WHERE s.object_id = OBJECT_ID('dbo.Tickets');
  ```
  If `modification_counter` is large or the last update is old, run `UPDATE STATISTICS dbo.Tickets IX_Tickets_Assignee_Priority_Created WITH FULLSCAN;` (or a large sample) off-peak.
- As a **one-off diagnostic only**, run the query once with `OPTION (RECOMPILE)` for the busy agent. If the estimate then gets close to 40k, sniffing is confirmed. Don't leave RECOMPILE on a query that runs 300×/min; the ESR index makes the plan shape stable anyway.

### 3. Optional: a rewrite that bounds the worst case

Use this only if the plan from #1 still shows a lot of rows read for some agents. It applies when `Priority` has a small, fixed set of values (please confirm: type and distinct values). The rewrite seeks each priority separately with `AssigneeId = , Priority = , CreatedAt >= @Since`, which turns the range into a **real seek bound** (E, E, R, already in `CreatedAt DESC` order). It then merges at most 20 rows per priority and looks up `Subject` only for the final 20:

```sql
WITH top_keys AS (
    SELECT TOP (20) x.TicketId, x.Priority, x.CreatedAt
    FROM (VALUES (4), (3), (2), (1)) AS p(Priority)          -- or SELECT Priority FROM dbo.Priorities
    CROSS APPLY (
        SELECT TOP (20) t.TicketId, t.Priority, t.CreatedAt
        FROM dbo.Tickets AS t
        WHERE t.AssigneeId = @AssigneeId
          AND t.Priority   = p.Priority
          AND t.CreatedAt >= @Since
        ORDER BY t.CreatedAt DESC
    ) AS x
    ORDER BY x.Priority DESC, x.CreatedAt DESC
)
SELECT k.TicketId, t.Subject, k.Priority, k.CreatedAt
FROM top_keys AS k
JOIN dbo.Tickets AS t ON t.TicketId = k.TicketId              -- 20 clustered seeks for Subject
ORDER BY k.Priority DESC, k.CreatedAt DESC;
```
It uses the same index as #1, and in every case it reads at most `20 × (number of priorities)` index rows plus 20 lookups. The tie-break order (`Priority DESC, CreatedAt DESC`) is the same as the original query. If you need a deterministic tie-break for equal `CreatedAt`, add `TicketId DESC` to both ORDER BYs **and** to the index key after `CreatedAt DESC`.

### Not recommended
- Optimizer hints (`FORCESEEK`, `INDEX(...)`, `OPTIMIZE FOR`) as the fix. They freeze a plan; fix the index order instead.
- Adding a second index while keeping the old one. That costs extra writes on an 18M-row table with no benefit for this query.

## Verify

Test on staging with a copy of production data, or on production after a reviewed change window. Run it for **your busiest agent** and for a typical one:

```sql
SET STATISTICS IO, TIME ON;
-- In SSMS: Include Actual Execution Plan (Ctrl+M), or SET STATISTICS XML ON;
EXEC sp_executesql
  N'SELECT TOP (20) TicketId, Subject, Priority, CreatedAt
    FROM dbo.Tickets
    WHERE AssigneeId = @AssigneeId AND CreatedAt >= @Since
    ORDER BY Priority DESC, CreatedAt DESC;',
  N'@AssigneeId int, @Since datetime2(3)',        -- use your real column types
  @AssigneeId = <busy agent id>, @Since = <30 days ago>;
```
It is a SELECT, so running it changes no data. To see what the app actually gets, also pull the cached plan from Query Store / `sys.dm_exec_query_stats`, because the app's cached plan can differ from your SSMS session (different SET options such as `ARITHABORT` give a different cache entry).

**What a good plan looks like**
- `Top` → `Nested Loops` → `Index Seek` on **IX_Tickets_Assignee_Priority_Created** with Seek Predicate `AssigneeId = @AssigneeId`, a residual Predicate `CreatedAt >= @Since`, and **Ordered = True, Scan Direction FORWARD**
- **no Sort operator**, no spill warning, no (or a tiny) memory grant
- Index Seek **Number of Rows Read** in the hundreds, not 41k
- Key Lookup **executions ≈ 20**
- logical reads in the tens to low hundreds

Please send back the same summary as before so we can fill in this table:

| Metric | Before | After |
|---|---|---|
| Elapsed / CPU | 655 ms / 312 ms | ? |
| Logical reads (Tickets) | 131,904 | ? |
| Rows read by seek | 41,233 | ? |
| Key Lookup executions | 41,233 | ? (expect ~20) |
| Sort / tempdb spill | yes / yes | ? (expect no / no) |
| Index used | IX_Tickets_Assignee_Created | ? |

If the rows read are still high for some agents, the top priority is sparse in the window and #3 is the next step. If the plan still picks the old index or a Sort, send the `.sqlplan` XML and the stats query output.

## Assumptions / open questions

1. **`Priority` type and distinct values.** I assumed a small integer with a handful of values (e.g. 1–4). What is it, and how are the ~40k recent tickets of a busy agent spread across priorities? This decides whether #1 alone is enough or #3 is worth it.
2. **Edition.** `ONLINE = ON` / `RESUMABLE = ON` need Enterprise (or Developer). On Standard, drop those options and build in a quiet window, because the offline build blocks writes to Tickets.
3. **Write rate on Tickets.** About how many inserts per second, and how often `Priority` / `AssigneeId` / `CreatedAt` are updated? This confirms the write cost is acceptable (I expect it to be, since it replaces an index).
4. **Other queries using `IX_Tickets_Assignee_Created`.** Please check before dropping it.
5. **How the app sends the query** (sp_executesql parameters, stored procedure, or ORM) and the **parameter data types**. This is for the estimate gap in #2.
6. **Target latency.** What p95 does the helpdesk need? Today it is 655 ms; I would expect single-digit ms after the change, but the actual plan will confirm.
