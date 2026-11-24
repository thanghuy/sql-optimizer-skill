---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

SQL Server 2019 Standard, compat level 150. This proc is called from our .NET API and sometimes takes 20+ seconds, but when I copy it into SSMS it runs in under a second, so I can't reproduce it. Do we need another index?

CREATE PROCEDURE dbo.GetTicketsByStatus @Status varchar(20)
AS
SELECT TicketId, Subject, CreatedAt
FROM dbo.Tickets
WHERE Status = @Status
ORDER BY CreatedAt DESC;

Index: IX_Tickets_Status_CreatedAt ON dbo.Tickets (Status, CreatedAt DESC) INCLUDE (Subject). Tickets has 12M rows: 'Closed' ~11.4M, 'Open' ~9,000, 'Pending' ~600k.
The API calls it ~200 times/min, almost always with 'Open', but a nightly export calls it once with 'Closed'.

How I tested in SSMS:
DECLARE @s varchar(20) = 'Open';
EXEC dbo.GetTicketsByStatus @s;

From the plan cache (the slow one used by the API): Estimated rows 11,402,113, Actual rows 9,041, ParameterCompiledValue = 'Closed', ParameterRuntimeValue = 'Open'. The plan is a Clustered Index Scan + Sort with a large memory grant.
