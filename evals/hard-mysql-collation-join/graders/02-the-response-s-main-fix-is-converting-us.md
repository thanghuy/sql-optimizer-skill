---
type: llm
focus: trace
weight: 1
---

The response's main fix is converting users.email to the same charset and collation as leads.email (ALTER TABLE ... MODIFY / CONVERT TO), and it says another index on users.email would not help
