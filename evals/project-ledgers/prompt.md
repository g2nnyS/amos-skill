---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

为一个未发布过本地规则的 AMOS 项目起草 policy.yaml。维护者已确认两条项目约定：① 直接写 ops/ledgers/ 下面任何层级的文件须确认，用 ID project-ledger-write；② 修改账本格式前先和维护者讨论，用 ID project-ledger-format。把第二条作为开发建议，不要推断提交内容。不要写死工作区位置；不连接服务，不发布。
