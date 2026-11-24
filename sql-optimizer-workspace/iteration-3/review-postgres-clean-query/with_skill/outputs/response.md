Short answer: **this query is ready to ship as it is.** It uses the right pagination pattern, it has the right index for it, and your plan for the worst-case author proves it. I have no performance changes to recommend. Below is why, plus three things to check in the application code around it. None of them need a change to the SQL.

## Diagnosis

The plan for your biggest author (about 400k posts) is close to the best you can get:

- **`Index Scan using posts_author_created_id_idx`**: both `author_id = 42` and `ROW(created_at, id) < ROW(...)` show up under **`Index Cond`**. So the planner jumps straight to the cursor position inside that author's part of the index. There is no `Filter` line and no `Rows Removed by Filter`.
- **There is no `Sort` node.** The index order `(author_id, created_at DESC, id DESC)` matches `ORDER BY created_at DESC, id DESC`, so the `Limit` stops after 20 rows.
- **The scan reads 20 rows to return 20 rows, using 24 buffers**, all of them cache hits, in 0.09 ms. The estimate of `rows=399812` is the author's total match count, before the LIMIT. It is not a bad estimate, because the `Limit` node's `rows=20` matches the actual count.
- **The cost does not depend on page depth.** Because this is keyset pagination, page 1,000 costs the same as page 1. With OFFSET it would grow with every page.

At 600 QPS this is about 600 × 24 ≈ 15k buffer hits per second, which is trivial.

Mapped to the ESR rule, the existing index is a textbook fit:
- E = `author_id`
- S = `created_at DESC, id DESC`, with `id` as the unique tie-breaker placed right after the sort column, in the same direction
- R = the row comparison on `(created_at, id)`. This is the same column pair as S, so the range is simply a starting point in the already-sorted index.

## Issues found

No significant issues in the query text. I checked it against the usual anti-patterns:

- **The row comparison `(created_at, id) < ($2, $3)` is not a problem.** It is the recommended keyset form in PostgreSQL, and the plan shows it used as an index condition.
- **The SELECT list is narrow**: `id, title, created_at`, with no `SELECT *`. The wide `body` column is not read.
- **There is a unique tie-breaker (`id`)**, so pages are stable and no rows are duplicated or skipped between pages.
- **There is no function or cast on any indexed column.**

These are optional checks on the code around the query (Low severity, and only relevant if they apply):

| # | Where | Problem | Severity | Why it matters | Fix |
|---|---|---|---|---|---|
| 1 | `($2, $3)` on the **first page** | If the app sends `NULL` for the cursor on page 1, the query returns **0 rows**. The PG 16 docs say that when a compared pair contains a null, "the result of the row comparison is unknown (null)" ([9.24.5](https://www.postgresql.org/docs/16/functions-comparisons.html)). | Low (correctness) | Not a speed problem. It would be a visible bug on the first page. | Use a separate first-page query without the row predicate (same index, same plan shape). Or send a sentinel cursor such as `('infinity'::timestamptz, 9223372036854775807)`. |
| 2 | `$2` round-trip through the client | `timestamptz` has microsecond precision. If the cursor goes through something with millisecond precision (JavaScript `Date`, some JSON serializers), the next page can **skip rows** that have the same millisecond as the last row. | Low (correctness) | Not a speed problem. Rows silently disappear from the feed. | Pass `created_at` back as the exact string PostgreSQL returned, or encode the cursor as text or epoch microseconds. |
| 3 | Prepared statement (`$1` …) | After 5 executions, PostgreSQL may switch to a **generic plan** (`plan_cache_mode = auto`). The plan you showed used literal values. | Low (verify only) | With an equality on the leading column plus LIMIT, the generic plan should be the same index scan. It is worth one quick check, though, because the 400k-row author is very different from a typical one. | Run `EXPLAIN (GENERIC_PLAN)` (available from PG 16; see the Verify section). |

## Rewritten query

No rewrite is needed. The query as written is correct and efficient on PostgreSQL 16:

```sql
SELECT id, title, created_at
FROM posts
WHERE author_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

If you choose the separate first-page query from issue 1:

```sql
SELECT id, title, created_at
FROM posts
WHERE author_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## Recommendation

1. **Keep the existing index `posts_author_created_id_idx` and the query as they are.** No new index is needed.
   - ESR mapping: E = `author_id`, S = `created_at DESC, id DESC`, R = `(created_at, id) <` cursor, served by the same index columns.
   - Expected effect: none, because it is already about 0.1 ms and 24 buffers for the worst-case author.
2. **I do not recommend a covering index.** An index such as `(author_id, created_at DESC, id DESC) INCLUDE (title)` would enable an Index Only Scan, but:
   - it only saves about 20 heap page visits per call, which are already cache hits;
   - it copies every `title` into a 25M-row index, so the index gets much bigger and writes cost more;
   - it only helps if the visibility map stays current (`Heap Fetches` near 0).

   Consider it only if heap reads (`shared read`) appear in production plans and p99 latency misses your target.
3. **Do not add hints and do not drop anything.** If another index exists whose columns are only a leading part of this one, for example `posts (author_id)`, that index is now redundant and probably safe to drop. Check `idx_scan` first, and keep it if it backs a constraint:
   ```sql
   SELECT s.indexrelname, s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
   FROM pg_stat_user_indexes s WHERE s.relname = 'posts' ORDER BY s.idx_scan;
   ```

## Verify

1. **Generic plan** (what the prepared statement may use after 5 executions; PG 16+). This does not execute the query:
   ```sql
   EXPLAIN (GENERIC_PLAN)
   SELECT id, title, created_at FROM posts
   WHERE author_id = $1 AND (created_at, id) < ($2, $3)
   ORDER BY created_at DESC, id DESC LIMIT 20;
   ```
   A good result is `Limit -> Index Scan using posts_author_created_id_idx`, with both conditions under `Index Cond` and no `Sort`.
2. **Deep cursor for the biggest author.** Pick a cursor near that author's oldest posts and confirm it still shows about 20 rows and a few dozen buffers:
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) SELECT ... WHERE author_id = 42 AND (created_at, id) < ('<old timestamp>', <old id>) ...;
   ```
3. **The plan the app actually gets in production.** Temporarily enable `auto_explain` with `auto_explain.log_min_duration = '5ms'`, and check `pg_stat_statements.mean_exec_time` for this statement after release. It should stay well under 1 ms.

## Assumptions / open questions

- **Exact minor version and managed service.** I assumed PostgreSQL 16.x, so everything above, including `EXPLAIN (GENERIC_PLAN)`, works on 16.0 and later. If you are on RDS, Aurora or Cloud SQL, `auto_explain` may have to be enabled through the parameter group.
- **First page.** How does the app call this query for the first page: does it send NULL, a sentinel value, or use a separate query? (See issue 1.)
- **Cursor encoding.** How is the cursor passed back to the client, and does it keep microsecond precision? (See issue 2.)
- **Writes.** I assumed write traffic on `posts` is moderate. That only matters if you later add the `INCLUDE (title)` index, which I don't recommend right now.
