---
type: llm
focus: trace
weight: 1
---

The response flags IF (SELECT COUNT(*) ...) > 0 and replaces it with IF EXISTS (SELECT 1 ...)
