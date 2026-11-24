The right index depends on your **MariaDB version**, because the query sorts in two directions (`due_date ASC, amount DESC`). Real descending indexes only arrived in **MariaDB 10.8**. Before 10.8, MariaDB accepts `DESC` in an index definition but ignores it.

Check what prod is running (and any replicas, since they may differ):

```sql
SELECT VERSION();
```

Pick from the two cases below based on the result.

---

## Why `idx_status` alone is slow

`idx_status` finds the overdue rows, but it doesn't help with the `ORDER BY`. MariaDB has to fetch **every** overdue row (each one a lookup back into the clustered PK), filesort them all, and then throw away everything past the first 100. On 12M rows with a large overdue backlog, that's the cost. You want an index that already returns rows in the final order, so MariaDB can stop after reading 100 entries.

---

## Case A: MariaDB 10.8 or newer

Use a composite index whose column directions match the `ORDER BY`:

```sql
ALTER TABLE invoices
  ADD INDEX idx_status_due_amount (status, due_date ASC, amount DESC, customer_id),
  ALGORITHM=INPLACE, LOCK=NONE;
```

- `status` comes first because it's an equality filter.
- `due_date ASC, amount DESC` matches the sort exactly, so there's **no filesort**. MariaDB reads the first 100 index entries and stops.
- `customer_id` is added at the end so the index **covers** the query. InnoDB secondary indexes include the PK (`id`) automatically, and `status`, `due_date` and `amount` are already in the index, so MariaDB never has to go back to the table. If you'd rather keep the index smaller, drop `customer_id`. That costs 100 PK lookups, which is cheap.

**Caveat:** if you upgraded from a version older than 10.8, any index created back then with `DESC` is still ascending until you rebuild it. Create this index fresh on 10.8+.

## Case B: MariaDB older than 10.8 (10.6 LTS, 10.5, etc.)

`(status, due_date, amount DESC)` won't do what you want on these versions: the `DESC` is ignored, the directions don't match, and MariaDB falls back to a filesort. You have three options:

**Option 1: generated column holding the negated amount (recommended; works on 10.2+)**

```sql
ALTER TABLE invoices
  ADD COLUMN neg_amount DECIMAL(14,2) AS (-amount) VIRTUAL,   -- use the same type/precision as amount
  ADD INDEX idx_status_due_negamt (status, due_date, neg_amount, customer_id, amount);
```

Then change the query to sort on it. Ascending on `-amount` gives the same order as descending on `amount`:

```sql
SELECT id, customer_id, amount, due_date, status
FROM invoices
WHERE status = 'overdue'
ORDER BY due_date ASC, neg_amount ASC
LIMIT 100;
```

Now every sort column is ascending, so the index delivers rows already in order and there's no filesort. This also keeps working unchanged after you upgrade to 10.8+.

Note: adding a VIRTUAL column is normally a quick metadata-only change, but building the index still scans the table. Run it in a quiet window or use an online schema-change tool (`pt-online-schema-change` or `gh-ost`). Also, `ALGORITHM=INPLACE, LOCK=NONE` may be refused for an index on a virtual column, depending on the version.

**Option 2: no schema change beyond a plain index**

```sql
ALTER TABLE invoices ADD INDEX idx_status_due (status, due_date, amount),
  ALGORITHM=INPLACE, LOCK=NONE;
```

The rows still need a filesort here, because MariaDB can't use a partial index order for the sort. Two things still get better:
- The filter and sort columns are read from the index, so filtering is cheaper.
- The filesort uses a priority queue for `LIMIT 100`.

But it still reads every overdue row, so it's a partial fix only. It's fine if the overdue set is small (say, tens of thousands of rows) and bad if it's millions.

**Option 3: change the ordering.** If the business would accept `ORDER BY due_date ASC, amount ASC` (or just `due_date ASC, id`), the plain `(status, due_date, amount)` index avoids the filesort on any version. Product needs to confirm this, because it changes which rows show up first when many invoices share a due date.

---

## Verify

Run EXPLAIN before and after:

```sql
EXPLAIN SELECT id, customer_id, amount, due_date, status
FROM invoices WHERE status = 'overdue'
ORDER BY due_date ASC, amount DESC LIMIT 100;
```

What a good plan looks like:
- `key` is the new index.
- `Extra` shows `Using where; Using index` (or `Using index condition`).
- `Extra` does **not** show `Using filesort`.
- `rows` is small. `EXPLAIN` sometimes shows a large estimate even when the LIMIT stops the scan early, so also check with `ANALYZE FORMAT=JSON SELECT ...`, which reports the rows actually read.

## Cleanup

Once the new index is in place, `idx_status` is redundant: `status` is the leading column of the new index. Drop it to save write overhead and space on a 12M-row table, after checking that nothing uses an index hint on it:

```sql
ALTER TABLE invoices DROP INDEX idx_status;
```

## Questions that would sharpen this

1. **Your exact MariaDB version** (`SELECT VERSION();`). This decides between Case A and Case B.
2. **Roughly how many rows have `status = 'overdue'`** (`SELECT COUNT(*) FROM invoices WHERE status='overdue';`). If it's only a few thousand, even Option 2 is enough. If it's hundreds of thousands or more, you want the no-filesort options.
3. **The column types of `amount` and `due_date`.** This matters for the generated-column definition, and for whether `due_date` has many ties (a `DATE` will; a `DATETIME` may not).
4. **Does the admin page paginate** (OFFSET or "next page")? If it does, switch to keyset pagination: `WHERE status='overdue' AND (due_date, neg_amount, id) > (?, ?, ?)`. Otherwise deep pages will get slow again even with the right index.
