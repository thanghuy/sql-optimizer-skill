---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

PostgreSQL 16. This report query went from 2s to 9 minutes and I can't see why — the indexes are there and autovacuum/autoanalyze ran an hour ago. Should I add a composite index?

SELECT c.id, c.name, SUM(o.total)
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.city = 'Hanoi' AND c.country = 'VN'
GROUP BY c.id, c.name;

customers: 9M rows, indexes customers_city_idx (city), customers_country_idx (country). orders: 140M rows, index orders_customer_id_idx (customer_id).
Every customer in Hanoi is in VN. ~480k customers are in Hanoi; ~2.1M are in VN.

EXPLAIN (ANALYZE, BUFFERS) excerpt:
HashAggregate (actual time=541203..541880 rows=479812)
  -> Nested Loop (cost=1.13..2051.40 rows=112 width=40) (actual time=0.09..532011 rows=7398120 loops=1)
       -> Bitmap Heap Scan on customers c (cost=..  rows=112 ...) (actual rows=479812 loops=1)
             Recheck Cond: ((city = 'Hanoi') AND (country = 'VN'))
             -> BitmapAnd
                  -> Bitmap Index Scan on customers_city_idx (rows=480100) (actual rows=479812)
                  -> Bitmap Index Scan on customers_country_idx (rows=2100500) (actual rows=2103377)
       -> Index Scan using orders_customer_id_idx on orders o (cost=0.57..18.20 rows=15) (actual rows=15 loops=479812)
            Buffers: shared hit=1203411 read=6211040
