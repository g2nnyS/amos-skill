---
tags: [skill-creator]
max_turns: 25
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

# 从项目 AGENTS.md 生成项目级补充规则

为一个 AMOS 项目编写补充规则：从其 AGENTS.md 和贡献指南中提取项目专属的治理约束与工程规范。

**参考完整步骤：** [`docs/project-rules-from-agents.md`](../../docs/project-rules-from-agents.md)

## 任务

1. **扫描目标项目文档**（假设已在本机检出）：
   - `AGENTS.md` — 开发指南、架构、关键目录、重要文件、约定
   - `CONTRIBUTING.md` — 贡献流程、秘密存储、测试要求、审阅规范
   - `README.md` — 项目概览、依赖、发布流程
   - `.github/workflows/` — CI/CD 阶段与限制

2. **建立映射表**（AGENTS.md 节点 → operation → 判定 → 类型）

   | 位置 | operation | target | 判定 | 类型 | 理由 |
   | --- | --- | --- | --- | --- | --- |
   | Key Directories: data/ | file_write | `${SESSION_WORKSPACE}/data/**` | confirm | hard | 数据存储，需人工确认 |
   | Code Conventions: naming | — | — | advisory | advisory_only | 风格指导，无检查点 |

3. **生成 5-8 条规则**，混合：
   - **硬规则**（3-5 条）：关键目录写入、关键文件读写、部署配置等，需确定性拦截或确认
   - **建议规则**（2-3 条）：工程规范、测试覆盖、代码审阅建议等，无可执行检查点

4. **${SESSION_WORKSPACE} 使用**：
   - 所有项目路径模式以 `${SESSION_WORKSPACE}` 开头
   - 不硬编码绝对路径（除非是通用 glob 如 `/.git/**`）
   - 在模拟时将实际工作区路径替换进去验证

5. **验证**：
   - 本地导入校验（通过 `POST /import` 或离线 `parsePolicyDocument`）
   - 至少 1-2 条硬规则各测试 1 个应命中 + 1 个不应命中的动作
   - 记录 `${SESSION_WORKSPACE}` 展开验证结果

6. **与全局基线分离**：
   - 确认无与全局规则重复的 ID
   - 不将项目规则"升级"为全局
   - 在边界报告说明分工

7. **输出**：
   - 完整的 `policy.yaml`（14 字段完整，可直接导入）
   - `project-boundary-report.md`（规则总结、验证结果、缺口说明）
   - `project-context.md`（AGENTS.md 提取摘要，可选）

## 不可包含

- 猜测的项目约定（无 AGENTS.md 证据）
- 无法追溯的需求
- 秘密、token、API key 在 `quote` 中
- 硬编码的绝对路径（如 `/home/user/project/...`）
- `...` 占位符、`TODO` 或未完成的规则
- 与已发布全局规则重复的 ID
- 拼接多份 YAML 或省略必填字段

## 补充

- 项目规则是全局基线之上的**补充**，不替代全局规则
- `activation.includedRuleIds` 可为 `null`（选中所有）或明确列表
- 若编辑既有策略，保留其他规则的顺序、字段、ID，`activation` 和 `injection` 一并保留
- 仅生成与导入验证；不提交提案，不发布
