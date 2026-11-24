Short answer: **don't do either.** A second index on `users.email` would be ignored for the same reason `uq_users_email` is ignored. A subquery (`IN (SELECT ...)` / `EXISTS`) runs the same comparison with the same conversion, so the plan stays the same. The cause is that the two `email` columns use **different character sets and collations**. Fix that and your existing unique index starts working.

(Target: MySQL 8.0.36 as you said. I checked the 8.0 manual pages on collation coercibility and online DDL.)

## Diagnosis

- `users.email` is `utf8mb3` / `utf8mb3_general_ci`. `leads.email` is `utf8mb4` / `utf8mb4_0900_ai_ci`.
- To compare them, MySQL has to pick one character set and one collation. utf8mb4 is the superset, so MySQL converts the **users** side. The join condition is really evaluated as
  `convert(u.email using utf8mb4) = l.email` under `utf8mb4_0900_ai_ci`.
- The B-tree `uq_users_email` is sorted by `utf8mb3_general_ci`, not by "converted to utf8mb4, compared with 0900_ai_ci". So the optimizer can't look up a lead's email in it. Your EXPLAIN shows exactly this: `possible_keys = NULL` on `u`, `type = ALL` over 5,981,220 rows, and a hash join (8.0.18+) that has to read all of `users` on every run.
- The `leads` side is fine: `created_at > NOW() - INTERVAL 7 DAY` keeps the arithmetic on the parameter side, so it's a proper range on `idx_leads_created` (~198k rows).

You can see the conversion directly:
```sql
EXPLAIN SELECT ...;   -- your query
SHOW WARNINGS\G       -- the rewritten statement shows (convert(`u`.`email` using utf8mb4) = `l`.`email`)
-- or
EXPLAIN FORMAT=TREE SELECT ...;   -- shows the hash join condition with convert(...)
```

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
| 1 | `JOIN users u ON u.email = l.email` | Charset/collation mismatch between join columns (utf8mb3_general_ci vs utf8mb4_0900_ai_ci) | **High** | `users.email` is converted on every row, so `uq_users_email` can't be used for lookups. You get a full scan of 6M rows plus a hash join on each execution, and the cost grows with `users` | Make the column definitions identical (Recommendation 1). Until then, convert the **leads** side in the query instead (Recommendation 2) |
| 2 | (same) | The match rule may not be what you think. The join currently compares with `utf8mb4_0900_ai_ci` (NO PAD, UCA 9.0 weights). Your uniqueness rule is `utf8mb3_general_ci` (PAD SPACE, older weights). These collations disagree on some values, e.g. trailing spaces and characters like `ß` | Low (correctness) | Not a speed issue. It means the "same email" test in this join is not the same test your unique key enforces | Choose one collation for emails everywhere (Recommendation 1 does this) |
| 3 | `SELECT l.id, l.email ... WHERE l.created_at > ...` | `idx_leads_created` isn't covering, so each of the ~198k leads needs a clustered-index lookup to read `email` | Low / Medium (depends on how often this runs) | About 198k random PK lookups per run. Small next to issue 1, but it's the next cost once issue 1 is fixed | Optional: extend the index to `(created_at, email)` (Recommendation 3) |

The query has no other problems: no function on an indexed column, no fan-out (`users.email` is unique), and no `SELECT *`.

## Rewritten query

**Use this until the schema is fixed.** It converts the small side (the ~198k leads) into `users.email`'s own charset and collation. The users column is then compared unchanged, and `uq_users_email` becomes an `eq_ref` lookup:

```sql
SELECT l.id, l.email, u.id AS user_id
FROM leads l
JOIN users u
  ON u.email = CONVERT(l.email USING utf8mb3) COLLATE utf8mb3_general_ci
WHERE l.created_at > NOW() - INTERVAL 7 DAY;
```

Caveats:
- `COLLATE` has coercibility 0 (explicit), so it wins and the users side is left untouched.
- Any 4-byte character in `leads.email` (e.g. an emoji) can't be represented in utf8mb3 and becomes `?`. In practice that row just won't match.
- Matching now follows `utf8mb3_general_ci` rules, the same rules your unique key enforces. Results can differ from today's query in the edge cases from issue 2. Usually that's the behavior you actually want.
- `utf8mb3` is deprecated in 8.0, so you'll see deprecation warnings. That's another reason to treat this as a stopgap.

The schema fix below is the real solution. After it, the original query works unchanged.

## Recommendation

### 1. Fix the column definition (the real fix)

Make `users.email` the same charset and collation as `leads.email`:

```sql
-- 8.0: changing a column's character set is ALGORITHM=COPY: full table rebuild, writes blocked for the duration.
ALTER TABLE users
  MODIFY email VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL;
```

- **ESR mapping:** E = `users.email` (the inner-table join column acts as equality for each lead). This uses the existing `uq_users_email`, so no new index is needed.
- **Expected effect:** `u` goes from `ALL` (6M rows) to `eq_ref` on `uq_users_email`, about 198k index lookups. Runtime should drop from "scan all of users" to roughly the cost of reading one week of leads.
- **How to run it:** On 6M rows, a plain `ALTER` blocks writes to `users` (it can't run with `LOCK=NONE`) for minutes or more. Use **gh-ost** or **pt-online-schema-change** off-peak, and watch replication lag. You need free disk space about the size of the table plus its indexes.
- **Check before you run it:**
  1. **Duplicates under the new collation.** The unique key is rebuilt with `utf8mb4_0900_ai_ci` rules. Run this on a replica, since it's a heavy query:
     ```sql
     SELECT CONVERT(email USING utf8mb4) COLLATE utf8mb4_0900_ai_ci AS e, COUNT(*)
     FROM users GROUP BY e HAVING COUNT(*) > 1;
     ```
     Any rows returned would make the ALTER fail. Under NO PAD, values like `'a@x.com'` and `'a@x.com '` also become distinct, so uniqueness gets slightly *looser* for them.
  2. **Index size limit.** An index on utf8mb4 `VARCHAR(255)` is up to 1020 bytes. That's fine with `ROW_FORMAT=DYNAMIC`/`COMPRESSED` (3072-byte limit, the 8.0 default). It fails on `COMPACT`/`REDUNDANT` (767-byte limit). Check with `SELECT row_format FROM information_schema.tables WHERE table_name='users';`.
  3. **Other dependents.** Look for foreign keys that reference `users.email` (FK columns must have matching charsets) and other queries or tables that join on it. Better yet, standardize every email column on `utf8mb4_0900_ai_ci` (or one chosen collation) so this doesn't come back on the next join.
  4. **Bigger index.** `uq_users_email` grows (up to 4 bytes per character instead of 3). For ASCII emails the actual stored size barely changes.
- **Cost:** a one-time rebuild. No extra write overhead afterwards.
- **Rollback:** `MODIFY` back to `utf8mb3 COLLATE utf8mb3_general_ci`, which is another rebuild. With gh-ost/pt-osc you can keep the old table until you've checked the result. Test the whole thing on staging first.

Alternative: convert `leads.email` *down* to utf8mb3_general_ci instead. That's a smaller table, but I don't recommend it, because utf8mb3 is deprecated and can't store 4-byte characters.

### 2. Stopgap with no DDL

Ship the rewritten query above now. Expected plan: `l` range on `idx_leads_created`, then `u` **eq_ref** on `uq_users_email`, with no hash join. Rollback: revert the query text.

### 3. Optional: make the leads side covering

Only worth it if this query is frequent and step 1 or 2 leaves you short of your target:

```sql
ALTER TABLE leads
  ADD INDEX idx_leads_created_email (created_at, email),
  ALGORITHM=INPLACE, LOCK=NONE;
-- then, once confirmed, drop the old one (it's a leftmost prefix of the new index):
ALTER TABLE leads ALTER INDEX idx_leads_created INVISIBLE;   -- test first (8.0+)
ALTER TABLE leads DROP INDEX idx_leads_created;
```

- **ESR mapping:** R = `created_at`, then `email` as a trailing covering column. `id` is the InnoDB PK and is already appended to every secondary index, so don't add it.
- **Effect:** `Using index` on `l`, which removes ~198k clustered-index lookups.
- **Cost:** a wider index (up to ~1 KB of key per row in the worst case, much less for real emails). Every insert into leads (~200k a week, ~0.3/s on average) pays a little more. That's cheap here.

## Verify

Run this on staging with production-like data, or on a replica. `EXPLAIN ANALYZE` (8.0.18+) **executes** the SELECT, which is read-only and safe, but it does use resources:

```sql
EXPLAIN FORMAT=TREE  <query>;
EXPLAIN ANALYZE      <query>;
FLUSH STATUS; <query>; SHOW SESSION STATUS LIKE 'Handler_read%';
```

A good plan looks like this:
- A nested loop: index range scan on `l` using `idx_leads_created` (or `idx_leads_created_email` with "covering index"), then a single-row index lookup on `u` using `uq_users_email`.
- No `Using join buffer (hash join)`, and no `convert(... using utf8mb4)` on `u.email` in `SHOW WARNINGS`.
- Actual rows on `u` ≈ number of matched leads (≤ ~198k), not ~6M.
- `Handler_read_rnd_next` close to 0; `Handler_read_key` ≈ number of leads.

Send me the before and after `EXPLAIN ANALYZE` output and I'll check it.

## Assumptions / open questions

These would change the details, not the diagnosis. Answer them if you can:
1. **Managed service?** On RDS/Aurora, gh-ost needs extra settings. Aurora also has its own options.
2. **Traffic:** How often does this query run, what's its current latency, and what's your target? How write-heavy is `users`? This decides how carefully the rebuild has to be scheduled, and whether index 3 is worth adding.
3. **Other email columns:** Do other tables or queries join on `users.email` (or have FKs to it)? Which collation do *they* use? Decide which collation is the standard before changing anything.
4. **Emoji/4-byte characters:** Can `leads.email` contain them (it accepts them today)? This only matters for the stopgap query.
5. **Row format:** Is `users` on `ROW_FORMAT=DYNAMIC`? (See check 2.)
