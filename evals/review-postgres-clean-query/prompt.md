---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

PostgreSQL 16. Can you review this query for performance problems before we ship? It's the "posts by author" feed.

posts(id bigint PK, author_id bigint NOT NULL, title text NOT NULL, body text, created_at timestamptz NOT NULL)
Index: posts_author_created_id_idx ON posts (author_id, created_at DESC, id DESC)
~25M rows, the most active author has 400k posts. ~600 QPS.

SELECT id, title, created_at
FROM posts
WHERE author_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;

EXPLAIN (ANALYZE, BUFFERS) for the biggest author:
Limit  (cost=0.56..41.20 rows=20 width=52) (actual time=0.031..0.074 rows=20 loops=1)
  Buffers: shared hit=24
  ->  Index Scan using posts_author_created_id_idx on posts  (cost=0.56..812345.10 rows=399812 width=52) (actual time=0.030..0.070 rows=20 loops=1)
        Index Cond: ((author_id = 42) AND (ROW(created_at, id) < ROW('2026-11-20 10:00:00+00'::timestamptz, 98765432)))
        Buffers: shared hit=24
Planning Time: 0.110 ms
Execution Time: 0.091 ms
