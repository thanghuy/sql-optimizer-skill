---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

SQL Server 2019, compat level 150. Following advice I created this index:

CREATE NONCLUSTERED INDEX IX_Tickets_Assignee_Created ON dbo.Tickets (AssigneeId, CreatedAt, Priority);

Query (runs ~300/min from the helpdesk app):
SELECT TOP (20) TicketId, Subject, Priority, CreatedAt
FROM dbo.Tickets
WHERE AssigneeId = @AssigneeId AND CreatedAt >= @Since
ORDER BY Priority DESC, CreatedAt DESC;

Tickets has 18M rows, clustered PK on TicketId. A busy agent has ~400k tickets, about 40k of them since @Since (usually 30 days ago). Subject is nvarchar(400).

Here's the actual execution plan summary I got (STATISTICS IO/TIME + actual plan):
- Top (20 rows)
- Sort (Priority DESC, CreatedAt DESC) — Actual rows 41,233 — warning: Operator used tempdb to spill data during execution with spill level 1
- Nested Loops (Inner Join)
  - Index Seek IX_Tickets_Assignee_Created, Seek Predicate: AssigneeId = @AssigneeId AND CreatedAt >= @Since, Actual rows 41,233, Estimated rows 1,850
  - Key Lookup (Clustered) PK_Tickets, executions 41,233
Table 'Tickets'. Scan count 1, logical reads 131,904
CPU time = 312 ms, elapsed time = 655 ms.

Still slow. What's wrong and what should I do next?
