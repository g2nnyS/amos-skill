# amos-skill

给 Agent 用的 [AMOS](https://github.com/g2nnyS/project-amos) 规则编写指南。产物是一份可导入控制台的完整 YAML 草案：全局基线写给内置 `global` 项目；项目补充规则写给实际工作区。审批和发布由人完成。

## 使用

让 Agent 读取本仓库的 [`SKILL.md`](SKILL.md)，或将整个 `amos-skill` 目录放进客户端的 skills 目录。例如，Claude Code 的用户级安装位置是 `~/.claude/skills/amos-skill/`。

**从零生成全局基线：**

> 使用 amos-skill，生成全局规则 YAML 草案。要求写入任何仓库的 `.git/config` 前确认；区分读取和写入。给一个应命中和一个不应命中的动作。只生成草案，不提交提案。

**编辑项目规则：**

> 使用 amos-skill，从当前项目已发布的策略和仓库里的贡献指南开始。为指南里列出的发布目录补充写入确认规则，保留其他规则、激活选择和注入配置。输出完整 YAML、规则来源以及模拟结果。没有服务连接时，请标记哪些用例尚未运行。

## 草案怎么进入 AMOS

| 步骤 | 接口 | 结果 |
| --- | --- | --- |
| 查规则字段 | `GET /api/schema` | `rules` 数组的 schema |
| 导入草案 | `POST /api/projects/:id/import`，提交 `{"yaml":"…"}` | 校验结果和解析后的 `candidate`，不发布 |
| 再次校验 | `POST /api/projects/:id/validate`，提交 `{"candidate": …}` | 校验编辑后的候选对象 |
| 模拟动作 | `POST /api/projects/:id/simulate`，提交 `candidate` 和 `action` | 查看命中的规则和裁决，不执行动作 |
| 提交提案 | `POST /api/projects/:id/proposals` | 有明确授权时由 Agent 提交，交给人审批 |

`:id` 取 `global` 或控制面返回的项目 ID。接入本机服务时用 Agent Bearer 凭据；检查导入响应里的 `valid`，再把同一个 `candidate` 交给模拟或提案接口。直接向 `/validate`、`/proposals` 传 `yaml` 会丢弃 `injection`。没有服务时也可以离线编写，稍后再导入。

示例：[全局基线](examples/global.yaml) · [项目补充](examples/project.yaml)。两份都是独立文档，不要直接拼接，也不要用示例覆盖已有策略。
