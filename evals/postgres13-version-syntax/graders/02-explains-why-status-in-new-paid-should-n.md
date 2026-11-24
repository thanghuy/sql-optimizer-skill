---
type: llm
focus: trace
weight: 1
---

The response explains why status IN ('NEW','PAID') should not sit between region and created_at (multi-value IN breaks the order / forces a sort), or why a partial index removes it from the key
