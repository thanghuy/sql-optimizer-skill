# Snowflake

Snowflake has **no user-created B-tree indexes** on standard tables. Performance comes from micro-partition pruning, clustering, warehouse size, caching, and (optionally) Search Optimization or materialized views. Translate ESR into clustering-key order (see `esr-rule.md` §9). Hybrid tables (Unistore) are the exception: they do have primary keys and secondary indexes, so ESR applies directly to them.

## Official docs to check
- Micro-partitions and data clustering: https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
- Clustering keys and clustered tables: https://docs.snowflake.com/en/user-guide/tables-clustering-keys
- Automatic clustering: https://docs.snowflake.com/en/user-guide/tables-auto-reclustering
- Search optimization service: https://docs.snowflake.com/en/user-guide/search-optimization-service
- Query profile: https://docs.snowflake.com/en/user-guide/ui-query-profile
- EXPLAIN: https://docs.snowflake.com/en/sql-reference/sql/explain
- GET_QUERY_OPERATOR_STATS: https://docs.snowflake.com/en/sql-reference/functions/get_query_operator_stats
- Query acceleration service: https://docs.snowflake.com/en/user-guide/query-acceleration-service
- Hybrid tables and indexes: https://docs.snowflake.com/en/user-guide/tables-hybrid
- Optimizing warehouses for performance: https://docs.snowflake.com/en/user-guide/performance-query-warehouse

## Version / edition
```sql
SELECT CURRENT_VERSION();
SHOW WAREHOUSES;   -- size, multi-cluster settings
```
Edition matters: Search Optimization, materialized views and Query Acceleration need Enterprise or higher.

## Table structure
```sql
SELECT GET_DDL('TABLE', 'DB.SCHEMA.ORDERS');
SHOW TABLES LIKE 'ORDERS';          -- cluster_by, rows, bytes, automatic_clustering
DESCRIBE SEARCH OPTIMIZATION ON orders;
```

## Data volume and clustering quality
```sql
SELECT row_count, bytes FROM information_schema.tables WHERE table_name = 'ORDERS';
SELECT SYSTEM$CLUSTERING_INFORMATION('orders', '(order_date, region)');
-- average_depth close to 1 = well clustered; large = filters on these columns scan many partitions
SELECT APPROX_COUNT_DISTINCT(region), APPROX_COUNT_DISTINCT(customer_id) FROM orders;
```

## Traffic and cost
```sql
SELECT query_id, query_text, warehouse_name, warehouse_size, execution_time/1000 AS exec_s,
       partitions_scanned, partitions_total, bytes_scanned, bytes_spilled_to_local_storage,
       bytes_spilled_to_remote_storage, queued_overload_time
FROM snowflake.account_usage.query_history
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY execution_time DESC LIMIT 50;
-- group by query_parameterized_hash to see how often a query pattern runs
```
Also ask: how often the table is loaded/updated (DML churn makes automatic clustering expensive), and the credit budget.

## ESR translated
- **Equality** filters on low/medium-cardinality columns → first clustering key columns.
- **Range** filter on a date/timestamp → next, usually as `TO_DATE(ts)` or `DATE_TRUNC('day', ts)` to limit cardinality.
- **Sort** does not drive clustering much; ORDER BY + LIMIT on a huge result is still a sort — reduce the input instead.
- High-cardinality point lookups (`WHERE id = ?`, `WHERE email = ?`, `LIKE '%x%'`, `ARRAY_CONTAINS`) → Search Optimization, not clustering.
- Clustering keys are only worth it on large tables (roughly 1 TB+ or thousands of micro-partitions) queried far more than they are modified.

## Get the plan
- Query Profile in Snowsight (ask the user for a screenshot or the operator stats).
- `EXPLAIN USING TEXT SELECT ...;` — estimated plan, shows `partitionsAssigned` vs `partitionsTotal`.
- `SELECT * FROM TABLE(GET_QUERY_OPERATOR_STATS(LAST_QUERY_ID()));` — actual per-operator rows, time, pruning, spills.

## Plan red flags
- `Partitions scanned` ≈ `Partitions total` with a selective filter → no pruning (filter not on clustering column, function on the column, poor clustering depth).
- `Bytes spilled to local/remote storage` → warehouse too small for the sort/join, or exploding join.
- Join producing far more rows than its inputs (duplicate keys) — "exploding join".
- High `queued_overload_time` → concurrency, not query design: multi-cluster warehouse.
- Most time in `TableScan` with low pruning → clustering / search optimization; most time in `Join`/`Aggregate` → query rewrite or bigger warehouse.

## Gotchas
- Result cache returns instantly for identical repeated queries — disable with `ALTER SESSION SET USE_CACHED_RESULT = FALSE` when benchmarking.
- Warehouse cache (local disk) makes the second run faster; compare cold vs warm.
- `SELECT *` on wide tables reads every column file — select only needed columns.
- Wrapping the clustering column in a function in WHERE (other than the one used in the clustering key) may prevent pruning.
