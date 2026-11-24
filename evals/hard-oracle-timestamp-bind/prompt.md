---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

Oracle 19c. This query from our Python service (python-oracledb) does a full table scan even though order_date is indexed. orders has 50M rows, order_date is DATE, index IDX_ORDERS_DATE (order_date). Stats were gathered last night.

cur.execute("""
    SELECT id, customer_id, total
    FROM orders
    WHERE order_date >= :start_ts AND order_date < :end_ts
""", start_ts=datetime(2026, 11, 23), end_ts=datetime(2026, 11, 24))

DBMS_XPLAN.DISPLAY_CURSOR(sql_id, NULL, 'ALLSTATS LAST +PEEKED_BINDS') shows:
| Id | Operation         | Name   | Starts | E-Rows | A-Rows | Buffers |
|  0 | SELECT STATEMENT  |        |      1 |        |  61234 |   1.9M  |
|* 1 |  TABLE ACCESS FULL| ORDERS |      1 |   125K |  61234 |   1.9M  |
Peeked Binds: 1 - :START_TS (TIMESTAMP): 23-NOV-26 12.00.00.000000 AM ; 2 - :END_TS (TIMESTAMP): 24-NOV-26 12.00.00.000000 AM
Predicate Information:
   1 - filter((INTERNAL_FUNCTION("ORDER_DATE")>=:START_TS AND INTERNAL_FUNCTION("ORDER_DATE")<:END_TS))

The same query in SQL Developer with literal dates uses IDX_ORDERS_DATE. Should we rebuild the index or add a hint?
