**Short answer: neither.** The index is fine and a hint won't fix it. The problem is the bind data type. Your plan output already shows this.

## What's happening

1. **Peeked Binds: `:START_TS (TIMESTAMP)`.** python-oracledb sends your `datetime` values as `TIMESTAMP` binds, but `orders.order_date` is a `DATE`.
2. **Predicate: `filter(INTERNAL_FUNCTION("ORDER_DATE") >= :START_TS ...)`.** Oracle's comparison rules promote the less precise type to the more precise one. So Oracle doesn't convert the bind to DATE. It converts **the column** to TIMESTAMP, row by row. That `INTERNAL_FUNCTION` is an implicit `CAST(order_date AS TIMESTAMP)`.
3. Once the column is wrapped in a function, the B-tree on plain `order_date` can't be used as an access predicate. The optimizer then has two choices: a full table scan, or a full scan of the whole index plus a filter. It picks the full table scan (1.9M buffers).
4. SQL Developer uses the index because `DATE '2026-11-23'` and `TO_DATE(...)` literals are of type DATE. Types match, so there's no conversion and you get a normal `INDEX RANGE SCAN`. You're comparing two different statements, not seeing a stats or index problem.

Why the other options don't help:
- **Rebuilding the index:** nothing is wrong with the index. Stats are fresh, and the estimate is reasonable (E-Rows 125K vs A-Rows 61K). A rebuild changes nothing.
- **`/*+ INDEX(orders IDX_ORDERS_DATE) */`:** the predicate on the column is still `INTERNAL_FUNCTION(order_date)`. The best the hint can give you is an `INDEX FULL SCAN` that reads all 50M index entries and filters them, not a range scan. It also hides the real bug, and you'd have to repeat the hint in every query that binds datetimes.
- **A function-based index on `CAST(order_date AS TIMESTAMP)`:** it would work, but it's an extra 50M-entry index that only exists to cover a client bug. Don't.

## Fix: bind the values as DATE

**Option A (preferred): tell the driver the bind type** with `setinputsizes`:

```python
import oracledb
from datetime import datetime

cur.setinputsizes(start_ts=oracledb.DB_TYPE_DATE, end_ts=oracledb.DB_TYPE_DATE)
cur.execute("""
    SELECT id, customer_id, total
    FROM orders
    WHERE order_date >= :start_ts AND order_date < :end_ts
""", start_ts=datetime(2026, 11, 23), end_ts=datetime(2026, 11, 24))
```

(`oracledb.DATETIME` is an alias that also maps to DATE. Call `setinputsizes` before every `execute` on that cursor, or set it once if you reuse the cursor for the same statement.)

**Option B: convert in SQL**, if you can't control the driver call:

```sql
WHERE order_date >= CAST(:start_ts AS DATE)
  AND order_date <  CAST(:end_ts   AS DATE)
```

Now the conversion is on the bind side (computed once), and the column stays bare, so the index works. One caveat: `CAST(... AS DATE)` drops fractional seconds. That doesn't matter for midnight boundaries like these. For an exclusive upper bound with fractions, though, such as `< 10:00:00.5`, it would silently exclude rows at exactly `10:00:00`.

You could also bind a `datetime.date` or a string with `TO_DATE(:s, 'YYYY-MM-DD')`. Options A and B are cleaner.

## Verify

After the change, run it from the service and check the new cursor. The changed bind type produces a new child cursor because of the bind mismatch, so don't look at the old child:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', NULL, 'ALLSTATS LAST +PEEKED_BINDS'));
```

Expected results:
- Peeked binds show `(DATE)`.
- The plan shows `TABLE ACCESS BY INDEX ROWID BATCHED ORDERS` over `INDEX RANGE SCAN IDX_ORDERS_DATE`.
- The predicate section shows **`access("ORDER_DATE">=:START_TS AND "ORDER_DATE"<:END_TS)`**, with no `INTERNAL_FUNCTION`.
- Buffers drop from 1.9M to a figure on the order of tens of thousands. The exact figure depends on the index's clustering factor, since 61K rows means up to 61K table block visits if the rows are scattered.

## Worth checking elsewhere

- **Other queries:** this affects every query in the service that binds a Python `datetime` against a `DATE` column. Grep for them, or find them in the shared pool:
  ```sql
  SELECT sql_id, sql_text FROM v$sql
  WHERE  sql_id IN (SELECT sql_id FROM v$sql_plan
                    WHERE  filter_predicates LIKE '%INTERNAL_FUNCTION%')
  AND    parsing_schema_name = '<APP_SCHEMA>';
  ```
  A small helper that always calls `setinputsizes` for date binds, or a convention of `CAST(:x AS DATE)`, prevents it from coming back.
- **If the query is still too slow after the fix:** with the range scan in place, most of the remaining cost is fetching table rows. A covering index `(order_date, customer_id, total, id)` would make it index-only. Only do this if the query is hot enough to justify the extra index on a 50M-row table.
