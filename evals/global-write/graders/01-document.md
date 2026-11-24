---
type: llm
focus: trace
weight: 1
---

交付物同时包含 `inventory.md`、`policy.yaml`、`boundary-report.md`。`policy.yaml` 是完整 `amos-project-policy` 文档，含 `format`、`version`、`activation`、`injection`、`rules`，没有占位符、TODO 或拼接的多份文档；结构可通过 AMOS 全局 import 校验，或明确标记尚未连接服务。
