There are four separate problems in this code. Three of them stop Oracle from using indexes you already have, and the fourth is the per-order loop. Here they are in rough order of likely impact, followed by a rewrite.

## 1. `TO_CHAR(order_date, ...)` turns off the `order_date` index

```sql
WHERE status = 'NEW' AND TO_CHAR(order_date, 'YYYY-MM-DD') = '2026-11-23'
```

Because the indexed column is wrapped in a function, Oracle can't do a range scan on the `order_date` index. Unless the `status` index is selective, you get a full scan of all 8M orders, and `TO_CHAR` runs on every row. Compare the bare column against a half-open range instead:

```sql
WHERE status = 'NEW'
  AND order_date >= :day_start
  AND order_date <  :day_start + 1
```

This keeps the time-of-day part of `DATE` (`23:59:59` rows are included, midnight of today is not). It also lets the optimizer range-scan `order_date`.

Since `status = 'NEW'` is always part of the filter, a composite index helps further:

```sql
CREATE INDEX orders_status_date_ix ON orders (status, order_date);
```

With that index, Oracle goes straight to the roughly 5,000 rows it needs instead of reading a whole day of orders and then discarding the ones that aren't NEW.

## 2. N+1 queries, with no bind variables

The loop sends one `SELECT ... FROM order_items WHERE order_id = 12345` per order, so about 5,000 round trips. Because the id is concatenated into the SQL text, every statement is new to Oracle and gets **hard parsed**. That means 5,000 parses each night, CPU and latch contention, and a shared pool full of single-use cursors. The extra `c2` cursors are also never closed.

One set-based query is better than 5,000 small ones. (Section 5 shows the rewrite.)

## 3. `ext_ref = 12345` (a number) disables the `ext_ref` index, and can crash

`ext_ref` is `VARCHAR2(30)`, but `str(ref)` puts a bare number into the SQL:

```sql
WHERE ext_ref = 12345
```

When Oracle compares a string column with a number, it converts the **column**: `TO_NUMBER(ext_ref) = 12345`. The index on `ext_ref` can't be used, so this is another full scan of the 8M-row `orders` table. It can also fail at runtime with `ORA-01722: invalid number` once any row has a non-numeric `ext_ref`.

The fix is to bind the value as a string:

```python
cur.execute("SELECT id FROM orders WHERE ext_ref = :ref", ref=str(ref))
```

One behavior change to check: the current numeric comparison treats `'00123'` and `'123'` as equal, and the string comparison won't. If the partner sends `123` while you have `'00123'` stored, you need to normalize the format, for example with `str(ref).zfill(n)` or by agreeing on a canonical format. Don't do it by wrapping the column in a function, because that disables the index again.

## 4. Building SQL by string concatenation

This is also a security issue. `ref` is "an int from the partner API" only as long as the partner and your parsing code keep it that way, which is an open door for SQL injection. Bind variables fix both the injection risk and the hard-parse problem.

## 5. Smaller things

- **`fetchall()` plus the default fetch size.** Raise `cursor.arraysize` (and `prefetchrows`) so each round trip brings back thousands of rows instead of the default 100. Iterate over the cursor instead of materializing everything.
- **`SELECT *`.** List the columns you actually export. Fewer bytes go over the network, and code that depends on `o[0]` won't break if someone adds a column.
- **Where "yesterday" comes from.** `yesterday_str()` uses the Python host's clock and timezone. If that differs from the database's, you can silently export the wrong day. Either compute the date in one agreed place (`TRUNC(SYSDATE) - 1` in SQL) or pass a timezone-aware date explicitly.
- **Consistency between the two queries.** If orders or items change while the export runs, the orders query and the items query can see different data. If that matters, run both in a read-only transaction (`SET TRANSACTION READ ONLY`) so they share one snapshot.

## Rewrite

There are two set-based queries: one for orders and one for all their items, sorted so the items can be streamed and grouped. This avoids repeating the wide order columns on every item row, as a single join would. Orders with no items still get exported.

```python
import datetime as dt
from itertools import groupby

ORDER_COLS = "o.id, o.ext_ref, o.status, o.order_date, o.customer_id"   # list what you export
ITEM_COLS  = "i.order_id, i.line_no, i.product_id, i.qty, i.price"     # list what you export

ORDERS_SQL = f"""
    SELECT {ORDER_COLS}
      FROM orders o
     WHERE o.status = 'NEW'
       AND o.order_date >= :day_start
       AND o.order_date <  :day_start + 1
     ORDER BY o.id
"""

ITEMS_SQL = f"""
    SELECT {ITEM_COLS}
      FROM order_items i
      JOIN orders o ON o.id = i.order_id
     WHERE o.status = 'NEW'
       AND o.order_date >= :day_start
       AND o.order_date <  :day_start + 1
     ORDER BY i.order_id
"""

def export(conn, ref=None):
    day_start = dt.datetime.combine(dt.date.today() - dt.timedelta(days=1), dt.time.min)

    # Optional: make both queries see the same snapshot
    conn.cursor().execute("SET TRANSACTION READ ONLY")

    with conn.cursor() as oc, conn.cursor() as ic:
        for c in (oc, ic):
            c.arraysize = 5000
            c.prefetchrows = 5000

        oc.execute(ORDERS_SQL, day_start=day_start)
        ic.execute(ITEMS_SQL, day_start=day_start)

        # Merge two streams that are both sorted by order id
        item_groups = groupby(ic, key=lambda r: r[0])
        cur_id, cur_items = next(item_groups, (None, None))

        for o in oc:
            oid = o[0]
            # Skip item groups for ids lower than this order (shouldn't happen, defensive)
            while cur_id is not None and cur_id < oid:
                cur_id, cur_items = next(item_groups, (None, None))
            if cur_id == oid:
                items = list(cur_items)
                cur_id, cur_items = next(item_groups, (None, None))
            else:
                items = []
            write_rows(o, items)

    conn.commit()  # ends the read-only transaction

    if ref is not None:
        with conn.cursor() as c:
            c.execute("SELECT id FROM orders WHERE ext_ref = :ref", ref=str(ref))
            return c.fetchall()
```

What changes: about 5,000 hard-parsed round trips become 2 parsed-once statements. Two full scans of `orders` become index range scans. For 5,000 orders × ~5 items, the items query is a nested loop over the `order_items.order_id` index (about 25k rows). That should take seconds, not hours.

If `write_rows` is cheap, consider streaming a single `LEFT JOIN` query and grouping on the client. It's simpler, at the cost of repeating the order columns on every row.

## Before you deploy, check where the 2 hours actually goes

Two hours across 5,000 orders is about 1.4 s per order. That is a lot even for an unbound, hard-parsed lookup on an indexed column (normally milliseconds). So I'd confirm these three things rather than assume the fixes above account for all of it:

1. **Data type of `order_items.order_id`.** Is it `NUMBER`, like `orders.id`? If it's a `VARCHAR2`, the same implicit-conversion problem as `ext_ref` makes every per-order lookup a full scan of 40M rows, which would explain most of the 2 hours.
2. **What `write_rows` does.** If it writes to a network share, an HTTP API, or does a per-row commit, it may be the real bottleneck. Time it separately (wrap each phase with `time.perf_counter()`).
3. **Execution plans.** Look at the plans the current statements actually get:
   ```sql
   SELECT sql_id, executions, elapsed_time/1e6 AS secs, sql_text
     FROM v$sql
    WHERE sql_text LIKE 'SELECT * FROM order%'
    ORDER BY elapsed_time DESC FETCH FIRST 20 ROWS ONLY;

   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('<sql_id>', NULL, 'ALLSTATS LAST'));
   ```
   For the new queries, look for `INDEX RANGE SCAN` on `orders` and on `order_items.order_id`, and for no `TABLE ACCESS FULL` on either table.

These are the things I'd want to know from you:
- What is the data type of `order_items.order_id`?
- Can `ext_ref` values have leading zeros or non-numeric characters?
- What does `write_rows` write to?
- Is there an existing index on `status`?

The answers tell us whether the rewrite above is the whole fix or whether there's a second bottleneck.
