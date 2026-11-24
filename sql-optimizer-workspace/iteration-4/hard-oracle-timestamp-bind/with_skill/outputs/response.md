**Short answer: don't rebuild, and don't add a hint.** The index is fine. The bind type is wrong. The driver sends your Python `datetime` values as **TIMESTAMP**, and the column is **DATE**. Oracle then has to convert the column (`INTERNAL_FUNCTION("ORDER_DATE")`) before it can compare, and an index on `order_date` can't seek on a converted column. Bind the parameters as DATE and the range scan comes back.

(Assuming Oracle 19c, as you said. Everything below is valid on 19c.)

## Diagnosis

The plan you pasted already shows the cause:

- **Peeked Binds: `:START_TS (TIMESTAMP)`, `:END_TS (TIMESTAMP)`.** The binds arrive as TIMESTAMP.
- **Predicate: `filter(INTERNAL_FUNCTION("ORDER_DATE") >= :START_TS ...)`.** TIMESTAMP ranks higher than DATE in Oracle's datatype precedence, so Oracle promotes the **column** (DATE → TIMESTAMP) on every row instead of converting the bind. The predicate becomes a `filter` on a full scan. For an index range scan it would have to be an `access(...)` predicate.
- **Literals in SQL Developer use the index** because `DATE '2026-11-23'` or `TO_DATE(...)` is a DATE. The types match, nothing is converted, and the predicate is SARGable. That's why the console and the app behave differently: they're the same SQL with different bind types.

What it's **not**:

- **Not statistics.** E-Rows is 125K and A-Rows is 61K. That's about 2× off, well within normal. Gathering stats last night was fine, and gathering again won't change anything.
- **Not a broken or bloated index.** If it were, the literal version wouldn't use it either. A rebuild only costs you an outage window.
- **Not something a hint can fix.** `/*+ INDEX(orders idx_orders_date) */` can't turn `INTERNAL_FUNCTION(col) >= :b` into a range seek. At best you'd get an `INDEX FULL SCAN` of all 50M entries followed by 50M conversions, which is often worse than the full table scan. It would also freeze a plan around a bug instead of fixing it.

The cost today: about 1.9M buffer gets to return 61K rows (one day out of 50M rows, roughly 0.12%). A selective range like this should be an index range scan.

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `start_ts=datetime(2026, 11, 23), end_ts=datetime(2026, 11, 24)` with `order_date >= :start_ts AND order_date < :end_ts` | Driver bind type mismatch: binds are TIMESTAMP, column is DATE | **High** | Oracle converts the column on every row (`INTERNAL_FUNCTION`), so the predicate isn't SARGable. Result: full scan of 50M rows, 1.9M buffers. | Bind as DATE: `cur.setinputsizes(start_ts=oracledb.DB_TYPE_DATE, end_ts=oracledb.DB_TYPE_DATE)`, or `CAST(:start_ts AS DATE)` on the parameter side |
| 2 | `SELECT id, customer_id, total` via `IDX_ORDERS_DATE (order_date)` | Not covering: about 61K `TABLE ACCESS BY INDEX ROWID` lookups after the fix | Low (optional) | Each matched row costs a table block visit. The real cost depends on `clustering_factor`. | Only if this query is hot: a covering index (see Recommendation 2) |

Things that are already fine and don't need changing: the half-open range `>= start AND < end` is the correct shape for a DATE/TIME range (no `BETWEEN`, no `TRUNC(order_date)`), and you already use bind variables, so there's no hard-parse problem.

## Rewritten query

**Option A, preferred: fix the bind type in Python. The SQL text stays the same.**

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

`setinputsizes()` applies to the next `execute()` on that cursor, so call it before every execute, or wrap the pair in a small helper. python-oracledb maps Oracle DATE to `DB_TYPE_DATE` and accepts `datetime.date` and `datetime.datetime` values for it ([python-oracledb, Appendix A, supported data types](https://python-oracledb.readthedocs.io/en/stable/user_guide/appendix_a.html)).

**Option B: convert on the parameter side in SQL. Works with any driver or ORM.**

```sql
SELECT id, customer_id, total
FROM   orders
WHERE  order_date >= CAST(:start_ts AS DATE)
AND    order_date <  CAST(:end_ts   AS DATE)
```

The conversion now applies to the bind, once, not to the column. The predicate becomes `access("ORDER_DATE">=CAST(:START_TS AS DATE) ...)`.

Edge case for Option B: `CAST(timestamp AS DATE)` drops fractional seconds. With midnight boundaries, as here, nothing changes. If you ever pass a boundary like `10:00:00.700`, it becomes `10:00:00`, which can include rows from that exact second. Option A has the same truncation, since DATE has no fractional seconds. Either way it matches what the column can store.

Also search the service for the same pattern elsewhere. Every `datetime` compared to a `DATE` column through this driver path has the same problem: `WHERE`, `JOIN`, `UPDATE ... WHERE`, `DELETE ... WHERE`. If you use SQLAlchemy anywhere, map these columns as `oracle.DATE`, not `TIMESTAMP`/`DateTime` with fractional precision.

## Recommendation

1. **Bind as DATE (Option A or B).** No DDL needed.
   ESR mapping: there's no equality or sort column. `order_date` is the only **R** (range) column, and the existing `IDX_ORDERS_DATE (order_date)` is already the correct index for this query.
   Expected effect: `INDEX RANGE SCAN IDX_ORDERS_DATE` + `TABLE ACCESS BY INDEX ROWID BATCHED ORDERS`. Buffers should drop from about 1.9M to roughly the number of table blocks holding one day of orders, plus a few hundred index leaf blocks. That's typically tens of thousands of gets when rows are inserted in date order (low clustering factor), and up to about 61K if they're scattered.
   Cost: none on writes. The first execution with DATE binds creates a new child cursor (bind mismatch), which is expected.
   Rollback: revert the code change.

2. **Optional covering index, only if this query is hot** (many executions per minute, tight latency SLO) and the plan after fix 1 shows most buffers in `TABLE ACCESS BY INDEX ROWID`:
   ```sql
   CREATE INDEX idx_orders_date_cov ON orders (order_date, customer_id, total, id) ONLINE INVISIBLE;
   ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;   -- test in your session only
   -- if it wins: ALTER INDEX idx_orders_date_cov VISIBLE;  then consider dropping IDX_ORDERS_DATE
   ```
   ESR: R = `order_date` (leading, so it can seek). `customer_id, total, id` are trailing columns that make it covering. Oracle has no `INCLUDE` clause.
   Cost: a second index on a 50M-row table: extra work on every INSERT and on updates to `total`/`customer_id`, about the size of a 4-column index in disk space, plus temp space during the online build. Once it's visible, it makes `IDX_ORDERS_DATE` redundant, since the leading column is the same. Before dropping the old index, check that no constraint or other query depends on it.
   Rollback: `DROP INDEX idx_orders_date_cov;`
   Don't do this until fix 1 is in and you've seen the new plan and the traffic numbers. Fix 1 alone may be enough.

## Verify

After deploying the bind fix, run the query once from the service, then:

```sql
-- find the cursor and confirm there is a new child with DATE binds
SELECT sql_id, child_number, executions, buffer_gets/NULLIF(executions,0) gets_per_exec,
       ROUND(elapsed_time/NULLIF(executions,0)/1000,1) avg_ms
FROM   v$sql
WHERE  sql_text LIKE 'SELECT id, customer_id, total%FROM orders%' ;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('<sql_id>', <new_child>, 'ALLSTATS LAST +PEEKED_BINDS'));

-- optional: why a new child was created (expect BIND_MISMATCH / BIND_LENGTH_UPGRADEABLE = 'Y')
SELECT child_number, bind_mismatch FROM v$sql_shared_cursor WHERE sql_id = '<sql_id>';
```

`ALLSTATS LAST` needs row-source statistics. For the app's cursor, either set `STATISTICS_LEVEL=ALL` for a test session or add `/*+ GATHER_PLAN_STATISTICS */` temporarily. Otherwise A-Rows and Buffers are empty.

A good plan looks like this:
- Peeked Binds show `(DATE)`, not `(TIMESTAMP)`
- `INDEX RANGE SCAN | IDX_ORDERS_DATE` with **`access("ORDER_DATE">=:START_TS AND "ORDER_DATE"<:END_TS)`** and no `INTERNAL_FUNCTION`
- `TABLE ACCESS BY INDEX ROWID BATCHED | ORDERS` above it
- Buffers far below 1.9M, with A-Rows still about 61K

If you still see a full scan **with DATE binds and an `access` predicate available**, the problem is cost, not type. Then send me:

```sql
SELECT index_name, clustering_factor, leaf_blocks, num_rows, last_analyzed
FROM   all_indexes WHERE table_name = 'ORDERS' AND index_name = 'IDX_ORDERS_DATE';
SELECT num_rows, blocks FROM all_tab_statistics WHERE table_name = 'ORDERS';
SELECT column_name, num_distinct, histogram FROM all_tab_col_statistics
WHERE  table_name = 'ORDERS' AND column_name = 'ORDER_DATE';
```

A `clustering_factor` close to `num_rows` (50M) means a day's rows are scattered across the table. In that case the covering index (Recommendation 2) is the right next step, not a hint.

## Assumptions / open questions

- **python-oracledb mode and version** (thin or thick; `oracledb.__version__`): the fix is the same, but it helps in case an ORM or wrapper sets bind types for you.
- **Traffic:** how often this runs (per minute, at peak) and the latency target. That decides whether Recommendation 2 is worth its write cost.
- **Write rate on `orders`** (inserts/updates per second): relevant only if you add the covering index.
- **Do other queries against DATE columns use the same code path?** If so, they're very likely full scanning for the same reason. Check `v$sql` for high `buffer_gets` statements whose plans show `INTERNAL_FUNCTION`.
- I haven't connected to your database. Everything above is based on the plan and predicate section you pasted.

Sources:
- [python-oracledb Appendix A, supported Oracle data types (DATE ↔ DB_TYPE_DATE)](https://python-oracledb.readthedocs.io/en/stable/user_guide/appendix_a.html)
- [python-oracledb binding guide (setinputsizes)](https://python-oracledb.readthedocs.io/en/latest/user_guide/bind.html)
- [Oracle 19c SQL Tuning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [jOOQ manual, Oracle DATE type and the INTERNAL_FUNCTION problem with TIMESTAMP binds](https://www.jooq.org/doc/dev/manual/reference/reference-data-types/data-types-oracle-date)
- [Oracle Support note 1557194.1, poor performance with JDBC bind variables against a DATE column](https://support.oracle.com/knowledge/Oracle%20Database%20Products/1557194_1.html)
