---
type: llm
focus: trace
weight: 1
---

The response identifies that python datetime values are bound as TIMESTAMP and compared to a DATE column, so Oracle applies INTERNAL_FUNCTION to the column and cannot use IDX_ORDERS_DATE
