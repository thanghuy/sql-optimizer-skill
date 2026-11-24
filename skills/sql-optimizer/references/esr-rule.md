# The ESR rule for composite indexes

## Contents
1. Why column order matters
2. The rule
3. Worked example
4. Classifying predicates
5. Exceptions and trade-offs (selective range, several ranges, mixed directions, pagination tie-breakers, partial indexes)
6. Covering indexes
7. Designing for many queries
8. Things that silently break an index
9. ESR on engines without B-trees (Snowflake, BigQuery, ClickHouse)
10. Checklist

## 1. Why column order matters

A B-tree composite index on `(a, b, c)` is a sorted list: sorted by `a`, then by `b` inside each `a`, then by `c` inside each `(a, b)`. The engine can jump (seek) to a position and read a contiguous slice. A slice stays contiguous **and** stays in sorted order only as long as every column before it is fixed to a single value. That one fact produces the whole rule:

- Equality fixes a column to one value → the next column is still sorted → you can keep going.
- A range on a column spans many values → the columns after it are no longer globally sorted, and can only be used as a filter on the slice, not for seeking or ordering.

## 2. The rule

Order the columns of a composite index as:

1. **E — Equality**: columns compared with `=` (or `IS NULL`, or `IN` with exactly one value). Order among equality columns does not matter for this query; choose it for reuse by other queries (section 7).
2. **S — Sort**: columns of `ORDER BY` (and `GROUP BY` / `DISTINCT`, which are sort-like), in the same order, with matching ASC/DESC directions (or all reversed).
3. **R — Range**: columns with `>`, `>=`, `<`, `<=`, `BETWEEN`, `LIKE 'prefix%'`, multi-value `IN`, `<>`, `IS NOT NULL`. Only the first range column is used to bound the seek; later range columns are filters.

Result: the engine seeks on E, reads rows already in S order (no sort step, and `LIMIT` can stop early), and filters on R while reading.

## 3. Worked example

```sql
SELECT id, total, created_at
FROM orders
WHERE customer_id = ?          -- E
  AND status = 'paid'          -- E
  AND created_at >= ?          -- R
ORDER BY total DESC            -- S
LIMIT 20;
```

ESR index: `(customer_id, status, total DESC, created_at)`
- Seek to `customer_id=?, status='paid'`, rows already ordered by `total DESC`, check `created_at` per row, stop after 20 matches. No sort.

Common wrong index: `(customer_id, status, created_at, total)` (E-R-S)
- Seek is tighter (only matching dates are read), but rows come out ordered by `created_at`, so the engine must read **all** matches and sort them before returning 20.

Which is better depends on data — see 5.1.

## 4. Classifying predicates

| Predicate | Role | Note |
|---|---|---|
| `col = ?`, `col IS NULL` | E | |
| `col IN (1)` | E | single value |
| `col IN (1,2,3)` | E for seek, R for sort | Seek works (multiple ranges), but output is not in order of the next column across the list values. Some engines merge the lists (Oracle INLIST iterator, PG 17+ improved `= ANY` handling) — check the plan. |
| `col > ?`, `BETWEEN`, `col <> ?` | R | |
| `col LIKE 'abc%'` | R | prefix only; `'%abc'` cannot seek at all |
| `ORDER BY a, b` | S | directions must match or be fully reversed |
| `GROUP BY a, b` | S | lets the engine stream-aggregate without hashing/sorting |
| `JOIN ... ON t.fk = o.id` | E (on the inner table) | for nested-loop joins the join column on the looked-up table behaves as equality |

## 5. Exceptions and trade-offs

### 5.1 Very selective range → E-R-S can win
ESR optimizes for avoiding a sort. If the range is very selective (e.g. `created_at` in the last hour matches 50 rows out of 5 million for this customer), reading 50 rows and sorting them is cheaper than walking the ESR index ordered by `total` and discarding millions of non-matching rows. Decide with numbers from Step 3:
- rows matching E only: N_e
- rows matching E and R: N_er
- with `LIMIT k` and ESR you read about `k × N_e / N_er` rows; with E-R-S you read `N_er` rows and sort them.
Pick the smaller. If both patterns run often, two indexes may be justified — check write traffic first.

### 5.2 No ORDER BY → it is just E then R
Without a sort, put equality first, then the most selective range column.

### 5.3 Several range predicates
Only one range can bound the seek. Put the most selective range first among R. Others are filters — consider including them so the filter is applied in the index without a table lookup. Skip scan (Oracle, MySQL 8.0.13+, PostgreSQL 18+, SQLite with stats, Db2 jump scan) can sometimes use later columns after a low-cardinality leading column, but do not design for it; verify in the plan.

### 5.4 Mixed sort directions
`ORDER BY a ASC, b DESC` needs an index declared `(a ASC, b DESC)` (or `(a DESC, b ASC)` read backwards). Supported by PostgreSQL, SQL Server, Oracle, SQLite, Db2, MySQL 8.0+ (older MySQL ignored DESC), MariaDB 10.8+.

### 5.5 Equality on low-cardinality columns
Still put them in E. The old advice "most selective column first" is mostly obsolete for equality columns in a composite index: once all equality columns are fixed the slice is the same regardless of order. Order them for prefix reuse instead.

### 5.6 Pagination tie-breakers belong right after the sort columns
Keyset pagination (`WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC`) needs a unique tie-breaker, and that tie-breaker is part of S: it must come **immediately after** the other sort columns in the index. `(shop_id, created_at, status)` does not deliver `created_at, id` order even though InnoDB appends `id` at the end, because `status` sits in between. Write it as `(shop_id, created_at, id, ...)`, and put filter-only columns after it. Whenever you rewrite a query to add a tie-breaker, re-check that the index you proposed still matches the new ORDER BY.

### 5.7 Partial / filtered indexes
If an equality is on a constant that almost every query uses (`deleted_at IS NULL`, `status = 'active'`), a partial index (PostgreSQL, SQLite) or filtered index (SQL Server) can remove that column from the key and shrink the index a lot.

## 6. Covering indexes

If the index also contains every column the query reads, the engine never touches the table (Index Only Scan, "Using index", no Key Lookup / TABLE ACCESS BY ROWID). Add those columns:
- as `INCLUDE (...)` where supported (PostgreSQL 11+, SQL Server, Db2 unique indexes): they are stored only in leaf pages and do not affect ordering;
- otherwise as trailing key columns after R (MySQL, MariaDB, Oracle, SQLite).
In InnoDB (MySQL/MariaDB) the primary key is implicitly appended to every secondary index, so do not add it.

Covering is worth it for hot queries that return few columns. It is not worth it for `SELECT *` or wide text columns — fix the query first. Decide it after fixing the order: once the index is in ESR order and the query has a small `LIMIT`/`TOP`, only that many rows need a lookup, so covering a wide column (e.g. `nvarchar(400)`) may cost more in index size and write amplification than it saves. State the lookup count either way, so the user sees why you chose.

## 7. Designing for many queries (leftmost prefix)

An index `(a, b, c)` also serves queries on `(a)` and `(a, b)` — its leftmost prefixes — but not `(b)` or `(b, c)` (except via skip scan). So:
- Collect all hot queries on the table (from pg_stat_statements / performance_schema / Query Store / AWR).
- Choose the equality column order so that one index's prefixes serve several queries.
- An index that is a leftmost prefix of another index is usually redundant: `(a)` is covered by `(a, b)`. Exceptions: uniqueness constraints, FK requirements, or a much smaller index for a very hot query.
- Before calling any index redundant or unused and suggesting `DROP`, check what else it does:
  - **Backs a constraint**: PRIMARY KEY / UNIQUE indexes enforce correctness; low `idx_scan` does not matter.
  - **Backs a foreign key**: an index on the child's FK column(s) is what makes `DELETE`/`UPDATE` on the parent (and `ON DELETE CASCADE`) fast; without it every parent delete scans the child table and, in Oracle, takes a table-level lock on the child. Usage counters often show 0 for these because FK checks may not be counted as user scans. MySQL/InnoDB requires an index on FK columns and will refuse to drop it.
  - **Usage stats are reliable**: when were stats last reset (`pg_stat_database.stats_reset`, server restart for `sys.dm_db_index_usage_stats`)? Do they include monthly/yearly jobs? Are reads served by **replicas**, whose usage counters are separate from the primary's?
  - Prefer the reversible path: `ALTER INDEX ... INVISIBLE` (MySQL 8, Oracle), `IGNORED` (MariaDB 10.6+), `DISABLE` (SQL Server nonclustered) before `DROP`.
- Budget: OLTP tables with heavy writes typically should not have more than ~5–7 secondary indexes. Every index costs on every INSERT/DELETE and on UPDATE of its columns.

## 8. Things that silently break an index

- Function or arithmetic on the column: `WHERE DATE(created_at) = ?`, `WHERE amount * 1.1 > ?`, `WHERE LOWER(email) = ?` → rewrite as range (`created_at >= ? AND created_at < ?`) or create an expression/function-based index.
- Implicit type conversion: comparing a `VARCHAR` column to a number, or joining `INT` to `BIGINT`/`VARCHAR`, `NVARCHAR` parameter vs `VARCHAR` column in SQL Server → cast the parameter, not the column.
- Collation mismatch between joined columns (MySQL/MariaDB, SQL Server).
- Leading wildcard `LIKE '%x'` → full-text / trigram (PG `pg_trgm`) index.
- `OR` across different columns → `UNION ALL` of two indexed queries, or rely on index merge / bitmap OR (check the plan).
- `col <> ?` / `NOT IN` → usually not seekable.
- Parameter sniffing / generic plans: the index is used for one value and not another (SQL Server, Oracle bind peeking, PG generic plans after 5 executions).
- Stale statistics: the right index exists but estimates say a scan is cheaper.

## 9. ESR on engines without B-trees

Snowflake, BigQuery and ClickHouse prune storage blocks by min/max metadata rather than seeking a B-tree. The same intuition applies to the **physical sort / clustering order**:

- **ClickHouse** `ORDER BY` (primary key): put the columns that are filtered by equality most often first, generally low-cardinality before high-cardinality so the sparse index prunes granules well; then the range column (often time). A sort key that matches the query's ORDER BY allows `optimize_read_in_order`. Use projections or skip indexes for secondary access patterns.
- **BigQuery**: partition by the range column you filter by almost always (usually date/timestamp), then cluster (up to 4 columns) by the columns used in equality filters, in order of how often they are filtered. Clustering order matters: filters on the first clustering column prune best.
- **Snowflake**: clustering keys only pay off on large tables (multi-TB / many micro-partitions) with poor pruning. Put lower-cardinality equality columns first, then the date range; avoid very high-cardinality keys (use an expression like `TO_DATE(ts)`). For point lookups on high-cardinality columns use Search Optimization instead.

Here the measure of success is not "index used" but **partitions/bytes scanned vs total**.

## 10. Checklist

- [ ] Every equality column is before every sort column
- [ ] Sort columns are in ORDER BY order and direction
- [ ] Range columns come last; the most selective one first
- [ ] Checked whether E-R-S is better because the range is very selective (5.1)
- [ ] No function / cast / collation mismatch on indexed columns
- [ ] Pagination tie-breaker (e.g. `id`) sits right after the sort columns, and the final index matches the final rewritten ORDER BY
- [ ] Covering considered for hot queries; PK not duplicated in InnoDB
- [ ] Every DDL / EXPLAIN option used exists in the user's version (or is marked with its minimum version)
- [ ] Compared against existing indexes: extend or replace instead of adding
- [ ] Write cost justified by the traffic numbers
- [ ] Verified with the actual plan
