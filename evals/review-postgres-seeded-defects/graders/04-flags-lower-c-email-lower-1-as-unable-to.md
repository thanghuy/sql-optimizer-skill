---
type: llm
focus: trace
weight: 1
---

The response flags lower(c.email) = lower($1) as unable to use an index without an expression index and recommends CREATE INDEX ... (lower(email)) or citext
