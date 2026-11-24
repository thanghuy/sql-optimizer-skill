---
type: llm
focus: trace
weight: 1
---

The response fixes it by binding as DATE (e.g. cursor.setinputsizes(start_ts=oracledb.DB_TYPE_DATE, ...)) or by CAST(:start_ts AS DATE) / TO_DATE on the parameter side
