---
type: llm
focus: trace
weight: 1
---

The response says to keep idx_payments_order_id because it backs the foreign key with ON DELETE CASCADE used by the nightly purge, even though idx_scan is 0
