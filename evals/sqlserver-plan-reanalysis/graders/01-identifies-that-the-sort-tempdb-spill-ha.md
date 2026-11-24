---
type: llm
focus: trace
weight: 1
---

The response identifies that the Sort/tempdb spill happens because the index order (AssigneeId, CreatedAt, ...) does not match ORDER BY Priority DESC, CreatedAt DESC (range column before sort column)
