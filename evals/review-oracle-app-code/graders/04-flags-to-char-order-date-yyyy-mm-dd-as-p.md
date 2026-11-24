---
type: llm
focus: trace
weight: 1
---

The response flags TO_CHAR(order_date, 'YYYY-MM-DD') = '...' as preventing use of the order_date index and rewrites it as a date range (order_date >= :d AND order_date < :d + 1)
