---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

给 AMOS 的全局项目写一份新的策略草案：任何工作区里，对文件名恰好为 RELEASE-SIGNOFF 的文件做 file_write 时需要确认，读取不触发这条规则。要可导入的完整 policy.yaml，规则 ID 用 global-release-signoff。只在本地生成，不提交提案。
