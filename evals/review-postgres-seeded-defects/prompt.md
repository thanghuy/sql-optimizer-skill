---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

PostgreSQL 16. This admin report endpoint got slow as data grew (customers 4M rows, orders 60M rows, blacklist 20k rows). Can you review the query and tell me what's wrong?

Tables:
customers(id bigint PK, name text, email text, created_at timestamptz)  -- indexes: PK only, plus customers_created_idx (created_at)
orders(id bigint PK, customer_id bigint, total numeric, created_at timestamptz)  -- indexes: PK only
blacklist(customer_id bigint NULL, reason text)  -- no indexes; customer_id is sometimes NULL for blocked emails

SELECT c.id, c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count,
       (SELECT MAX(o.created_at) FROM orders o WHERE o.customer_id = c.id) AS last_order_at
FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM blacklist)
  AND lower(c.email) = lower($1)
ORDER BY c.created_at DESC
OFFSET 50000 LIMIT 20;
