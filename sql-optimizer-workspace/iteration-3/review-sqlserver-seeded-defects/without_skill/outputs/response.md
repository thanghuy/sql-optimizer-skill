# Review: "active buyers" search and the detail-page check (SQL Server 2019, compat 150)

Short version: both statements make SQL Server read far more of `Orders` (40M rows) than it needs to. The causes are non-sargable predicates, a join on mismatched data types, a `DISTINCT` that has to remove a join explosion, and no index on `Orders.UserId`. Below are the fixes in priority order, the rewritten SQL, the indexes, and how to check the result. I haven't seen your execution plans, so the assumptions I'm making are listed at the end. Please check them before you deploy.

---

## 1. What is wrong, in order of impact

| # | Problem | Why it hurts | Fix |
|---|---|---|---|
| 1 | **No index on `Orders.UserId`** | The detail-page check (`WHERE UserId = @id`) has to scan the 40M-row clustered index every time someone opens a page. The search can't use `UserId` either. | Add `IX_Orders_UserId_OrderDate (UserId, OrderDate)`. |
| 2 | **Join on `UserCode` with mismatched types**: `Users.UserCode` is `varchar(20)` and `Orders.UserCode` is `nvarchar(20)` | Data type precedence makes SQL Server add `CONVERT_IMPLICIT(nvarchar, u.UserCode)` on the Users side. With a SQL collation (for example `SQL_Latin1_General_CP1_CI_AS`) this blocks seeks on `IX_Users_UserCode`. In every case it skews cardinality estimates and adds CPU per row. `Users.UserCode` also has no unique constraint, so one order could match several users. | Join on `UserId`, which both tables already have and which is an int matching the PK. Separately, align the `UserCode` types. |
| 3 | **`YEAR(o.OrderDate) = 2025`** is non-sargable | A function on the column means `IX_Orders_OrderDate` can't be seeked. Every candidate order row is read and `YEAR()` is computed for each one. | Use a half-open range: `o.OrderDate >= '20250101' AND o.OrderDate < '20260101'`. |
| 4 | **`JOIN` + `DISTINCT`** | The join produces one row per order (a buyer with 500 orders in 2025 gives 500 rows). `DISTINCT` then has to sort or hash them all away. That costs CPU, memory grants and tempdb spills. You only want to know whether the user has at least one order. | Use `EXISTS`, a semi-join that stops at the first matching order per user. |
| 5 | **`ISNULL(u.IsDeleted, 0) = 0`** | Non-sargable, and it makes estimates worse. `IsDeleted` being nullable is the root cause. | Short term: `(u.IsDeleted = 0 OR u.IsDeleted IS NULL)`. Proper fix: backfill NULL to 0 and make the column `NOT NULL DEFAULT 0`, then use `u.IsDeleted = 0`. |
| 6 | **`IX_Users_UserName` doesn't cover the query** | After the `UserName` prefix seek, every match needs a key lookup to get `Email` and `IsDeleted`. A short prefix like `'a'` can mean hundreds of thousands of lookups. | Rebuild it with `INCLUDE (Email, IsDeleted)`. `UserId` is already there because it is the clustering key. |
| 7 | **Prefix search with an unbounded `@search`** | `LIKE @search + '%'` on `nvarchar` against an `nvarchar` column can do a dynamic range seek, so that part is fine. But `@search = ''`, `'a'`, or input containing `%`, `_` or `[` matches most of the table. Whichever plan gets compiled first is reused for everything (parameter sniffing). This explains the "CPU spikes when support uses it" pattern: one broad search, or a plan compiled for a broad search, hurts everyone. | Require a minimum length (for example 2 or 3 characters), escape wildcards, add `TOP`/paging, and consider `OPTION (RECOMPILE)` (see section 4). |
| 8 | **`IF (SELECT COUNT(*) ...) > 0`** | It counts every order the user has when you only need to know one exists. The optimizer can often turn `COUNT(*) > 0` into a semi-join, but you shouldn't rely on that. With no index on `UserId` it is a 40M-row scan either way. | Use `IF EXISTS (SELECT 1 ... )` plus the index from #1. |

---

## 2. Rewritten SQL

### 2a. Active buyers search

```sql
-- @search: nvarchar(100), already wildcard-escaped and length-checked by the app (see section 4)
SELECT  u.UserId, u.UserName, u.Email
FROM    dbo.Users AS u
WHERE   (u.IsDeleted = 0 OR u.IsDeleted IS NULL)      -- becomes u.IsDeleted = 0 once the column is NOT NULL
  AND   u.UserName LIKE @search + N'%'
  AND   EXISTS (
          SELECT 1
          FROM   dbo.Orders AS o
          WHERE  o.UserId    = u.UserId
            AND  o.OrderDate >= '20250101'
            AND  o.OrderDate <  '20260101'
        )
ORDER BY u.UserName                                   -- optional; add TOP/OFFSET paging for the UI
-- OPTION (RECOMPILE)                                 -- see section 4
;
```

Notes:
- `DISTINCT` is gone. Each user appears once because the query reads `Users` and only tests `Orders` for existence. `UserId` is the PK, so the output can't contain duplicates.
- `'20250101'` (no dashes) is unambiguous whatever the `DATEFORMAT`/language settings. Converting it to `datetime2` is a constant fold, not a per-row conversion. The half-open range `< '20260101'` correctly includes `2025-12-31 23:59:59.9999999`. A `BETWEEN ... '2025-12-31'` would not.
- If "2025" really means "the last 12 months" or "current year", pass the boundaries as `datetime2` parameters (`@from`, `@to`) instead of hard-coding them.

**Semantics check before switching the join from `UserCode` to `UserId`.** The original query matches orders by `UserCode`. The rewrite gives the same result only if `Orders.UserId` and `Orders.UserCode` always agree with `Users`. Run this once:

```sql
-- Orders whose UserId and UserCode point at different users (should return 0)
SELECT COUNT_BIG(*)
FROM   dbo.Orders o
JOIN   dbo.Users  u ON u.UserId = o.UserId
WHERE  u.UserCode <> o.UserCode;        -- run off-hours: this reads all 40M rows

-- Duplicate UserCodes in Users (should return nothing)
SELECT UserCode, COUNT(*) FROM dbo.Users GROUP BY UserCode HAVING COUNT(*) > 1;
```

If either query returns rows, the original query was already giving wrong results: it either attributed orders to the wrong user or matched several users. Decide which column is authoritative before you change anything.

### 2b. Detail-page check

```sql
IF EXISTS (SELECT 1 FROM dbo.Orders WHERE UserId = @id)
    SELECT ... -- load page
```

With the new index this is one index seek that stops at the first row: a few logical reads instead of a scan.

Also worth asking: do you need this round-trip at all? If the page query already returns nothing for a user with no orders, you can drop the check. Or you can fold it into the page query as `EXISTS`.

---

## 3. Indexes

```sql
-- 1) Serves the EXISTS in the search AND the detail-page check.
--    (UserId, OrderDate) lets the search seek straight to "this user, in 2025".
CREATE NONCLUSTERED INDEX IX_Orders_UserId_OrderDate
    ON dbo.Orders (UserId, OrderDate)
    WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, DATA_COMPRESSION = PAGE);  -- ONLINE needs Enterprise/Developer edition

-- 2) Cover the search on Users so there are no key lookups.
CREATE NONCLUSTERED INDEX IX_Users_UserName
    ON dbo.Users (UserName)
    INCLUDE (Email, IsDeleted)
    WITH (DROP_EXISTING = ON, ONLINE = ON);
```

- Building index 1 on 40M rows takes a while and generates a lot of log. Run it in a maintenance window, check log and tempdb space first, and drop `ONLINE = ON` if you're on Standard edition.
- `DATA_COMPRESSION = PAGE` is optional. It usually makes a narrow `(int, datetime2)` index much smaller, at a small CPU cost on writes.
- After the rewrite, check `sys.dm_db_index_usage_stats` over a full business cycle. If nothing else uses `IX_Orders_UserCode`, drop it: it's a 40M-row index that costs on every insert.
- Don't create a filtered index `WHERE IsDeleted = 0` until `IsDeleted` is `NOT NULL`. Filtered indexes don't support `OR`/`ISNULL`, so NULL rows would be left out and the optimizer couldn't match the predicate.

---

## 4. Application side (C#) and parameter sniffing

1. **Declare the parameter type and length explicitly.** Don't use `AddWithValue`, which sends `nvarchar(<length of the string>)`. That gives one cached plan per distinct length and invites plan-cache bloat and inconsistent plans.
   ```csharp
   cmd.Parameters.Add("@search", SqlDbType.NVarChar, 100).Value = escaped;
   cmd.Parameters.Add("@id", SqlDbType.Int).Value = id;
   ```
   `nvarchar` is the right type here because `UserName` is `nvarchar`. Keep it. (If you ever query `Users.UserCode`, which is `varchar`, send `SqlDbType.VarChar`. Sending nvarchar against a varchar column is the classic implicit-conversion scan.)
2. **Escape LIKE wildcards** so a user typing `%`, `_` or `[` doesn't turn a prefix search into a full scan:
   ```csharp
   var escaped = input.Replace("[", "[[]").Replace("%", "[%]").Replace("_", "[_]");
   ```
3. **Enforce a minimum search length** (2 or 3 characters) and **page the results** (`TOP (50)` or `OFFSET/FETCH` with `ORDER BY u.UserName`). A support screen never needs 200k rows. Unbounded result sets drive the memory grants and CPU spikes.
4. **Parameter sniffing.** Selectivity varies widely (`'a'` against `'johnsmith'`), so one cached plan rarely suits every input. Choose one of these:
   - `OPTION (RECOMPILE)`: compile cost per call is small for a support-tool query, and every execution gets a plan matched to its prefix. This is the simplest choice and usually the right one here.
   - Or keep the cached plan and pin a good shape with Query Store (`sp_query_store_force_plan`) once you've seen which plan is good for typical inputs.

---

## 5. Schema clean-up (do these after the quick wins)

- **Align `UserCode` types.** Make `Orders.UserCode` `varchar(20)` to match `Users` (or the other way round if codes can contain non-ASCII characters). On 40M rows `ALTER COLUMN` is a size-of-data, blocking operation, and you first have to drop and recreate `IX_Orders_UserCode`. Plan it for a maintenance window, or ask whether `Orders.UserCode` is needed at all once `UserId` is used. Duplicating the code in `Orders` is what made the mismatch possible.
- **Add a unique constraint on `Users.UserCode`** if codes are meant to be unique, and **a foreign key `Orders.UserId → Users.UserId`** if there isn't one. A trusted FK also helps the optimizer.
- **Make `IsDeleted` `NOT NULL DEFAULT 0`.** Backfill in batches (`UPDATE TOP (50000) ... WHERE IsDeleted IS NULL` in a loop) to keep log growth and blocking down, then `ALTER COLUMN IsDeleted bit NOT NULL`. After that the predicate becomes `u.IsDeleted = 0`.

---

## 6. How to verify

Run the old and new versions with representative inputs: a selective prefix (`'johns'`), a broad one (`'a'`), and an `@id` with orders and one without.

```sql
SET STATISTICS IO, TIME ON;
-- run old query, then new query, with actual execution plan enabled
```

What to look for:
- Logical reads on `Orders` should drop from millions (a scan of the clustered index or of `IX_Orders_UserCode` plus lookups) to a few per matching user (seeks on `IX_Orders_UserId_OrderDate`).
- The plan shows no `CONVERT_IMPLICIT` warning, no `Sort`/`Hash Match (Aggregate)` for the `DISTINCT`, and no key lookups on `Users`.
- The detail-page check becomes a single `Index Seek` with a few logical reads.
- Check Query Store before and after for CPU per execution and for the variation between executions. The variation is what shows up as spikes.

---

## 7. Assumptions to confirm (I can't check these without your environment)

1. `Orders.UserId` and `Orders.UserCode` always refer to the same user, and `Users.UserCode` is effectively unique. Run the check queries in 2a. If this doesn't hold, the join-key change also changes results.
2. "Active buyers" means "not deleted, with at least one order dated in calendar year 2025". If it should be a rolling window, parameterize the dates.
3. The database collation. It decides whether the `varchar`/`nvarchar` mismatch also blocks seeks on `IX_Users_UserCode`. The `UserId` rewrite avoids the question either way.
4. Edition (Enterprise/Developer vs Standard), which decides whether `ONLINE = ON` is available.
5. Whether anything else depends on `IX_Orders_UserCode` and `IX_Orders_OrderDate` before you drop or change them.

If you can share the actual execution plans (`.sqlplan`) for the current query with a broad and a narrow `@search`, I can confirm which of these issues dominates in your data.
