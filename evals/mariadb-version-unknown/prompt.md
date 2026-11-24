---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

Our admin "overdue invoices" page is slow on MariaDB in prod. invoices has ~12M rows.

SELECT id, customer_id, amount, due_date, status
FROM invoices
WHERE status = 'overdue'
ORDER BY due_date ASC, amount DESC
LIMIT 100;

Only index besides the PK is idx_status (status). Which index do I need?
