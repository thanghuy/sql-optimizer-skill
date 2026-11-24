# Review: admin report query (PostgreSQL 16)

Short version: there's one **correctness bug** that probably makes the query return nothing at all, plus four separate performance problems. They add up. In the worst case it's tens of thousands of full scans of the 60M-row `orders` table.

I haven't run anything against your database. The plan behaviour below comes from the schema you gave me, so check it with `EXPLAIN (ANALYZE, BUFFERS)` before and after.

---

## 1. Correctness bug: `NOT IN` against a nullable column (fix this first)

```sql
WHERE c.id NOT IN (SELECT customer_id FROM blacklist)
```

You said `blacklist.customer_id` is sometimes NULL. When the subquery returns even one NULL, `c.id NOT IN (...)` evaluates to NULL instead of TRUE for every customer that isn't blacklisted, so **no rows ever pass**. If you have any email-only blacklist rows today, this report returns an empty result. It's quick to check:

```sql
SELECT count(*) FROM blacklist WHERE customer_id IS NULL;
```

It also hurts performance. PostgreSQL can't turn `NOT IN` into a proper anti-join because of these NULL semantics. You get a "hashed SubPlan" at best, and a per-row re-scan of the subquery if the hash doesn't fit in `work_mem`.

**Fix:** use `NOT EXISTS`. It has the semantics you almost certainly intended, and the planner can run it as a Hash Anti Join:

```sql
AND NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = c.id)
```

Related question: rows with a NULL `customer_id` are "blocked emails", but `blacklist` has no email column. That means this query can never exclude a customer by email. If blocked emails are meant to be excluded, the blacklist needs an email column (or the email lives in `reason`?), and the query needs a second `NOT EXISTS` on it.

## 2. `lower(c.email) = lower($1)` can't use any index → full scan of 4M customers

No index exists on `email`, and certainly none on `lower(email)`. So every call reads the whole `customers` table and applies `lower()` to 4M rows.

```sql
CREATE INDEX CONCURRENTLY customers_lower_email_idx
    ON customers (lower(email), created_at DESC, id DESC);
```

The leading `lower(email)` matches your predicate exactly. The trailing `created_at DESC, id DESC` lets the planner return rows already in the `ORDER BY` order, with no Sort step. (An alternative is the `citext` type, or a nondeterministic case-insensitive collation, but the expression index is the smallest change.)

## 3. Two correlated subqueries on `orders`, which has no `customer_id` index

```sql
(SELECT COUNT(*)        FROM orders o WHERE o.customer_id = c.id),
(SELECT MAX(created_at) FROM orders o WHERE o.customer_id = c.id)
```

`orders` only has its PK. **Each** subquery is therefore a sequential scan of 60M rows, run twice for every customer row it's evaluated for. This is almost certainly the main reason the endpoint got slow as the data grew.

There's an extra cost. Because of `OFFSET 50000`, the `Limit` node pulls 50,020 rows through the projection before it throws away the first 50,000. The subqueries run for all of those rows, not just the 20 you return.

**Fix:** index `orders` so one index range serves both aggregates:

```sql
CREATE INDEX CONCURRENTLY orders_customer_created_idx
    ON orders (customer_id, created_at);
```

With this index, `COUNT(*)` is an index-only range scan and `MAX(created_at)` is a single index probe (when the visibility map is current, so keep autovacuum healthy on `orders`). Also merge the two subqueries into one `LATERAL` so the index is walked once, not twice. And compute the aggregates only for the final 20 rows (see the rewrite below).

## 4. `OFFSET 50000` is deep pagination

`OFFSET` doesn't skip work. PostgreSQL produces and discards 50,000 rows to return 20, and the cost grows linearly with the page number. Switch to **keyset (seek) pagination**: the client sends the `(created_at, id)` of the last row it saw, and the query starts right after it.

## 5. `ORDER BY c.created_at DESC` is not deterministic

`created_at` isn't unique. Rows with the same timestamp can come back in any order, so pages can repeat or skip rows between requests. Add `id` as a tie-breaker. Keyset pagination needs it anyway.

## A logic question about the query itself

`lower(email) = lower($1)` normally matches one customer, maybe a handful. Combined with `OFFSET 50000`, that almost always returns an empty page. That suggests one of these:

- the email filter is supposed to be optional (for example `$1 IS NULL OR ...`),
- it's meant to be a partial match (`ILIKE '%...%'`, which needs a `pg_trgm` GIN index instead of the btree above), or
- emails really are shared by many accounts.

It's worth confirming. The right index depends on the answer. If the filter is optional, the access path you need is `customers_created_idx`, walked in `created_at` order, with keyset pagination.

---

## Suggested rewrite

```sql
-- $1 = email, $2 = last created_at seen, $3 = last id seen (both NULL for the first page)
SELECT p.id, p.name, s.order_count, s.last_order_at
FROM (
    SELECT c.id, c.name, c.created_at
    FROM customers c
    WHERE lower(c.email) = lower($1)
      AND NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.customer_id = c.id)
      AND ($2::timestamptz IS NULL OR (c.created_at, c.id) < ($2, $3))
    ORDER BY c.created_at DESC, c.id DESC
    LIMIT 20
) p
CROSS JOIN LATERAL (
    SELECT count(*)          AS order_count,
           max(o.created_at) AS last_order_at
    FROM orders o
    WHERE o.customer_id = p.id
) s
ORDER BY p.created_at DESC, p.id DESC;
```

Notes:
- The inner query picks the 20 customers first. The `orders` aggregates then run for just those 20 rows, not 50,020.
- An aggregate with no `GROUP BY` always returns exactly one row. `CROSS JOIN LATERAL` therefore keeps customers with zero orders (`order_count = 0`, `last_order_at = NULL`), same as your original scalar subqueries.
- If the client really needs to jump to an arbitrary page number, keep `OFFSET` for now. Indexes 2 and 3 plus the "limit first, aggregate later" structure still remove most of the cost. Keyset is what makes it constant-time.
- Row-wise comparison `(created_at, id) < ($2, $3)` works with the `(lower(email), created_at DESC, id DESC)` index. When the first-page branch (`$2 IS NULL`) causes a bad generic plan, issue two separate statements from the app (first page / next page) instead of the `OR`.

## Indexes summary

```sql
CREATE INDEX CONCURRENTLY orders_customer_created_idx ON orders (customer_id, created_at);
CREATE INDEX CONCURRENTLY customers_lower_email_idx   ON customers (lower(email), created_at DESC, id DESC);
-- optional; with 20k rows a hash anti-join is already cheap:
CREATE INDEX CONCURRENTLY blacklist_customer_id_idx   ON blacklist (customer_id);
ANALYZE customers; ANALYZE orders; ANALYZE blacklist;
```

Operational notes:
- Building the `orders` index on 60M rows takes a while and adds write I/O. `CONCURRENTLY` avoids blocking writes, but it can't run inside a transaction block. If it fails, it leaves an `INVALID` index you must drop and recreate.
- A `customer_id` index on `orders` is probably useful elsewhere too (customer detail pages, FK checks if you add a foreign key to `customers`).
- If this report is hit often and doesn't need to be real-time, keep `order_count` / `last_order_at` in a summary table (updated by trigger or periodic job). That removes the `orders` work entirely.

## Priority order

1. Replace `NOT IN` with `NOT EXISTS`. That's the correctness fix, and the report is likely returning wrong (empty) results right now.
2. Add `orders (customer_id, created_at)`. This is the biggest performance win.
3. Restructure so the aggregates run only for the returned page, in one `LATERAL`.
4. Add the `lower(email)` expression index (after confirming how the email filter is meant to work).
5. Move to keyset pagination with an `id` tie-breaker.

To confirm the plan improved, compare `EXPLAIN (ANALYZE, BUFFERS)` before and after. Look for the `Seq Scan on orders` under the SubPlans and the `Seq Scan on customers` to disappear.
