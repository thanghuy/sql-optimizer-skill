---
type: llm
focus: trace
weight: 1
---

The response flags ISNULL(u.IsDeleted, 0) = 0 as non-SARGable and rewrites it (IsDeleted = 0 OR IsDeleted IS NULL, or make the column NOT NULL)
