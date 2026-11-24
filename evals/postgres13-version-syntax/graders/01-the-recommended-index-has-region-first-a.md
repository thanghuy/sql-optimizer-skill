---
type: llm
focus: trace
weight: 1
---

The recommended index has region first and created_at (DESC) right after it, with status handled by a partial index predicate (WHERE status IN ('NEW','PAID')) or placed after created_at
