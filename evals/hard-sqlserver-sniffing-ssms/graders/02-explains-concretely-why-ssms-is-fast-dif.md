---
type: llm
focus: trace
weight: 1
---

The response explains concretely why SSMS is fast: different SET options (ARITHABORT) give a separate plan cache entry, and/or the DECLARE local-variable test uses density estimates instead of sniffing
