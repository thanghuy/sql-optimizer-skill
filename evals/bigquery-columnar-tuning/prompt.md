---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

our bigquery bill exploded, this dashboard query scans ~2.3 TB every time it runs (looker refreshes it every 15 min). table is `analytics.events`, ~9 billion rows, not partitioned, not clustered. columns: event_id STRING, event_ts TIMESTAMP, project_id STRING, user_id STRING, event_name STRING, country STRING, properties JSON.

SELECT event_name, COUNT(*) AS n, COUNT(DISTINCT user_id) AS users
FROM analytics.events
WHERE project_id = 'prj_8812'
  AND DATE(event_ts) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) AND CURRENT_DATE()
GROUP BY event_name
ORDER BY n DESC;

we have ~300 projects, almost every query filters on project_id and a time window. what index should i add?
