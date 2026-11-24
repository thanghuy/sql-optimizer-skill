---
type: llm
focus: trace
weight: 1
---

The response flags ext_ref = <number> on a VARCHAR2 column as an implicit TO_NUMBER(ext_ref) that prevents index use (and can raise ORA-01722), and fixes it by binding the value as a string
