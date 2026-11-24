# Query anti-patterns: what to look for in the SQL itself

Use this in Step 5 whenever the user shows a query (or application code that issues queries). Walk the query clause by clause — SELECT list, FROM/JOIN, WHERE, GROUP BY/HAVING, ORDER BY, LIMIT/OFFSET — and check each item below. Most of these can be spotted from the SQL text alone; the schema (column types, nullability, existing indexes) confirms the rest, so say which findings depend on an assumption about the schema.

## Contents
1. How to report findings
2. Predicates that prevent index use (SARGability)
3. Wasted work and row explosion
4. Pagination and ordering
5. Correctness traps that also hurt the plan
6. Application-side patterns
7. Joins
8. Writes
9. Engine-specific patterns
9b. When the SQL text looks fine but the query is still slow (sniffing, bind types, collations, statistics, FKs)
10. Things that look bad but are fine (avoid false positives)

## 1. How to report findings

For each finding give: where it is (quote the fragment), what is wrong, severity, why it is slow (mechanism, not just "bad practice"), and the concrete fix (rewritten fragment or DDL).

Severity:
- **High** — cost grows with table size or row count: blocks index use on a large table, per-row subquery/round trip, full sort of a large set, wrong results.
- **Medium** — real but bounded cost: extra lookups, wider rows, avoidable sort of a small set, plan instability.
- **Low** — hygiene or future risk: style that becomes a problem later, readability that hides bugs.

Only report what you can justify. A query with no real problem should get "no significant issues" — inventing findings destroys the user's trust in the real ones. If a pattern is harmless here (small table, already indexed, engine handles it), either skip it or mention it as Low with the reason.

## 2. Predicates that prevent index use (SARGability)

A predicate is SARGable when the engine can turn it into an index seek/range. Anything that transforms the **column** (not the parameter) usually breaks that.

| Pattern | Example | Fix |
|---|---|---|
| Function on column | `WHERE DATE(created_at) = '2026-11-01'`, `YEAR(OrderDate) = 2025`, `TO_CHAR(order_date,'YYYY-MM-DD') = ?`, `TRUNC(d) = ?` | Half-open range: `created_at >= '2026-11-01' AND created_at < '2026-11-02'`; or an expression / function-based index on exactly that expression |
| Case-folding on column | `lower(email) = lower(?)`, `UPPER(name) LIKE ?` | Expression index on `lower(email)`, case-insensitive collation / `citext` (PG), `COLLATE NOCASE` (SQLite) |
| Arithmetic on column | `WHERE price * 1.1 > 100`, `created_at + INTERVAL 1 DAY > NOW()` | Move the math to the parameter side: `price > 100 / 1.1` |
| NULL-handling wrapper | `ISNULL(IsDeleted, 0) = 0`, `COALESCE(status,'x') = 'x'`, `NVL(col, 0) = 0` | `(IsDeleted = 0 OR IsDeleted IS NULL)`, or make the column NOT NULL with a default |
| Implicit type conversion | `phone = 0912345678` when `phone` is VARCHAR (MySQL casts the **column** to number → full scan; the leading zero is lost, so it can also return wrong rows); Oracle `varchar2_col = 12345` → `TO_NUMBER(col)`; SQL Server NVARCHAR parameter vs VARCHAR column → `CONVERT_IMPLICIT` | Pass the parameter in the column's type (`phone = '0912345678'`), fix the ORM/driver parameter type |
| Collation / charset mismatch | joining `utf8mb4_general_ci` to `utf8mb4_unicode_ci` columns | Align column collations |
| Leading wildcard | `LIKE '%@gmail.com'`, `LIKE '%abc%'` | Store a reversed / extracted column (e.g. `email_domain`) and index it; trigram (`pg_trgm`), full-text, or search index |
| OR across different columns | `email LIKE ? OR coupon_code = ?` | `UNION ALL` of two indexed queries (dedupe if needed), or rely on index merge / bitmap OR only after checking the plan |
| Negation | `status <> 'done'`, `NOT LIKE`, `NOT IN (list)` | Rewrite as positive set (`status IN ('new','paid')`) or partial index |
| Optional-parameter catch-all | `WHERE (@x IS NULL OR col = @x)` | Build the SQL dynamically per filter combination, or `OPTION (RECOMPILE)` (SQL Server) |

## 3. Wasted work and row explosion

| Pattern | Why it hurts | Fix |
|---|---|---|
| `SELECT *` | Reads every column (wide TEXT/JSON/LOB), prevents covering indexes, more network; in columnar engines bills every column | List the needed columns |
| `DISTINCT` (or `GROUP BY` on all columns) used to hide duplicate rows from a JOIN | Join produces N× rows, then sorts/hashes them all to remove duplicates | Use `EXISTS` / semi-join when the joined table is only a filter |
| Correlated scalar subquery in SELECT | `(SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id)` runs once per outer row; two such subqueries = two passes | One pre-aggregated join (`LEFT JOIN (SELECT customer_id, COUNT(*), MAX(created_at) ... GROUP BY customer_id)`) or `LATERAL` / `OUTER APPLY` with an index on the FK |
| `COUNT(*)` to test existence | `IF (SELECT COUNT(*) ...) > 0` counts all matches | `IF EXISTS (SELECT 1 ...)` / `SELECT 1 ... LIMIT 1` |
| `ORDER BY RAND()` / `NEWID()` / `random()` | Generates a value for every row and sorts the whole set to return a few | Random id range, `TABLESAMPLE`, pre-shuffled column, or pick in the app |
| `UNION` where `UNION ALL` is enough | Forces a dedup sort/hash | `UNION ALL` when branches cannot overlap |
| `HAVING` on a non-aggregate condition | Filters after grouping | Move to `WHERE` |
| Aggregating before filtering in subquery / CTE | Large intermediate result | Push filters into the inner query |
| Missing `LIMIT` on an endpoint that shows a page | Unbounded result | Add `LIMIT` and pagination |

## 4. Pagination and ordering

- **Large `OFFSET`** (`OFFSET 50000 LIMIT 20`) reads and discards 50,000 rows every time; cost grows with page number. Fix: keyset pagination `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC`, or a deferred join (page over ids in a covering index, then join for the 20 rows).
- **ORDER BY without a unique tie-breaker** gives unstable pages and duplicate/missing rows between pages. Add the PK as the last ORDER BY column — and put it right after the sort columns in the index (`esr-rule.md` §5.6).
- **ORDER BY on an expression or on columns from two tables** cannot be served by one index → sort.

## 5. Correctness traps that also hurt the plan

- **`NOT IN (subquery)` where the subquery column is nullable**: a single NULL makes the predicate unknown for every row, so the query returns **no rows**; planners also cannot turn it into an efficient anti-join. Use `NOT EXISTS`.
- **`BETWEEN` on timestamps with date literals**: `BETWEEN '2026-11-01' AND '2026-11-30'` misses most of Nov 30. Use half-open ranges.
- **Implicit cast that changes meaning** (number vs zero-padded string, as above).
- **Accidental cross join** (comma join with a missing condition) — row count explodes.
- **Non-deterministic `LIMIT` without `ORDER BY`**.

## 6. Application-side patterns

When the user shows code, not only SQL:
- **N+1 queries**: one query for the list, then one per row in a loop (`for o in orders: query(... WHERE order_id = o.id)`). Cost = N round trips + N parses. Fix: one `JOIN`, or `WHERE order_id IN (...)` / `= ANY(?)` batch, or the ORM's eager loading (`select_related`, `prefetch_related`, `JOIN FETCH`, `Include`).
- **String-concatenated literals instead of bind parameters**: a new hard parse per value (Oracle shared pool / SQL Server plan cache pollution, PG no plan reuse), and SQL injection. Use bind parameters; mention injection explicitly.
- **Row-by-row INSERT/UPDATE** in a loop → batch / multi-row insert / `COPY` / bulk APIs, inside one transaction.
- **Fetching everything to filter / paginate / count in the app** → push it into SQL.
- **Long transactions** around user think-time or HTTP calls → lock waits, bloat, undo growth.

## 7. Joins

- Join columns with different types or collations → conversion on one side, no index seek (e.g. `Orders.UserCode nvarchar` = `Users.UserCode varchar` in SQL Server converts the VARCHAR side).
- Joining on an expression (`ON a.id = CAST(b.ref AS INT)`, `ON DATE(a.ts) = b.day`).
- Missing index on the inner side's join column (usually the FK) for nested-loop joins.
- Filtering a LEFT JOINed table in WHERE (turns it into an inner join silently) — correctness first, then plan.

## 8. Writes

- Huge `UPDATE`/`DELETE` in one statement → lock escalation, replication lag, undo/WAL spikes. Chunk by PK range.
- Non-SARGable `WHERE` on UPDATE/DELETE → scans and locks many rows.
- Updating indexed columns that did not change (`SET col = col`) → index maintenance, no HOT (PG).

## 9. Engine-specific patterns

- **MySQL/MariaDB**: implicit string↔number cast on columns; `IN (subquery)` on very old versions (< 5.6); `ORDER BY` with mixed directions before 8.0 / MariaDB 10.8; `SQL_CALC_FOUND_ROWS` (deprecated, slow).
- **PostgreSQL**: CTEs are optimization fences before 12 (use `NOT MATERIALIZED` or inline); `lower()` without expression index; `NOT IN` with NULLs; generic plans for prepared statements; `OFFSET` pagination.
- **SQL Server**: `CONVERT_IMPLICIT` from NVARCHAR parameters; `ISNULL`/functions on columns; scalar UDFs in SELECT/WHERE (row-by-row before 2019 inlining); multi-statement TVFs and table variables (bad estimates on older compat levels); catch-all `(@p IS NULL OR col = @p)`; parameter sniffing.
- **Oracle**: missing bind variables; implicit `TO_NUMBER`/`TO_CHAR` on columns; `TRUNC(date_col)`; `ROWNUM` applied before `ORDER BY` in the same query block (wrong results).
- **SQLite**: type affinity mismatches; `LIKE` case rules; missing `ANALYZE`.
- **Db2**: static SQL not rebound after index changes; `REOPT` for skewed parameters.
- **Snowflake**: functions on clustering columns defeat pruning; `SELECT *`; exploding joins; result cache hiding real cost in benchmarks.
- **BigQuery**: `SELECT *` and `LIMIT` do not reduce bytes billed; filter not on the partition column; `CURRENT_DATE()`/non-deterministic functions disable the result cache; `ORDER BY` without `LIMIT` on huge results; cross joins.
- **ClickHouse**: filters not on the sort-key prefix; `FINAL`; large right-hand table in `JOIN` (put the small table on the right, or use dictionaries / `IN`); `SELECT *` on wide tables.

## 9b. When the SQL text looks fine but the query is still slow

Some of the worst problems are invisible in the SQL text. Look for them when the user reports "fast in the console, slow in the app", "sometimes slow", "slow after a data load", or when a plan shows a good index existing but not used, or estimates far from actuals.

- **Parameter sniffing / generic plans**: the plan was compiled for one parameter value and reused for a very different one (skewed column: `status = 'Closed'` 95% vs `'Open'` 0.1%). Evidence: estimated rows match the compile-time value, not the runtime one (`ParameterCompiledValue` vs `ParameterRuntimeValue` in SQL Server plan XML; `+PEEKED_BINDS` in Oracle; generic plan in PG after 5 executions). Fixes by version: SQL Server `OPTION (RECOMPILE)` for infrequent queries, `OPTIMIZE FOR`, separate code paths per value class, Query Store plan forcing; PSP optimization only on 2022 / compat 160. Oracle: histograms + adaptive cursor sharing, SQL plan baselines. PG: `plan_cache_mode = force_custom_plan` for that role/statement.
- **"Fast in SSMS, slow in the app" (SQL Server)**: SSMS sets `ARITHABORT ON`, most drivers leave it `OFF`, so they get **separate plan cache entries** compiled with different sniffed values. Testing in SSMS with local variables (`DECLARE @x = ...`) also changes estimates (density, not sniffing). Reproduce with `sp_executesql` and the app's SET options (compare `sys.dm_exec_plan_attributes`), never by toggling ARITHABORT as the "fix".
- **Driver bind type mismatch**: the parameter type differs from the column type, so the engine converts the **column**:
  - Oracle: a Python/Java `datetime` bound as `TIMESTAMP` against a `DATE` column → `INTERNAL_FUNCTION("COL") >= :B1` in the predicate section, index not usable. Fix: bind as DATE (`cursor.setinputsizes(oracledb.DB_TYPE_DATE)` / `setObject(..., Types.DATE)`), or `CAST(:b AS DATE)` on the parameter side.
  - SQL Server: .NET `AddWithValue` sends strings as `nvarchar(n)` → `CONVERT_IMPLICIT` on a `varchar` column (seek may degrade to scan depending on collation). Fix: explicit `SqlDbType.VarChar` with size.
  - PostgreSQL/JDBC: `setString` for a non-text column or `numeric` vs `float8` parameters → casts on the column side.
- **Charset / collation mismatch in joins (MySQL/MariaDB)**: joining `utf8mb3` to `utf8mb4` columns, or columns with different collations, converts one side per row → `type: ALL` on a table that has an index on the join column. Fix: `ALTER TABLE ... MODIFY col ... CHARACTER SET utf8mb4 COLLATE <same>` on the mismatched column (online-DDL caveats), not a new index.
- **Bad statistics, not a missing index**: estimates off by 10×+ — stale stats after bulk loads (`last_analyzed`, `last_autoanalyze`, `modification_counter`), **correlated columns** (`city = 'Hanoi' AND country = 'VN'` estimated as independent → rows=1 instead of 480k → nested loop disaster; fix PG `CREATE STATISTICS ... (dependencies, mcv)`, Oracle column-group extended stats, SQL Server multi-column stats), skew without histogram, or ascending keys beyond the histogram. Fix the statistics first; adding an index rarely fixes a misestimate.
- **Unindexed foreign keys**: deletes/updates on the parent table are slow or block, because each one scans the child table (Oracle also locks the whole child table). See `esr-rule.md` §7 before dropping such an index.

## 10. Things that look bad but are fine (avoid false positives)

Do not flag these as problems unless the plan shows otherwise:
- `LIKE 'abc%'` (prefix) — index-usable. In SQL Server, `LIKE @p + '%'` can also seek (dynamic range), as long as the parameter does not start with a wildcard.
- Row-value comparison `(created_at, id) < ($1, $2)` in PostgreSQL — index-usable with a matching composite index; it is the recommended keyset form there.
- `COUNT(*)` vs `COUNT(1)` — identical in every major engine.
- `SELECT *` inside `EXISTS (SELECT * ...)` — the column list is ignored.
- Short `IN (...)` lists on an indexed column — fine for filtering (only the sort-order caveat in `esr-rule.md` applies).
- CTEs in PostgreSQL 12+ that are referenced once — inlined automatically.
- `ORDER BY ... LIMIT` that matches an existing index — no sort happens.
- A full scan on a tiny table (a few thousand rows) — often the cheapest plan.
- Arithmetic on the **parameter** side: `created_at > NOW() - INTERVAL 7 DAY`, `order_date >= SYSDATE - 1` — SARGable; only arithmetic on the column is a problem.

When the query is already good, say so, show briefly why (e.g. "index scan on (author_id, created_at DESC, id DESC) feeds the LIMIT, no sort"), and stop. Optional polish can be listed as Low, clearly marked optional.
