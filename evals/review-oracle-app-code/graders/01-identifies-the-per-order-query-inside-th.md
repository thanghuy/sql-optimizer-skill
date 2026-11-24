---
type: llm
focus: trace
weight: 1
---

The response identifies the per-order query inside the loop as an N+1 pattern (about 5,000 extra round trips) and replaces it with a single join or a batched IN / array query
