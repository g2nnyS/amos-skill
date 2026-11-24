---
type: llm
focus: trace
weight: 1
---

如果选择了 advisory 规则（例如把探测到的 MCP、Shell、联网、进程执行写成建议），它们的 ID 必须同时出现在 `activation.includedRuleIds`，且 `injection.baselineRuleIds` 或 `injection.routes` 只引用已激活的 advisory ID；没有悬空引用。若没有 advisory 规则，`boundary-report.md` 说明为何保持空注入，而不是留下未解释的孤立配置。
