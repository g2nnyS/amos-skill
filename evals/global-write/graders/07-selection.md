---
type: llm
focus: trace
weight: 1
---

`boundary-report.md` 给出有/无 hard rule 的选择流程：本 fixture 因明确要求 `.git/config` 直接写入而选择一条窄 confirm；并说明如果没有这种明确共通要求，应退回 advisory-only。报告体现优先级“明确需求/已采用约定 > 仓库固定文件 > 机器探测”，明确检测到工具不自动生成禁令，也没有用广泛黑名单假装完整性。
