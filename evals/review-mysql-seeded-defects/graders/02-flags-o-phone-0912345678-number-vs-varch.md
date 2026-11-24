---
type: llm
focus: trace
weight: 1
---

The response flags o.phone = 0912345678 (number vs VARCHAR column) as an implicit cast of the column that prevents use of idx_phone, and fixes it by quoting the value
