---
type: llm
focus: trace
weight: 1
---

The response gives a PostgreSQL 13-valid way to see the app's real plan: PREPARE/EXECUTE 6+ times, SET plan_cache_mode = force_generic_plan, and/or auto_explain with log_analyze
