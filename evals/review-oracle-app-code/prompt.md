---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

Oracle 19c. Our nightly export job takes 2 hours. Here's the relevant Python (python-oracledb). orders has 8M rows (about 5,000 with status NEW each night), order_items 40M rows, ext_ref is VARCHAR2(30) and indexed, order_date is DATE and indexed, order_items.order_id is indexed.

def export(conn, ref=None):
    cur = conn.cursor()
    cur.execute("SELECT * FROM orders WHERE status = 'NEW' AND TO_CHAR(order_date, 'YYYY-MM-DD') = '" + yesterday_str() + "'")
    for o in cur.fetchall():
        c2 = conn.cursor()
        c2.execute("SELECT * FROM order_items WHERE order_id = " + str(o[0]))
        write_rows(o, c2.fetchall())
    if ref:
        cur.execute("SELECT id FROM orders WHERE ext_ref = " + str(ref))   # ref is an int from the partner API

What's slowing this down and how should I rewrite it?
