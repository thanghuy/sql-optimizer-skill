---
type: llm
focus: trace
weight: 1
---

The response flags the two correlated subqueries on orders as executing per customer row and rewrites them as a single aggregation (join to a GROUP BY subquery, or LATERAL)
