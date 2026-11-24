---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

Postgres 13, self-hosted, and we can't upgrade this year. This query comes from our Java service via JDBC prepared statements:

SELECT id, status, total
FROM orders
WHERE region = ? AND status IN ('NEW','PAID') AND created_at > now() - interval '1 day'
ORDER BY created_at DESC
LIMIT 50;

orders has 80M rows, 12 regions, NEW+PAID are ~3% of all rows, ~20k new orders/day per region. ~400 QPS on this query, ~300 inserts/sec, each order's status is updated 1-2 times.
Existing indexes: orders_pkey (id), idx_orders_region (region), idx_orders_created (created_at).

What index should I add, and how do I see the plan the app actually gets (not just what psql shows me)?
