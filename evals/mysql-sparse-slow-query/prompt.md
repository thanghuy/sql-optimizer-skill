---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

this query takes 4-5s on prod (mysql), my boss wants it fixed asap. what index do I need to make it fast?

SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207 AND status IN ('paid','shipped') AND created_at >= '2026-10-01'
ORDER BY created_at DESC
LIMIT 50 OFFSET 2000;
