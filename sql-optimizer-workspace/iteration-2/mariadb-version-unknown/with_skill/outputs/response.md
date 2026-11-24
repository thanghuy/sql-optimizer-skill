**First question: which exact MariaDB version is this?** Run `SELECT VERSION();` and tell me whether it's self-hosted or managed (RDS for MariaDB, SkySQL, Azure, etc.). The version matters for this query because `ORDER BY due_date ASC, amount DESC` mixes directions, and **MariaDB only supports descending index parts from 10.8** ([10.8 release notes](https://mariadb.com/kb/en/what-is-mariadb-108/): "Individual columns in the index can now be explicitly sorted in the ascending or descending order", MDEV-13756). Older versions accept `DESC` in `CREATE INDEX` but ignore it.

You don't know the version yet, so I give two answers below: one for **10.8 and later**, and a fallback for **10.6 and earlier** (or if you're not sure). Everything else uses syntax that works on 10.2+.

---

## Diagnosis

With only `idx_status (status)`, MariaDB either:
- uses `idx_status` to find every `status = 'overdue'` row, does one primary-key lookup per row, then runs a **filesort** over all of them to get the top 100, or
- decides `overdue` is too common and does a **full scan** of the 12M rows, then a filesort.

Either way the work grows with the number of overdue invoices, not with `LIMIT 100`. What the query needs is an index that hands the rows back already in `due_date ASC, amount DESC` order, so MariaDB can stop after 100 rows.

ESR mapping for this query:
- **E (equality):** `status = 'overdue'`
- **S (sort):** `due_date ASC, amount DESC`, in that order and with those directions
- **R (range):** none

## Recommendation

### Option A: MariaDB 10.8+ (recommended)

```sql
ALTER TABLE invoices
  ADD INDEX idx_status_due_amount (status, due_date ASC, amount DESC),
  ALGORITHM=INPLACE, LOCK=NONE;
```

- **ESR:** `status` = E. `due_date ASC, amount DESC` = S, matching the ORDER BY exactly, so there's **no filesort** and the scan stops after 100 index entries.
- **Covering?** The query also reads `customer_id` (and `id`, but InnoDB adds the PK to every secondary index automatically). With `LIMIT 100` that's only about 100 PK lookups, so covering adds little. If this page is hit very often, you can make it fully index-only with `(status, due_date ASC, amount DESC, customer_id)`. Don't add `id`.
- **Expected effect:** you read about 100 index entries plus about 100 row lookups, no matter how many invoices are overdue. That's typically milliseconds.
- **Cost:** roughly 30–50 bytes per row, so about **0.4–0.6 GB** for 12M rows (check the real size afterwards with `index_length`). Every INSERT pays for it, and so does every UPDATE of `status`, `due_date` or `amount`. Status changes (pending → overdue → paid) move the entry inside the index. The online build reads the whole table and needs temporary space about the size of the index, so run it off-peak and check free disk first.
- **Makes redundant:** `idx_status`. It's a leftmost prefix of the new index, and the new index also satisfies a foreign key on `status` if there is one. Retire it safely:
  ```sql
  ALTER TABLE invoices ALTER INDEX idx_status IGNORED;      -- 10.6+, instant, reversible
  -- watch for a few days; if nothing regresses:
  ALTER TABLE invoices DROP INDEX idx_status;
  -- if something regresses: ALTER TABLE invoices ALTER INDEX idx_status NOT IGNORED;
  ```
- **Rollback:** `ALTER TABLE invoices DROP INDEX idx_status_due_amount;`

### Option B: MariaDB 10.7 or older (or version unknown)

On these versions `amount DESC` in the index is silently stored as ASC. An index on `(status, due_date, amount)` gives `due_date ASC, amount ASC` order, which doesn't match `amount DESC`. You still get `Using filesort` over all overdue rows. Pick one of these, from cheapest to most work:

**B1. Change the sort if the product allows it.** If the admin page is fine with `amount ASC` within the same due date, or with `id` as the tie-breaker:
```sql
ALTER TABLE invoices ADD INDEX idx_status_due_amount (status, due_date, amount),
  ALGORITHM=INPLACE, LOCK=NONE;
-- query: ... WHERE status = 'overdue' ORDER BY due_date ASC, amount ASC LIMIT 100;
```
This is a full E-S index with no filesort, and the same plan as Option A.

**B2. Keep `amount DESC` with a negated generated column.** This needs 10.2+ (InnoDB can index virtual columns).
```sql
ALTER TABLE invoices
  ADD COLUMN amount_neg DECIMAL(15,2) AS (-amount) VIRTUAL,   -- use amount's precision; must be SIGNED
  ADD INDEX idx_status_due_amtneg (status, due_date, amount_neg),
  ALGORITHM=INPLACE, LOCK=NONE;

SELECT id, customer_id, amount, due_date, status
FROM invoices
WHERE status = 'overdue'
ORDER BY due_date ASC, amount_neg ASC      -- same order as amount DESC
LIMIT 100;
```
`amount_neg ASC` is the same order as `amount DESC`, so the index fully matches the ORDER BY. Caveats:
- If `amount` is `UNSIGNED`, `-amount` fails or overflows. Give the generated column a signed type, as above.
- The app query has to change to `ORDER BY ... amount_neg`.
- `ALGORITHM=INPLACE, LOCK=NONE` for an index on a virtual column depends on the minor version. If MariaDB rejects it, it tells you so instead of locking the table. Test on staging first.

**B3. If you can't change the query: use a covering index and accept a filesort.**
```sql
ALTER TABLE invoices ADD INDEX idx_status_due_amount_cust (status, due_date, amount, customer_id),
  ALGORITHM=INPLACE, LOCK=NONE;
```
The filesort stays, but it reads only the overdue slice of a narrow index (`Using index`, no 12M-row scan, no per-row PK lookups). With `LIMIT 100` MariaDB uses a bounded (priority-queue) sort. This is much better than today if overdue invoices are a small share of the table, but it still grows with the number of overdue rows.

**B4. Upgrade to a current LTS (10.11 / 11.4 / 11.8)** and use Option A. Re-check other hot queries after a jump to 11.x, because the 11.0 cost model rewrite can change plans.

Whichever option you pick, `idx_status` becomes redundant (same retire steps as above; `IGNORED` needs 10.6+, otherwise drop it after a staging test).

### Also check
- **Pagination:** if the page uses `LIMIT 100 OFFSET n`, deep pages get slower linearly. Switch to keyset pagination with a unique tie-breaker `id` placed right after the sort columns in the index. Tell me how the page pages and I'll adjust the index.
- **Statistics:** if the optimizer still prefers a full scan after the index exists, refresh stats with `ANALYZE TABLE invoices PERSISTENT FOR COLUMNS (status) INDEXES ();` to give `status` a histogram.

## Verify

Run on staging with production-like data first, or on a replica. This `SELECT` is read-only, so `ANALYZE` is safe even though it actually executes the query:

```sql
EXPLAIN
SELECT id, customer_id, amount, due_date, status
FROM invoices WHERE status = 'overdue'
ORDER BY due_date ASC, amount DESC LIMIT 100;

ANALYZE FORMAT=JSON
SELECT id, customer_id, amount, due_date, status
FROM invoices WHERE status = 'overdue'
ORDER BY due_date ASC, amount DESC LIMIT 100;
```

A good plan (Option A, B1, B2):
- `key` = the new index, `type = ref`, `ref = const`
- **no `Using filesort`** in `Extra` (in JSON: no `filesort` node)
- `r_rows` around 100 for the table, and `r_total_time_ms` in the low milliseconds

For B3, expect `Using where; Using index; Using filesort`, with `r_rows` equal to the number of overdue rows. Send me both plans (before and after) and I'll compare rows examined, time and the sort step, then iterate if needed.

## Assumptions / open questions

I've assumed InnoDB, that `status` is a VARCHAR/ENUM compared to a literal (no cast), and that overdue invoices are a minority of the 12M rows. To confirm the choice and size the cost, please send:

1. **Version and hosting:** `SELECT VERSION();`, plus managed or self-hosted.
2. **Structure:** `SHOW CREATE TABLE invoices\G` and `SHOW INDEX FROM invoices;` (I especially need the types of `status` and `amount`, and whether `amount` is UNSIGNED).
3. **Data volume and skew:**
   ```sql
   SELECT status, COUNT(*) FROM invoices GROUP BY status;   -- run on a replica / off-peak
   SELECT table_rows, ROUND(data_length/1024/1024) data_mb, ROUND(index_length/1024/1024) index_mb
   FROM information_schema.tables WHERE table_schema = DATABASE() AND table_name = 'invoices';
   ```
   Also: how many new invoices per day?
4. **Traffic:** how often is the page loaded (normal and peak)? Current latency and target? Inserts and status updates per second on `invoices`? Is it served from the primary or a replica?
5. **Current plan:** the output of the `EXPLAIN` above, run before you add anything.

Please don't send real customer data; schemas, counts and plans are enough.
