---
type: llm
focus: trace
weight: 1
---

global-release-signoff 对不同工作区下文件名为 RELEASE-SIGNOFF 的直接写入命中 confirm；读取该文件和写同名后缀 .bak 均不命中此规则。
