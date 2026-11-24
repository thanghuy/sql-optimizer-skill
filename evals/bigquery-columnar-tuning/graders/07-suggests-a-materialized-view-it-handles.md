---
type: llm
focus: trace
weight: 1
---

If the response suggests a materialized view, it handles COUNT(DISTINCT) correctly (HLL/APPROX functions or a non-incremental alternative) rather than proposing an exact COUNT(DISTINCT) incremental MV; passes if no MV is proposed
