---
type: llm
focus: trace
weight: 1
---

The response identifies the charset/collation mismatch (users.email utf8mb3_general_ci vs leads.email utf8mb4_0900_ai_ci) as the reason uq_users_email cannot be used for the join
