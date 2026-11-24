---
type: llm
focus: trace
weight: 1
---

正反例匹配保持正确：直接 `file_write` 到 `/workspace/demo/.git/config` 得到 `confirm` 且命中 `global-git-config-write`；`file_read` 同一路径不触发该规则；`file_write` 到 `.git/config.bak` 或 `.git/hooks/pre-commit` 不命中。`boundary-report.md` 记录实际 simulation/离线结果，或明确说明未连接服务、哪些动作未运行。
