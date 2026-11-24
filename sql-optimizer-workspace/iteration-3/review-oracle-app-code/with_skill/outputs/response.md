The query text and the numbers you gave are enough to see the main problems, so here is the full review now. The questions at the end would let me confirm the index choice. I assume plain Oracle 19c, not Autonomous or Exadata. Everything below works on 19c.

## Diagnosis

There are three problems. Each one alone could explain most of the 2 hours:

1. **N+1 round trips.** The job sends 1 query for the orders, then **one query per order** for its items. That is about 5,000 extra queries, and each is a separate round trip and a separate **hard parse** (see 2).
2. **No bind variables.** Every `"... = " + str(o[0])` creates a new SQL text. Oracle has to hard-parse each one and store it in the shared pool. On a busy instance, 5,000 hard parses per night take real time and push useful cursors out of the cache. It is also an SQL injection hole.
3. **`TO_CHAR(order_date, 'YYYY-MM-DD') = '...'` cannot use the `order_date` index.** Oracle has to apply `TO_CHAR` to all 8M rows, so the plan is a `TABLE ACCESS FULL` on `orders` to find about 5,000 rows.

There is also a separate bug on the `ext_ref` lookup. It is a slowdown and a correctness risk (finding 4).

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `for o in cur.fetchall(): c2.execute("SELECT * FROM order_items WHERE order_id = " + str(o[0]))` | N+1 queries | **High** | About 5,000 round trips, each with a parse, execute and fetch. With an index on `order_items.order_id`, each query is fast on its own. The cost comes from latency × 5,000 plus parse overhead, and it grows with the number of orders. | Fetch all items for the night's orders in **one** set-based query (join or semi-join to the same filter), then group them in Python |
| 2 | `"... = '" + yesterday_str() + "'"`, `"... = " + str(o[0])`, `"... = " + str(ref)` | Literals concatenated instead of bind variables | **High** | A new SQL text for every value means a hard parse each time, shared pool churn and library cache latch/mutex contention. The python-oracledb statement cache (default 20 statements per connection) never gets a hit. **SQL injection** risk too, especially for `ref`, which comes from a partner API. | Use `:name` binds everywhere |
| 3 | `TO_CHAR(order_date, 'YYYY-MM-DD') = '<date>'` | Function on an indexed column, so the predicate is not SARGable | **High** | The index on `order_date` stores `order_date`, not `TO_CHAR(order_date)`. Oracle must compute the expression for every row, so it does a full scan of 8M rows to return about 5,000 | Half-open range on the raw column: `order_date >= TO_DATE(:d,'YYYY-MM-DD') AND order_date < TO_DATE(:d,'YYYY-MM-DD') + 1` |
| 4 | `WHERE ext_ref = " + str(ref)` (ref is an int, `ext_ref` is VARCHAR2(30)) | Implicit conversion of a string column compared to a number | **High** (correctness + speed) | Oracle converts the **column**, not the literal: `TO_NUMBER(ext_ref) = 123`. That disables the `ext_ref` index, so this is another full scan of `orders`. It can also return wrong results: `'00123'`, `'123'` and `' 123'` all match 123. It can fail with **ORA-01722: invalid number** as soon as any row has a non-numeric `ext_ref`, and whether it fails depends on the plan, so it can start failing after a stats refresh. | Bind it as a string: `ext_ref = :ref` with `str(ref)`. Make sure the string is formatted exactly as stored (leading zeros, if any). |
| 5 | `cur.execute("SELECT id FROM orders WHERE ext_ref = ...")` at the end of `export()` | Result is never fetched or used | Medium (bug) | The query runs and its result is thrown away. It also reuses `cur`, so any further use of the orders cursor would get confusing results. | Use its own cursor and `fetchone()`, and return or use the result (or remove the query if it is dead code) |
| 6 | `SELECT * FROM orders`, `SELECT * FROM order_items` | `SELECT *` | Medium | Moves every column, including wide or LOB columns if there are any, over the network on every fetch. It also prevents a covering index. It breaks silently when columns are added, and `o[0]` relies on column position. | List only the columns `write_rows` needs |
| 7 | `cur.fetchall()` with the default `arraysize` | Fetch tuning | Low–Medium | python-oracledb fetches 100 rows per round trip by default. That is fine for 5,000 orders, but after the rewrite the items query returns about 25,000 rows or more in one result set. | Set `cur.arraysize = 1000` (the docs suggest starting around 1,000 and benchmarking), and iterate over the cursor instead of `fetchall()` if the result is large |

## Rewritten code

This version uses two set-based queries: one for the orders and one for **all** their items. I chose that over one big join so that the order columns are not repeated on every item row (about 5 items per order). It runs 2 SQL statements instead of about 5,001, all with bind variables.

```python
import oracledb
from collections import defaultdict

ORDER_SQL = """
    SELECT o.id, o.ext_ref, o.status, o.order_date /* , ...only the columns write_rows needs */
    FROM   orders o
    WHERE  o.status = 'NEW'
    AND    o.order_date >= TO_DATE(:d, 'YYYY-MM-DD')
    AND    o.order_date <  TO_DATE(:d, 'YYYY-MM-DD') + 1
    ORDER  BY o.id
"""

ITEMS_SQL = """
    SELECT i.order_id, i.id, i.product_id, i.qty, i.price /* , ...needed columns */
    FROM   order_items i
    WHERE  i.order_id IN (
             SELECT o.id
             FROM   orders o
             WHERE  o.status = 'NEW'
             AND    o.order_date >= TO_DATE(:d, 'YYYY-MM-DD')
             AND    o.order_date <  TO_DATE(:d, 'YYYY-MM-DD') + 1
           )
    ORDER  BY i.order_id
"""

def export(conn, ref=None):
    d = yesterday_str()                      # 'YYYY-MM-DD'; stays a bind, not concatenated

    with conn.cursor() as cur:
        cur.arraysize = 1000
        cur.execute(ORDER_SQL, d=d)
        orders = cur.fetchall()              # ~5,000 rows: fine in memory

    items_by_order = defaultdict(list)
    with conn.cursor() as cur:
        cur.arraysize = 1000
        cur.execute(ITEMS_SQL, d=d)
        for row in cur:                      # streams ~25k rows, 1000 per round trip
            items_by_order[row[0]].append(row)

    for o in orders:
        write_rows(o, items_by_order.get(o[0], []))

    if ref is not None:
        with conn.cursor() as cur:
            cur.execute("SELECT id FROM orders WHERE ext_ref = :ref", ref=str(ref))
            row = cur.fetchone()
            return row[0] if row else None
```

Notes on the rewrite:
- **Keep `'NEW'` as a literal.** It is a constant, not user input, so it does not cause re-parsing. Leaving it visible to the optimizer lets it use a histogram on `status` if one exists.
- **The date stays a string bind converted with `TO_DATE` in SQL**, so the comparison is DATE against DATE and the index on `order_date` can be used. Avoid binding a Python `datetime` without setting its type. If it is bound as a TIMESTAMP, Oracle converts the column (`INTERNAL_FUNCTION("ORDER_DATE")`), which brings back the full-scan problem. If you prefer a datetime bind, call `cur.setinputsizes(d=oracledb.DB_TYPE_DATE)`.
- If both the app server and the DB server use the same time zone for "yesterday", you can use `order_date >= TRUNC(SYSDATE) - 1 AND order_date < TRUNC(SYSDATE)` and drop the bind. Keep `yesterday_str()` if the app's time zone is the one that defines "yesterday".
- **If `write_rows` writes to a file or another DB row by row, that may now be the bottleneck.** Batch it (for example `executemany()` for inserts) if it does DB writes.
- **Consistency:** the two queries run at slightly different times. If new NEW orders for yesterday can still arrive while the job runs, run both in one read-only transaction (`SET TRANSACTION READ ONLY` first) so they see the same snapshot. You can also use a single join query instead.

## Recommendation

**1. Code rewrite (above): do this first.** Expected effect: about 5,000 hard-parsed round trips become 2 parsed statements, and the 8M-row full scan on `orders` should become an index range scan. Measured on its own, the N+1 alone typically accounts for minutes to tens of minutes. It depends on network latency and shared pool pressure. Cost: none on the DB side. Rollback: redeploy the old code.

**2. Composite index for the nightly filter: only if the plan in Verify shows it is needed.**

```sql
CREATE INDEX orders_status_date_ix ON orders (status, order_date) ONLINE INVISIBLE;
-- test in one session:
ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;
-- if the plan is better:
ALTER INDEX orders_status_date_ix VISIBLE;
```
- ESR mapping: **E** = `status` (`= 'NEW'`), **S** = none (there is no ORDER BY to serve, and ordering about 5,000 ids is trivial), **R** = `order_date` (one-day range). The index seeks straight to the about 5,000 NEW rows from yesterday and reads nothing else.
- Why it may not be needed: after the rewrite, the existing `order_date` index can already range-scan one day. If one day holds, say, 5–20k orders, of which about 5,000 are NEW, that is cheap, and a new index on a write-heavy table would not pay off. The index matters if a day has many more rows than the NEW ones, or if the plan still shows `filter("STATUS"='NEW')` after reading many rowids.
- Cost: one more index to maintain on every insert into `orders` and on every `status` update (each status change moves the index entry). Size is roughly 8M × (status + date + rowid), so a few hundred MB at most. Build it `ONLINE` outside peak hours. A covering variant is not useful while you need most columns of `orders`.
- Overlap with existing indexes: if you already have a single-column index on `status`, this one makes it redundant once you have confirmed nothing else needs it. Do not drop the `order_date` index unless no other query uses it.
- Rollback: `DROP INDEX orders_status_date_ix;` (or keep it `INVISIBLE`).

**3. Items side: no new index needed.** The existing index on `order_items.order_id` supports the semi-join (about 5,000 index probes in one execution, or a hash join if the optimizer prefers it). Check its `clustering_factor`. If the items of one order are spread across many blocks, about 25k rowid lookups can still be thousands of single-block reads. That is still seconds, not hours.

**4. `ext_ref`: no index change.** Once the value is bound as a string, the existing index is usable again.

## Verify

Run the rewritten statements once from SQL*Plus or SQL Developer with the same bind value, then show the actual plan:

```sql
VARIABLE d VARCHAR2(10)
EXEC :d := '2026-11-23'

SELECT /*+ GATHER_PLAN_STATISTICS */ o.id, o.status, o.order_date
FROM   orders o
WHERE  o.status = 'NEW'
AND    o.order_date >= TO_DATE(:d,'YYYY-MM-DD')
AND    o.order_date <  TO_DATE(:d,'YYYY-MM-DD') + 1;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PEEKED_BINDS +PREDICATE'));
```
Do the same for `ITEMS_SQL`. These are SELECTs, so running them is safe. Run them on a replica or staging copy if you can.

A good plan looks like this:
- `orders`: `INDEX RANGE SCAN` on the `order_date` index (or on `ORDERS_STATUS_DATE_IX`), with `access("ORDER_DATE">=... AND "ORDER_DATE"<...)` in the Predicate Information section. No `TABLE ACCESS FULL ORDERS`, and no `TO_CHAR` or `INTERNAL_FUNCTION` on `ORDER_DATE`.
- If the plan shows `access` on `order_date` but `filter("STATUS"='NEW')` on the table line, with A-Rows of the range scan much larger than 5,000, create the composite index from recommendation 2.
- `order_items`: an index range scan on `order_id` under a nested loop (or a hash join), A-Rows about 25k, and Buffers in the low tens of thousands, not millions.
- `ext_ref`: `INDEX RANGE SCAN` / `UNIQUE SCAN` with `access("EXT_REF"=:REF)`, not `filter(TO_NUMBER("EXT_REF")=...)`.
- Compare E-Rows and A-Rows. If they differ by 10× or more on `status = 'NEW'`, refresh statistics with `EXEC DBMS_STATS.GATHER_TABLE_STATS('APP','ORDERS', cascade => TRUE)` and check that `status` has a histogram.

Also time the whole job before and after. If it is still slow after the rewrite, `write_rows` is the next suspect.

## Assumptions / open questions

If you can answer these, I can confirm or drop the composite index:
1. **Exact version / platform:** `SELECT banner_full FROM v$version;`. Is this on-premises, RDS, OCI or Exadata?
2. **Existing indexes on `orders`:** is there an index on `status`? Is the `order_date` index single-column? Get them with:
   ```sql
   SELECT index_name, column_name, column_position FROM all_ind_columns
   WHERE table_name = 'ORDERS' ORDER BY index_name, column_position;
   ```
3. **How many orders per day in total** (all statuses), and how many rows have `status = 'NEW'` overall? Do NEW rows from earlier days pile up?
   ```sql
   SELECT status, COUNT(*) FROM orders GROUP BY status;
   SELECT COUNT(*) FROM orders WHERE order_date >= TRUNC(SYSDATE)-1 AND order_date < TRUNC(SYSDATE);
   ```
4. **Write traffic on `orders`** (inserts and status updates per second at peak), to decide whether a second index is worth its cost: `SELECT inserts, updates, deletes FROM all_tab_modifications WHERE table_name='ORDERS';`
5. **What does `write_rows` do?** Write to a file, or insert into another DB? Does it run on the same network as the DB? This decides whether it becomes the next bottleneck.
6. **Format of `ext_ref`:** is it always digits with no leading zeros? If not, tell me how the partner's int maps to the stored string, so the bind matches exactly.

I assumed about 5 items per order (40M / 8M) and a job host that is not on the same machine as the DB, so each round trip has real network latency.
