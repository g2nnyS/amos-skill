# amos-skill

给 Agent 用的 [AMOS](https://github.com/g2nnyS/project-amos) 规则编写指南。产物是一份可导入控制台的完整 YAML 草案：全局基线写给内置 `global` 项目；项目补充规则写给实际工作区。审批和发布由人完成。

## 使用

让 Agent 读取本仓库的 [`SKILL.md`](SKILL.md)，或将整个 `amos-skill` 目录放进客户端的 skills 目录。

## 从零生成全局基线

详见 [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) 的完整步骤：
- 第 0-1 步：环境准备与本机盘点（OS/arch、工具版本、skills 目录结构）
- 第 2 步：扫描 project-amos 仓库——提取真实的 operation 映射、现有规则、治理约定
- 第 3 步：建立映射表（依据 → operation → 判定 → 正反例）
- 第 4 步：编写 policy.yaml（14 个必填字段，source 出处或 null）
- 第 5-6 步：导入验证与模拟测试（至少 2 条规则各有完整正反例）
- 第 7 步：编写 boundary-report.md（规则总结、验证结果、缺口说明）

交付：`policy.yaml`（完整可导入）+ `boundary-report.md`（依据和缺口）+ `inventory.md`（环境盘点，可选）。仅生成与导入验证；不提交提案，不发布。

## 从项目 AGENTS.md 生成项目级规则

详见 [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) 的完整步骤：
- 扫描目标项目的 AGENTS.md、CONTRIBUTING.md、README 等文档
- 提取项目专属的治理约束与工程规范
- 建立映射表（AGENTS.md 节点 → operation → 判定 → 类型）
- 编写项目 policy.yaml（5-8 条规则，使用 `${SESSION_WORKSPACE}`）
- 导入验证与模拟测试（至少 1-2 条规则各有完整正反例）
- 明确与全局基线的分离（无重复 ID）
- 编写项目边界报告

交付：`policy.yaml`（完整可导入）+ `project-boundary-report.md`（依据和缺口）+ `project-context.md`（AGENTS.md 摘要，可选）。仅生成与导入验证；不提交提案，不发布。

## 草案怎么进入 AMOS

| 步骤 | 接口 | 结果 |
| --- | --- | --- |
| 查规则字段 | `GET /api/schema` | `rules` 数组的 schema |
| 查当前策略 | `GET /api/projects/:id/policy` | 当前已发布版本，不是 `/seed` |
| 导出完整 YAML | `GET /api/projects/:id/versions/:n/export?format=yaml` | 编辑用完整文档 |
| 导入草案 | `POST /api/projects/:id/import`，提交 `{"yaml":"…"}` | 校验结果和解析后的 `candidate`，不发布 |
| 再次校验 | `POST /api/projects/:id/validate`，提交 `{"candidate": …}` | 校验编辑后的候选对象 |
| 模拟动作 | `POST /api/projects/:id/simulate`，提交 `candidate` 和 `action` | 查看命中的规则和裁决，不执行动作 |
| 提交提案 | `POST /api/projects/:id/proposals` | 有明确授权时由 Agent 提交，交给人审批 |

`:id` 取 `global` 或控制面返回的项目 ID。接入本机服务时用 Agent Bearer 凭据；检查导入响应里的 `valid`，再把同一个 `candidate` 交给模拟或提案接口。直接向 `/validate`、`/proposals` 传 `yaml` 会丢弃 `injection`。没有服务时也可以离线编写，稍后再导入。

示例：[全局基线](examples/global.yaml) · [项目补充](examples/project.yaml)。两份都是独立文档，不要直接拼接，也不要用示例覆盖已有策略。
