# 合成机器盘点

- OS：Linux（fixture）
- 架构：x86_64
- Shell：bash
- 已见工具：node 26.x、npm、git、rg、docker、python3
- 用户 skills：`fixture/home/.claude/skills/` 下的 example-skill
- Agent settings：`fixture/home/.config/agent/settings.json`

只记录结构：

- hooks：`preToolUse`、`postToolUse`
- plugins：一个启用的 fixture 插件
- mcpServers：`filesystem`、`github`

settings 文件含占位 secret 字段。只读取结构和键名，不读取、复制或输出它们的值。
