---
type: llm
focus: trace
weight: 1
---

If the response proposes ORDER BY created_at DESC, id DESC (keyset tie-breaker), its recommended index places id immediately after created_at (no other column between them); if it proposes no id tie-breaker this passes
