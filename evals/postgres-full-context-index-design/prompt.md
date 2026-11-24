---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, WebFetch, WebSearch]
---

We're on PostgreSQL 16.4 (AWS RDS). This endpoint is our hottest read path and p95 is ~900ms, target is <50ms.

Query (prepared statement from our Go service):
SELECT id, title, price, updated_at
FROM products
WHERE tenant_id = $1 AND is_active = true AND price BETWEEN $2 AND $3
ORDER BY updated_at DESC
LIMIT 24;

Table:
CREATE TABLE products (
  id bigserial PRIMARY KEY,
  tenant_id int NOT NULL,
  title text NOT NULL,
  description text,
  price numeric(12,2) NOT NULL,
  is_active boolean NOT NULL DEFAULT true,
  updated_at timestamptz NOT NULL,
  created_at timestamptz NOT NULL
);
Existing indexes: products_pkey (id), idx_products_tenant (tenant_id), idx_products_price (price).

Data: 42M rows, ~1,800 tenants, the biggest tenant has 6M products. ~85% of rows are is_active = true. A typical price range filter matches ~30% of a tenant's active products.
Traffic: ~1,200 calls/sec at peak, the products table gets ~150 updates/sec (mostly price and updated_at) and ~20 inserts/sec.

What index(es) should we create and how do we verify it worked?
