**Short version: I don't see a performance problem. Ship it as is.** This is textbook keyset pagination, and the plan shows it doing what it should.

## Why it's fine

- **The index matches the query exactly.** `(author_id, created_at DESC, id DESC)` covers the equality on `author_id`, the row-value range `(created_at, id) < ($2, $3)`, and the `ORDER BY created_at DESC, id DESC`. Both sort columns go the same direction, so Postgres can use the row comparison as an **Index Cond** (it does, look at the plan) rather than a post-scan Filter. It seeks straight to the cursor position and reads forward.
- **No sort node and no filter.** The scan returns rows already in order and stops after 20. The estimate `rows=399812` on the Index Scan line is the full matching range; the `Limit` cuts it off, which is why actual rows = 20.
- **Cost is constant per page.** 24 buffers (about 3–4 index pages plus about 20 heap pages) and 0.09 ms. With keyset pagination that cost doesn't grow with page depth, so page 1 and page 20,000 for the 400k-post author cost about the same. The `OFFSET` approach would have had that problem; this query doesn't.
- **The tie-breaker is correct.** `id` is the PK, so `(created_at, id)` is unique and you won't skip or repeat rows when timestamps collide.
- **600 QPS is easy at this cost.** About 24 buffer hits × 600 is around 15k buffer accesses/s, which is trivial.
- **Prepared statements / generic plans are safe here.** Each parameter is either an equality or a range on the leading index columns, so the generic plan is the same index scan. You won't get the "custom plan good, generic plan bad" flip.

## Things I'd check (correctness and ops, not query rewrites)

1. **Cursor timestamp precision.** `timestamptz` stores microseconds. If your app serializes the cursor `created_at` at millisecond precision (JavaScript `Date`, many JSON encoders), the cursor gets rounded. Rows can then be **skipped or duplicated** at page boundaries. Pass the exact value back, as the full-precision string or epoch microseconds. This is the most likely real bug in feeds like this.
2. **First page.** Use a separate query with no cursor predicate, rather than passing sentinel values like `'infinity'` or a max bigint. Sentinels also work with this index, but the plain query is clearer and avoids type and sentinel edge cases.
3. **Warm vs cold cache.** Your EXPLAIN shows `shared hit=24` with no reads, so everything came from cache. On a cold or evicted page you'd see about 20 random heap reads per request. That's usually still fine, but if p99 matters, check `pg_stat_statements` in production (`shared_blks_read` vs `shared_blks_hit` for this query) rather than relying on one warm run.
4. **Index-only scan? Probably not worth it.** You could avoid the heap by using `INCLUDE (title)` and keeping the table well vacuumed so the visibility map is set. But `title` is unbounded text, so the index would get bigger, and it would only save about 20 heap hits per request. I'd only do this if production stats show heap I/O on this query actually hurting.
5. **No redundant indexes.** If you also have a plain `(author_id)` index, this composite makes it redundant. Dropping it saves write cost and memory.

## Bottom line

Nothing to change in the SQL or the index. Before shipping, verify that the cursor round-trips `created_at` at full microsecond precision. After launch, watch `pg_stat_statements` for this query.
