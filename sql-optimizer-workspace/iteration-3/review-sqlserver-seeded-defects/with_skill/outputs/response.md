I'm working from **SQL Server 2019, compatibility level 150**, as you said. All syntax below is valid there. Your edition isn't stated, so options that need Enterprise (or Developer, or Azure SQL) are marked. I checked the SQL Server 2019 `CREATE INDEX` docs for the online, resumable and filtered-index rules used below.

## Diagnosis

The query text alone explains the CPU spikes. Plans and row counts will confirm the details, but the main causes are already visible:

1. **There is no usable access path into Orders for this query.** The filter on Orders is `YEAR(o.OrderDate) = 2025`, which can't use an index seek. The join column `UserCode` is indexed alone, without `OrderDate`. So SQL Server has two choices:
   - scan the 40M-row Orders table (or its `OrderDate` index plus lookups) and hash-join it, or
   - seek `IX_Orders_UserCode` for each matching user, then do a **Key Lookup for every one of that user's orders** just to read `OrderDate`.

   A short search prefix like `'a'` or `'Ng'` from support can match tens of thousands of users. At about 20 orders per user (40M / 2M), that is hundreds of thousands to millions of lookups, or a full scan with a parallel plan. Either way, CPU spikes.
2. **The join columns have different types.** `Users.UserCode` is `varchar(20)` and `Orders.UserCode` is `nvarchar(20)`. Because `nvarchar` has higher type precedence, SQL Server converts the **Users** side (`CONVERT_IMPLICIT`). That blocks or degrades seeks on `IX_Users_UserCode` and skews the row estimates. Orders already has an `int UserId` column, so the join should not use `UserCode` at all.
3. **`DISTINCT` hides the join fan-out.** Every 2025 order of a user produces a row, and then all of them get sorted or hashed back down to one row per user. You only need to know whether the user has *any* 2025 order, which is a semi-join (`EXISTS`) that stops at the first match.
4. **The detail-page check scans the whole table.** `IF (SELECT COUNT(*) FROM dbo.Orders WHERE UserId = @id) > 0` has **no index on `Orders.UserId`**. Each page open can therefore scan the 40M-row clustered index. This probably accounts for much of the background CPU, not only the search.

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `FROM dbo.Orders WHERE UserId = @id` (detail page) | No index on `Orders.UserId` | **High** | Every page open scans the 40M-row clustered index to answer a yes/no question | Index on `Orders(UserId, OrderDate)` (rec. 1). It also serves the rewritten search. |
| 2 | `IF (SELECT COUNT(*) ...) > 0` | `COUNT(*)` used to test whether a row exists | **High** (together with #1) / Medium (alone) | Asks for every matching row to be counted. The optimizer can sometimes turn this pattern into an existence check, but you shouldn't rely on that. | `IF EXISTS (SELECT 1 FROM dbo.Orders WHERE UserId = @id)` |
| 3 | `YEAR(o.OrderDate) = 2025` | Function on the column (non-SARGable) | **High** | The `OrderDate` index can't be seeked. `YEAR()` is evaluated on every row read. | Half-open range: `o.OrderDate >= '20250101' AND o.OrderDate < '20260101'` |
| 4 | `ON o.UserCode = u.UserCode` | `nvarchar` vs `varchar` join, so `CONVERT_IMPLICIT` on `Users.UserCode` | **High** | No clean seek on the converted side, poor estimates, and a hash join or scan becomes likely. Look for the yellow implicit-conversion warning in the plan. | Join on `o.UserId = u.UserId` (int = int, and `UserId` is the clustered PK of Users). If you must keep `UserCode`, align the column types (see Assumptions). |
| 5 | `SELECT DISTINCT ... JOIN dbo.Orders` | `DISTINCT` used to remove join duplicates | **High** | It builds one row per order (about 20× more rows than users), then sorts or hashes them. That is a large memory grant and possible tempdb spills. | `EXISTS` semi-join. `u.UserId` is the PK, so the result needs no `DISTINCT`. |
| 6 | `ISNULL(u.IsDeleted, 0) = 0` | NULL-handling wrapper on the column | Medium | It can't be seeked and is always a residual predicate. A filtered index can't express it either, because filtered-index predicates don't allow `OR`. | Now: `(u.IsDeleted = 0 OR u.IsDeleted IS NULL)`. Properly: backfill NULLs, make the column `NOT NULL DEFAULT 0`, then use `u.IsDeleted = 0`. |
| 7 | Whole search query | No `TOP` / no `ORDER BY` on an interactive search | Medium | A one-letter or empty `@search` returns a huge set (an empty string matches all 2M users) to a support UI | `SELECT TOP (50) ... ORDER BY u.UserName, u.UserId`. The `UserName` index already delivers this order, so the query can stop after 50 rows. Also enforce a minimum search length in the app. |
| 8 | `u.UserName LIKE @search + '%'` | Plan depends on the sniffed `@search` value, plus user-typed wildcards | Low–Medium | The prefix `LIKE` itself is fine: `nvarchar` parameter vs `nvarchar` column, so there is no conversion and SQL Server can do a dynamic range seek. But a plan compiled for `'Nguyen'` is reused for `'a'`. If a user types `%` or `_`, the seek becomes a scan. | Escape `%`, `_` and `[` in the app, and pair with `TOP` (#7). In C#, declare the parameter as `SqlDbType.NVarChar, Size = 100`, not `AddWithValue`, which sizes it per string and creates one cached plan per length. |

Not a problem: `@search` being `nvarchar` against `UserName nvarchar(100)` is a type match, so it causes no implicit conversion. That conversion trap would only apply if you searched `UserCode` (a `varchar` column) with an `nvarchar` parameter.

## Rewritten query

```sql
-- Search (support "active buyers")
DECLARE @from datetime2 = '20250101', @to datetime2 = '20260101';  -- or pass as parameters

SELECT TOP (50)                       -- optional but recommended, see #7
       u.UserId, u.UserName, u.Email
FROM dbo.Users AS u
WHERE u.UserName LIKE @search + N'%'
  AND (u.IsDeleted = 0 OR u.IsDeleted IS NULL)   -- becomes "u.IsDeleted = 0" after the NOT NULL fix
  AND EXISTS (SELECT 1
              FROM dbo.Orders AS o
              WHERE o.UserId    = u.UserId
                AND o.OrderDate >= @from
                AND o.OrderDate <  @to)
ORDER BY u.UserName, u.UserId;        -- UserId as a unique tie-breaker for stable paging
```

```sql
-- Detail page
IF EXISTS (SELECT 1 FROM dbo.Orders WHERE UserId = @id)
    SELECT ... -- load page
```

If the year comes from the app as a number, compute the bounds on the parameter side: `@from = DATEFROMPARTS(@year,1,1)` and `@to = DATEFROMPARTS(@year+1,1,1)`. Never wrap the column.

Semantics: the result is the same set of users as your original query, assuming `Orders.UserId` and `Orders.UserCode` always point to the same user (please confirm, see open questions). Adding `TOP` changes the output only when more than 50 users match.

## Recommendation

**1. New index on Orders. This is the main fix and serves both statements.**

```sql
CREATE NONCLUSTERED INDEX IX_Orders_UserId_OrderDate
ON dbo.Orders (UserId, OrderDate)
WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 60 MINUTES, SORT_IN_TEMPDB = ON);
-- ONLINE/RESUMABLE: Enterprise, Developer, Azure SQL. RESUMABLE for CREATE INDEX is 2019+ and needs ONLINE = ON.
-- On Standard edition: drop those two options and build in a low-traffic window (the build blocks writes to Orders).
```

- ESR mapping: **E** = `UserId` (the join and correlation column behaves as an equality for each probe; it is also the detail-page equality). **R** = `OrderDate` (2025 range). There is no S here. `OrderId` (the clustering key) is carried in the index automatically.
- Expected effect:
  - Each `EXISTS` probe becomes one seek to `(UserId, 2025-01-01)` that stops at the first row. There are no key lookups because the index covers the subquery.
  - The detail-page check becomes a single-row seek instead of a 40M-row scan.
- Cost:
  - Size is about 40M × ~25 bytes (int + datetime2 + bigint locator + row overhead), roughly **1–1.5 GB**. Plan for about the same again in tempdb or data files during the build.
  - One extra index write on every Orders insert. Updates to `UserId` or `OrderDate` should be rare.
- Rollback: `DROP INDEX IX_Orders_UserId_OrderDate ON dbo.Orders;` To test first, `ALTER INDEX ... DISABLE` keeps the definition.
- Existing index afterwards: once nothing joins on `UserCode` any more, `IX_Orders_UserCode` may be dead weight on a 40M-row table. **Don't drop it blindly.** Check `user_seeks`/`user_scans` in `sys.dm_db_index_usage_stats` (query below) over a full business cycle first. `IX_Orders_OrderDate` probably serves reports, so keep it unless usage stats say otherwise.

**2. Extend the existing UserName index to cover the search instead of adding a new one.**

```sql
CREATE NONCLUSTERED INDEX IX_Users_UserName
ON dbo.Users (UserName)
INCLUDE (Email, IsDeleted)
WITH (DROP_EXISTING = ON, ONLINE = ON);   -- ONLINE: Enterprise/Developer/Azure; otherwise off-peak
```

- ESR mapping: **R** = `UserName` (prefix `LIKE`). The same column is also **S** for `ORDER BY u.UserName, u.UserId`. `UserId` is the clustering key, so it already sits right after `UserName` in the index and gives the tie-breaker order for free. `IsDeleted` and `Email` are INCLUDE columns, so they don't change the ordering.
- Expected effect: a range seek on the prefix, the `IsDeleted` check inside the index, no Key Lookup back to Users, no Sort operator, and `TOP (50)` stops early.
- Cost: `Email nvarchar(200)` makes the index noticeably wider (2M rows × the average email length × 2 bytes, probably a few hundred MB at most). Users is not a high-write table, so this is usually acceptable. With `TOP (50)`, the alternative is to include only `IsDeleted`, which costs at most 50 lookups for `Email`. I'd include `Email` only if this search is hot.
- Rollback: run the same statement with the original definition, `(UserName)` with no INCLUDE, and `DROP_EXISTING = ON`.

**3. Fix the data model (planned change, not urgent).**

- **`IsDeleted` NOT NULL.** Backfill in batches (`UPDATE TOP (50000) dbo.Users SET IsDeleted = 0 WHERE IsDeleted IS NULL;` in a loop), then make the column `NOT NULL` with a default. That turns #6 into a plain `IsDeleted = 0`. If most users are deleted, it also allows a filtered index `WHERE IsDeleted = 0`. Index #2 includes `IsDeleted`, so you'll need to drop it or remove the column from INCLUDE before you can alter the column type.
- **`UserCode` types.** Make both columns the same type. Prefer changing `Users.UserCode` to `nvarchar(20)`: it is the smaller table (2M rows vs 40M), and widening `varchar` to `nvarchar` can't lose data. `IX_Users_UserCode` must be dropped and recreated around the `ALTER`. This matters if any other query still joins on `UserCode`.

## Verify

Run on staging with production-like data, before and after, with a short prefix (`N'a'`) and a long one (`N'Nguyen Van'`):

```sql
SET STATISTICS IO, TIME ON;
SET STATISTICS XML ON;      -- or SSMS "Include Actual Execution Plan" (Ctrl+M)
-- run the rewritten search with your real @search values
-- run the detail-page check with a real @id
SET STATISTICS XML OFF;
SET STATISTICS IO, TIME OFF;
```

These are read-only SELECTs, so running them changes no data. Save the plans as `.sqlplan` and send them over.

A good plan looks like this:
- **Search:** Index Seek on `IX_Users_UserName` with a Seek Predicate on the `UserName` range. Nested Loops (Left Semi Join) into an Index Seek on `IX_Orders_UserId_OrderDate`, where both `UserId` and `OrderDate` appear as **Seek Predicates**. No Key Lookup, no Sort, no Hash Match, no `CONVERT_IMPLICIT` warning. Logical reads on Orders should fall from millions to roughly 3 × (users probed).
- **Detail page:** an Index Seek on `IX_Orders_UserId_OrderDate` with a handful of logical reads, instead of a Clustered Index Scan.
- If estimated and actual row counts differ by 10× or more on the Users seek, the problem is a sniffed short or long prefix, not the index. Tell me and we'll look at it.

To get before and after numbers from production, and to check whether the old index is still used:

```sql
-- Index usage on Orders since the last restart
SELECT i.name, s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE s.database_id = DB_ID() AND s.object_id = OBJECT_ID('dbo.Orders');
```

Query Store (if enabled) shows CPU and duration per execution for both statements over time.

## Assumptions / open questions

Please answer these when you can. I continued with the assumptions stated next to each one.

1. **Is `Orders.UserId` always consistent with `Orders.UserCode`?** In other words, does `Orders.UserCode` always equal the `UserCode` of the user with that `UserId`? I assumed yes, which is what makes the switch to the `UserId` join safe. Check with:
   ```sql
   SELECT COUNT(*) FROM dbo.Orders o JOIN dbo.Users u ON u.UserId = o.UserId
   WHERE o.UserCode <> CAST(u.UserCode AS nvarchar(20));   -- expect 0
   ```
   If the result isn't 0, the two joins return different users, and we need to decide which one is correct.
2. **Edition:** Enterprise or Standard? This decides whether the `ONLINE`/`RESUMABLE` builds are available. `SELECT SERVERPROPERTY('Edition');`
3. **How many users does a typical support search prefix match, and what share of users ordered in 2025?** I assumed short prefixes match many users and most users are not deleted. If 2025 buyers are a small minority, the optimizer may drive from Orders instead. The new index still supports that path through a seek on the `OrderDate` range.
4. **Traffic:** how often do the search and the detail-page check run, and what is the insert rate on Orders? I assumed the detail check runs often, which is why its index is the first recommendation. On a very insert-heavy Orders table, weigh the extra index write against the scan it removes. A 40M-row scan per page view is almost certainly the bigger cost.
5. **Can the search return only the top 50 rows sorted by name?** If the screen really needs every match, drop the `TOP`. The rewrite and both indexes still apply.
