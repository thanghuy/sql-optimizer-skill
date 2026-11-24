---
type: llm
focus: trace
weight: 1
---

The response offers a short-term workaround that converts the leads side so the users index can be used (e.g. CONVERT(l.email USING utf8mb3) or an explicit COLLATE) and notes its caveat (lossy for 4-byte characters / collation semantics)
