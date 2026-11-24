---
type: llm
focus: trace
weight: 1
---

The response flags SELECT DISTINCT over the Users-Orders join as hiding row multiplication (one row per order) and rewrites it with EXISTS / semi-join
