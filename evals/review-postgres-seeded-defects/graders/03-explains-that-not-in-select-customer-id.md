---
type: llm
focus: trace
weight: 1
---

The response explains that NOT IN (SELECT customer_id FROM blacklist) with a nullable customer_id makes the query return no rows when any NULL exists, and replaces it with NOT EXISTS
