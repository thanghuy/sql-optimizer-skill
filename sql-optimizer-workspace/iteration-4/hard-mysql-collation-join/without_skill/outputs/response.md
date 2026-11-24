**Short answer: don't do either.** Another index on `users.email` won't be used, for the same reason the current one isn't, and a subquery won't help either. The real problem is that the two `email` columns use **different character sets and collations**.

## What's going on

| column | charset | collation |
|---|---|---|
| `users.email` | utf8mb3 | utf8mb3_general_ci |
| `leads.email` | utf8mb4 | utf8mb4_0900_ai_ci |

MySQL can't compare those two directly. utf8mb4 is a superset of utf8mb3, so it converts the **utf8mb3 side (users)** to utf8mb4 and compares using `utf8mb4_0900_ai_ci`. Your join condition actually runs as:

```sql
ON CONVERT(u.email USING utf8mb4) = l.email   -- compared with utf8mb4_0900_ai_ci
```

`uq_users_email` is sorted by `utf8mb3_general_ci` values, not by `CONVERT(email USING utf8mb4)` under `utf8mb4_0900_ai_ci`. Once a function wraps the indexed column, the index can't be used for lookups. That's why EXPLAIN shows `possible_keys = NULL`, `type = ALL` and a hash join over all ~6M users. You can see the conversion yourself:

```sql
EXPLAIN FORMAT=TREE
SELECT l.id, l.email, u.id FROM leads l JOIN users u ON u.email = l.email
WHERE l.created_at > NOW() - INTERVAL 7 DAY;
-- or: EXPLAIN ... ; SHOW WARNINGS;   -- look for convert(`u`.`email` using utf8mb4)
```

**Why your two ideas don't help:**
- **Another index on `users.email`:** it would have the same collation, so the same thing happens. You'd just get a second unused 6M-row index and slower writes.
- **Subquery (`WHERE l.email IN (SELECT email FROM users)` or `EXISTS`):** the comparison still mixes the two collations, so the optimizer is stuck the same way. It also changes the result shape, since you need `u.id`.

## Fix 1: rewrite the query (works now, no DDL)

Convert the **leads** side to match the users column, so the indexed column stays untouched:

```sql
SELECT l.id, l.email, u.id AS user_id
FROM leads l
JOIN users u
  ON u.email = CONVERT(l.email USING utf8mb3) COLLATE utf8mb3_general_ci
WHERE l.created_at > NOW() - INTERVAL 7 DAY;
```

Now `u` should show `type = eq_ref` with `key = uq_users_email`. The cost drops from "scan 6M rows" to "about 200k unique index lookups". Check this with `EXPLAIN` / `EXPLAIN ANALYZE` before you ship it.

Caveats you should understand before relying on it:
1. **Matching rules change slightly.** Matches now follow `utf8mb3_general_ci` (the rules of the users unique key) instead of `utf8mb4_0900_ai_ci`. Both ignore case and accents for normal emails. They differ in edge cases: general_ci is PAD SPACE, so trailing spaces are ignored, while 0900_ai_ci is NO PAD. Some special characters also behave differently, such as `ß` vs `ss`. For ASCII emails the results are the same.
2. **4-byte characters are lost.** Any `leads.email` with a 4-byte character (emoji, some CJK) becomes `?` when converted to utf8mb3. It won't match the right user, and in theory it could match a user whose email has a literal `?`. Check whether you have any:
   ```sql
   SELECT COUNT(*) FROM leads
   WHERE CONVERT(CONVERT(email USING utf8mb3) USING utf8mb4) <> email;
   ```
   No user can have such an email anyway, because `users` is utf8mb3. So the only risk is the false match on a literal `?`.
3. **It's fragile.** Every query that joins these tables needs the same trick. Your ORM or other developers will keep writing the plain `=` and hitting the same slow plan.

## Fix 2 (the right long-term fix): give both columns the same charset and collation

`utf8mb3` is deprecated in MySQL 8.0, so the usual move is to convert `users.email` (ideally the whole `users` table) to utf8mb4 with the same collation as `leads`:

```sql
ALTER TABLE users
  MODIFY email VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL;
-- or the whole table:
-- ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

After that, the original query works as written and uses `uq_users_email` via `eq_ref`.

Things to plan for:
- **The table gets rebuilt.** A charset change can't be done in place: it's `ALGORITHM=COPY` and blocks writes while it runs. On 6M rows in production, use **gh-ost** or **pt-online-schema-change**, or run it in a maintenance window.
- **The unique key could break.** Collation decides which values count as "the same", and `utf8mb4_0900_ai_ci` differs from `utf8mb3_general_ci` (padding, `ß`, some accents). Two emails that were distinct before could become duplicates, and the ALTER would fail partway through. Check first:
  ```sql
  SELECT CONVERT(email USING utf8mb4) COLLATE utf8mb4_0900_ai_ci AS e, COUNT(*)
  FROM users
  GROUP BY e HAVING COUNT(*) > 1;
  ```
  (This query scans the full table. Run it on a replica.)
- **Index size is fine.** A utf8mb4 `VARCHAR(255)` key is up to 1020 bytes, well under InnoDB's 3072-byte limit for DYNAMIC/COMPRESSED row formats, which are the 8.0 default.
- **Check other dependencies.** Look for other tables or columns that join to `users.email`, plus stored procedures, triggers and app connection charset settings. Anything still on utf8mb3 will now have the mismatch in the other direction.
- **The other way round is worse.** Don't convert `leads` down to utf8mb3: that's the deprecated charset, and it can't store 4-byte characters.

## Optional extra on the leads side

Once the join uses the index, the next cost is reading about 200k leads rows. A covering index lets the range scan skip going back to the table rows (InnoDB adds the PK `id` to secondary indexes automatically):

```sql
ALTER TABLE leads ADD INDEX idx_leads_created_email (created_at, email);
-- then idx_leads_created (created_at) becomes redundant and can be dropped later
```

This is a much smaller gain than the collation fix, so only bother if the query still isn't fast enough.

## Recommendation

1. **Today:** ship the `CONVERT(... USING utf8mb3) COLLATE utf8mb3_general_ci` rewrite, after running the 4-byte-character check. Confirm with `EXPLAIN ANALYZE` that `u` is now `eq_ref` on `uq_users_email`.
2. **Next:** schedule an online migration of `users` to `utf8mb4` / `utf8mb4_0900_ai_ci`, after the duplicate check. Then drop the `CONVERT` from the query.
3. **Don't** add another index on `users.email`, and don't rewrite as a subquery. Neither fixes the cause.

One thing I can't tell from what you sent: whether `users` has other utf8mb3 columns that other utf8mb4 tables join on. If so, the same problem is happening in other queries, which is another reason to convert the whole table.
