---
name: sql-optimizer
description: Evidence-driven SQL performance review and index optimization for PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, SQLite, IBM Db2, Snowflake, BigQuery and ClickHouse. Checks official docs for the user's exact version; collects schema, existing indexes, data volume and traffic; reviews the query for anti-patterns (non-SARGable predicates, implicit casts, N+1, OFFSET paging, correlated subqueries, NOT IN with NULLs) with severity and fixes; designs composite indexes with the ESR rule (Equality, Sort, Range); then asks for the real EXPLAIN / query plan and iterates. Use it whenever the user shares a slow query, asks why a query is slow, asks to review a query or ORM code for performance, asks which index to create, pastes an execution plan, or mentions full scans, filesort, timeouts or high DB CPU, even without the word "optimize".
---

# SQL Optimizer

Optimize SQL the way a senior DBA does: **never guess, always measure.** An index that is perfect for one query on 1,000 rows can be useless on 100 million rows, and an index that speeds up reads can kill a write-heavy table. So this skill collects evidence first, designs indexes with the ESR rule, and then proves the result with a real query plan.

Reply in the user's language. Keep questions short and grouped, so the user answers once instead of ten times.

## The workflow at a glance

```
0. Identify DB engine + version
1. Check official docs for that version         → references/databases/<db>.md
2. Ask for table structure + existing indexes   → assets/intake-template.md
3. Ask for data volume (rows, size, cardinality)
4. Ask for traffic (QPS, read/write ratio, peak, SLO)
5. Review the query itself for anti-patterns   → references/query-antipatterns.md
6. Design indexes with ESR + rewrite the query  → references/esr-rule.md
7. Ask for the query plan, analyze, iterate     → references/query-plan-analysis.md
```

Steps 2–4 can be asked in **one message** using the intake template. Step 5 needs only the SQL text, so do it right away in the same reply instead of waiting for the answers — the user gets useful findings immediately, and the schema answers later confirm or refine them. Do not block on perfect answers: if the user does not know a number, give them the exact command to get it (every database file has those commands), and continue with clearly stated assumptions in the meantime.

---

## Step 0 — Identify the engine and its exact version

Always ask which database **and which exact version** the user runs, unless they already said it (e.g. "PostgreSQL 16.4", "MySQL 8.0.36", "SQL Server 2019, compat level 150"). Make it the first question in your message, and give the one-line command to get it (from the engine's reference file: `SELECT version();`, `SELECT VERSION();`, `SELECT @@VERSION`, `SELECT banner_full FROM v$version`, `db2level`, …). Also ask whether it is a managed flavor (RDS, Aurora, Azure SQL, Cloud SQL, ClickHouse Cloud, Snowflake edition), since some features and knobs are not available there.

The version drives two things, so get it right:

1. **Which docs you read.** Open the doc pages for that version (e.g. `postgresql.org/docs/16/`, `dev.mysql.com/doc/refman/8.0/`, Oracle `/19/` vs `/23/`), not "current" or "latest", which may describe features the user does not have.
2. **What you generate.** Every DDL, query rewrite and EXPLAIN command must be valid on that version. Examples of things that differ:
   - PostgreSQL: `INCLUDE` needs 11+, `EXPLAIN (GENERIC_PLAN)` 16+, skip scan only 18+.
   - MySQL: descending indexes, functional indexes, invisible indexes need 8.0+ (functional 8.0.13+); `EXPLAIN ANALYZE` 8.0.18+; `EXPLAIN FORMAT=TREE` 8.0.16+.
   - MariaDB: descending indexes 10.8+, `IGNORED` indexes 10.6+, a new cost model in 11.0.
   - SQL Server: `RESUMABLE` online index builds 2017+ (CREATE INDEX 2019+), PSP optimization 2022 (compat level 160); compat level matters as much as version.
   - Oracle: `FETCH FIRST n ROWS` 12c+; automatic indexing 19c+ on specific platforms.

If the version is still unknown when you must answer (single-turn, or the user wants a quick answer), state the version you assume in one line, use only syntax that works on the oldest supported version, and mark any version-dependent recommendation with its minimum version (e.g. "needs MySQL 8.0.13+"). Swap to the right syntax once the user tells you the version.

Supported engines and their reference file:

| Engine | File | Index model |
|---|---|---|
| PostgreSQL | `references/databases/postgresql.md` | B-tree, partial, expression, GIN/GiST/BRIN, INCLUDE |
| MySQL | `references/databases/mysql.md` | InnoDB clustered PK + secondary B-tree |
| MariaDB | `references/databases/mariadb.md` | InnoDB/Aria B-tree, like MySQL with differences |
| SQL Server | `references/databases/sqlserver.md` | Clustered / nonclustered + INCLUDE, filtered, columnstore |
| Oracle | `references/databases/oracle.md` | B-tree, bitmap, function-based, IOT, partitioning |
| SQLite | `references/databases/sqlite.md` | B-tree, partial, expression, rowid tables |
| IBM Db2 | `references/databases/db2.md` | B-tree, clustering, INCLUDE (unique), MDC, column-organized |
| Snowflake | `references/databases/snowflake.md` | No B-tree: micro-partitions, clustering keys, search optimization |
| BigQuery | `references/databases/bigquery.md` | No B-tree: partitioning, clustering, search indexes |
| ClickHouse | `references/databases/clickhouse.md` | Sparse primary key (ORDER BY), skip indexes, projections |

If the engine is not in this list (e.g. CockroachDB, TiDB, Redshift, Aurora DSQL), map it to the closest one (CockroachDB/TiDB → PostgreSQL/MySQL-like B-tree; Redshift → Snowflake/ClickHouse-like sort keys), say so, and look up its docs the same way.

## Step 1 — Check the official docs

Read the engine's reference file first. It lists the official doc pages that matter for indexing and query plans. Then, **if you have a web tool** (WebFetch, WebSearch, browser, MCP fetch), fetch the 2–3 pages relevant to this specific problem **for the user's exact version** (replace the version segment in the URL, e.g. `/docs/current/` → `/docs/16/`, `/refman/8.4/` → `/refman/8.0/`), for example the multicolumn index page plus the EXPLAIN output page. Fetching is worth it because optimizer behavior changes between versions and your training data may be stale.

- If a URL has moved, search `site:<vendor domain> <topic> <version>`.
- Quote the doc briefly and give the link when a recommendation depends on version-specific behavior.
- If you have no web access, say so in one line and continue from the reference file and your own knowledge, flagging anything version-sensitive.

Do not dump the docs on the user. The docs check is for *your* accuracy.

## Step 2 — Table structure and existing indexes

You need, for every table in the query:
- `CREATE TABLE` (column types, nullability, PK, FKs, partitioning)
- every existing index, including column order, sort direction, INCLUDE columns, partial/filter conditions, and uniqueness
- the full query exactly as the application runs it, with realistic parameter values (and whether it is a prepared statement / ORM generated)

Each database file has copy-paste commands to extract this. Why this matters: the most common "fix" — adding a new index — is often wrong because an existing index can be extended, or because a redundant index already costs write performance.

## Step 3 — Data volume

Ask for:
- row count and on-disk size per table (approximate is fine)
- **cardinality / selectivity** of every column in WHERE, JOIN, ORDER BY, GROUP BY: number of distinct values, and how many rows a typical parameter value matches
- data skew: is there a value that matches 40% of the rows (e.g. `status = 'done'`)?
- growth rate (rows per day) and retention

Why: the optimizer chooses by estimated rows. Selectivity decides whether an index is useful at all, which range is selective enough to go before the sort column, and whether stale statistics are the real problem. Rough rules: tables under ~10k rows rarely need tuning; a predicate that matches more than roughly 5–20% of a table usually will not use a B-tree index for row lookups (a covering index can still help).

## Step 4 — Traffic

Ask for:
- how often this query runs (QPS / per minute) at normal and peak
- current latency (p50/p95/p99) and the target (SLO)
- read/write ratio of the tables involved: inserts/updates/deletes per second
- whether it is OLTP (many small queries) or analytical/reporting (few big ones), and whether it runs on a replica

Why: traffic decides the cost/benefit. A query run 2,000 times per second deserves a dedicated covering index; a nightly report usually does not. Every extra index slows every INSERT and every UPDATE of indexed columns, uses memory, and adds lock/replication work, so on a write-heavy table prefer extending an existing index or rewriting the query over adding a new one. Each database file shows where to read real numbers (pg_stat_statements, performance_schema, Query Store, AWR, query history…).

## Step 5 — Review the query for anti-patterns

Before thinking about indexes, read the SQL (and any application code around it) clause by clause against `references/query-antipatterns.md`. Many slow queries are not missing an index — they make the index unusable (`DATE(col) = ?`, `varchar_col = 123`, `ISNULL(col,0) = 0`), multiply work (correlated subqueries, N+1 loops, `DISTINCT` over a fan-out join), or throw work away (`OFFSET 50000`, `ORDER BY RAND()`, `SELECT *`). An index cannot fix those; a rewrite can.

Report every finding in the **Issues found** table (see Output format), most severe first, each with the quoted fragment, the mechanism, and the concrete fix. Flag correctness bugs found along the way (e.g. `NOT IN` with a nullable column returns no rows; a number compared to a zero-padded VARCHAR) — they matter more than speed.

If the SQL text looks clean but the user says it is slow (or slow only in the app, only sometimes, or after a data load), go to section 9b of that file: parameter sniffing, driver bind types, collation mismatches and bad statistics do not show in the text, and a new index will not fix them. When the user asks to drop "unused" indexes, check constraints, foreign keys, stats reset time and replicas first (`esr-rule.md` §7).

Every row in the Issues table gets an explicit severity label (High / Medium / Low), not just an order. Be precise rather than exhaustive. Check section 10 of that file ("things that look bad but are fine") before reporting, and if the query is already well written, say so plainly instead of inventing problems.

## Step 6 — Design the index with ESR, and rewrite the query

Read `references/esr-rule.md` before proposing any composite index. Short version, for B-tree engines:

1. **E — Equality** columns first (`col = ?`, `col IS NULL`, and a single-value `IN`).
2. **S — Sort** columns next, in the same order and direction as ORDER BY (GROUP BY counts as sort).
3. **R — Range** columns last (`>`, `<`, `BETWEEN`, `LIKE 'abc%'`, multi-value `IN`, `<>`).
4. Optionally add the selected columns (INCLUDE or trailing columns) to make it **covering**.

Then check the exceptions in `esr-rule.md` (very selective range → E-R-S; multi-value IN breaks sort order; leftmost-prefix reuse across queries; mixed ASC/DESC; functions or implicit casts on indexed columns). For Snowflake, BigQuery and ClickHouse, translate ESR into partitioning/clustering/sort-key order as explained in that file — there is no B-tree to create.

Design the index for the **rewritten** query from Step 5, not the original one (e.g. after turning `DATE(created_at) = ?` into a range, `created_at` becomes an R column).

Present each recommendation as:
- the DDL (with the online / non-blocking option for that engine, e.g. `CREATE INDEX CONCURRENTLY`, `ONLINE = ON`, `ALGORITHM=INPLACE, LOCK=NONE`)
- which ESR role each column plays and why
- what it costs (write overhead, size estimate, build time on a big table)
- which existing index it replaces or makes redundant
- how to roll it back (`DROP INDEX`, or test first with an invisible/hypothetical index where supported)

Before sending, run the checklist at the end of `esr-rule.md` against your final answer. Two mistakes are easy to make here: you rewrite the query (adding a keyset tie-breaker, changing ORDER BY) but forget to update the index to match, or you use a DDL/EXPLAIN option the user's version does not have.

## Step 7 — Ask for the query plan and re-analyze

After the user creates the index (ideally on staging with production-like data), ask for the **actual** plan, not only the estimated one, using the exact command from the database file (e.g. `EXPLAIN (ANALYZE, BUFFERS)`, `EXPLAIN ANALYZE`, actual execution plan + `SET STATISTICS IO ON`, `DBMS_XPLAN.DISPLAY_CURSOR(format => 'ALLSTATS LAST')`). Warn when the command actually executes the statement (EXPLAIN ANALYZE on an UPDATE/DELETE changes data — wrap it in a transaction and roll back).

Analyze with `references/query-plan-analysis.md`:
- Is the new index used? For seek/range, or only scanned?
- Estimated vs actual rows — a gap of 10× or more means statistics problems, not index problems.
- Is there still a sort / filesort / temp table? Then the S part of ESR is not satisfied.
- Rows read vs rows returned — a high ratio means a column is in the wrong position or missing.
- Key lookups / table access by rowid per row — consider making the index covering.
- Compare before vs after: time, buffers/logical reads, rows examined.

If the plan is still not good, explain why, adjust (column order, covering columns, statistics, query rewrite), and ask for the plan again. Stop when the target SLO is met or when the remaining cost is inherent (e.g. the query really must read 30% of the table — then suggest summary tables, caching, or moving it to a replica/warehouse).

## Output format

When giving the final recommendation, use this structure:

```
## Diagnosis
What is slow and why (evidence: plan line, row counts, traffic)

## Issues found
| # | Where (quoted fragment) | Problem | Severity | Why it is slow | Fix |
|---|---|---|---|---|---|
(or one line: "No significant issues in the query text" with the reason)

## Rewritten query
The full query with all fixes applied, version-valid for the user's engine

## Recommendation
1. Index / query change — DDL or rewritten SQL
   ESR mapping: E = …, S = …, R = …
   Expected effect: …
   Cost: …   Rollback: …

## Verify
Exact command to get the new plan, and what a good plan should look like

## Assumptions / open questions
Anything you assumed because data was missing
```

## Safety

- Never run DDL or `EXPLAIN ANALYZE` on production yourself without the user's explicit approval; prefer giving the command for the user to run.
- Recommend building indexes online and outside peak hours on large tables; mention disk space (an index build can need roughly the size of the index again in temp space).
- Do not recommend optimizer hints as the first fix; they freeze a plan that may become wrong as data changes. Use them only as a last resort or a diagnostic.
- Do not ask users to paste real customer data; schemas, counts and plans are enough.
