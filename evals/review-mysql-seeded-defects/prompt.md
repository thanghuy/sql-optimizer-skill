---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

Can you review this query for performance problems? MySQL 8.0.36, the orders table has ~30 million rows, and it runs on the customer-support screen every time an agent looks up a customer.

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  phone VARCHAR(20) NOT NULL,
  customer_email VARCHAR(255),
  coupon_code VARCHAR(32),
  note TEXT,
  payload JSON,
  total DECIMAL(12,2),
  created_at DATETIME NOT NULL,
  KEY idx_phone (phone),
  KEY idx_created (created_at),
  KEY idx_coupon (coupon_code)
);

SELECT *
FROM orders o
WHERE DATE(o.created_at) = '2026-11-01'
  AND o.phone = 0912345678
  AND (o.customer_email LIKE '%@gmail.com' OR o.coupon_code = 'BF2026')
ORDER BY RAND()
LIMIT 10;
