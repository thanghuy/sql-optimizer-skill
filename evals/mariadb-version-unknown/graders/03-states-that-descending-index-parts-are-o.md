---
type: llm
focus: trace
weight: 1
---

The response states that descending index parts are only honored from MariaDB 10.8 (older versions parse but ignore DESC, so the sort cannot be served by the index)
