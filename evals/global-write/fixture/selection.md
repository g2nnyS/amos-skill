# 硬规则选择流程

用本 fixture 对照两种结果：

## 有明确硬规则需求（本 fixture）

- 用户明确把 `.git/config` 直接写入列为全局 confirm 路径。
- 它对仓库共通、使用真实的 `file_write` operation，并且有清楚的正反例。
- 结果：恰好一条窄 deterministic 规则（`global-git-config-write`，`confirm`），可按需附带 advisory 规则。

## 没有明确硬规则需求（对照）

- 如果用户只要求通用基线，没有明确指定共通路径/动作，不要发明硬规则。
- 结果：advisory-only 策略；在 `boundary-report.md` 说明为什么没有 hard rule。

## 不要

- 不要把探测到的工具（docker、MCP、Shell、联网、进程执行）变成硬规则。
- 不要把 `docs/releases/`、`.github/` 等项目专属目录写成全局硬规则。
- 不要用宽泛的 deny/allow 列表并宣称覆盖完整。
