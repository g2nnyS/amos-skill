---
type: llm
focus: trace
weight: 1
---

`policy.yaml` 只有一条 deterministic 硬规则：ID `global-git-config-write`，`op_in` 精确覆盖 `file_write`，`target_glob` 精确匹配任意仓库的 `.git/config`，`on_hit: confirm`。没有第二条硬规则，没有针对 docker/MCP/Shell/联网/进程执行的硬规则，也没有宽泛黑名单或宣称覆盖完整。`shell_input`、`http_request` 等不可评估通道只以 `advisory_only` 出现或省略。
