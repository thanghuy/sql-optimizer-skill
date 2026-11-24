---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

PostgreSQL 15 on RDS with one read replica that serves the reporting dashboards. Our write latency on orders-related tables is bad, so I want to drop unused indexes. I ran this on the primary:

SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) FROM pg_stat_user_indexes ORDER BY idx_scan;

 relname      | indexrelname                     | idx_scan | size
--------------+----------------------------------+----------+-------
 payments     | idx_payments_order_id            |        0 | 2.1 GB
 users        | users_email_key                  |        3 | 410 MB
 order_items  | idx_order_items_order_id         |       12 | 3.4 GB
 orders       | idx_orders_created_at            |       40 | 1.2 GB
 order_items  | idx_order_items_order_product    |   981233 | 4.0 GB

Definitions:
- idx_payments_order_id ON payments (order_id); payments.order_id REFERENCES orders(id) ON DELETE CASCADE. No other index on payments starts with order_id. We purge old orders every night.
- users_email_key is the UNIQUE constraint on users.email.
- idx_order_items_order_id ON order_items (order_id)
- idx_order_items_order_product ON order_items (order_id, product_id)
- idx_orders_created_at ON orders (created_at)

pg_stat_database.stats_reset for this DB was 3 days ago (we upgraded the instance).

Which of these can I drop? Please give me the DROP statements.
