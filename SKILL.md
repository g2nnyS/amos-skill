---
name: amos-skill
description: 为 AMOS 治理控制台编写可导入的全局基线或项目规则 YAML 草案，或安全地编辑既有策略。用于生成/修改 AMOS policy.yaml、规则、激活选择与注入配置，并完成导入校验、动作模拟和人工发布交接。
---

# AMOS 规则 YAML 编写

产物是一份**完整、可导入的 `policy.yaml` 草案**。全局基线交给内置 `global` 项目；仓库规则作为工作区项目的补充规则。草案只用于导入、审阅和模拟；运行期权威是人工批准的已发布版本。

从[全局示例](examples/global.yaml)或[项目示例](examples/project.yaml)选一个起点，不要拼接。[编写参考](references/authoring.md)列出算子参数、在线模拟和离线校验命令；服务可用时，以 `GET /api/schema` 返回的规则表契约为准。

**新建全局基线？** 参考[从零生成全局基线的完整步骤](docs/global-baseline-generation.md)，包括环境盘点、仓库扫描、映射表、导入与模拟验证的完整例子。

**生成项目规则？** 参考[从项目 AGENTS.md 生成项目级规则](docs/project-rules-from-agents.md)，包括项目文档扫描、`${SESSION_WORKSPACE}` 使用、与全局基线的分离。

**多智能体系统中的治理？** 参考[多智能体系统治理指南](docs/multi-agent-governance.md)，理解如何在多 Agent 协作中区分善意与恶意行为、分层规则模型、操作序列分析、Agent 身份与信任。

## 开始前确认

1. **目标**：内置 `global`，还是某个工作区项目 ID。
2. **现状**：新建、编辑指定的草稿文件，还是修改已发布策略？第二种以用户指定的文件为原稿；第三种先导出已发布版本。不要用仓库现状或示例重建缺失的原稿。
3. **授权**：默认只产出草案并做导入/模拟。只有用户明确授权时才提交提案；审批、发布永远不由 Agent 做。
4. **产物**：`inventory.md`（非秘密盘点）、`policy.yaml`（完整文档）、`boundary-report.md`（边界与验证报告）。三份同时交付。

## 只读调研

### 本机盘点（全局目标）

全局目标先了解运行环境，只读、只采样与任务相关的部分：

- OS/arch、shell，以及相关工具的版本/存在性：`node`、`npm`、`git`、`rg`、容器、语言运行时等；不要导出整个环境变量表。
- 用户 skills：只列客户端 skill 根下的目录名，按需再读具体 skill；不要把 skill 内容当系统事实。
- 已存在的 agent settings：只检查 hook、plugin、MCP 的结构（文件名、顶层键、服务名、启用状态）。**不要读取、打印或复制** `apiKey`、`token`、`env` 等 secret 值；需要时只输出键名和计数。
- 盘点结果写进 `inventory.md`，标记 present/absent 和“已见但不据此设规则”。检测到工具不等于要禁止或确认它。

### 仓库与固定目录

- 读当前仓库的 `AGENTS.md`、`CONTRIBUTING.md`、`README*`、包/构建清单，以及 `.github/`、`docs/`、`scripts/` 等固定目录中与任务相关的文件。
- **全局目标**：这些文件只提供共通约定样本，不等价于项目全部约定。只把跨仓库共通的路径/动作写进全局；项目专属目录、语言、发布流程写项目层，并在 `boundary-report.md` 说明。
- **项目目标**：以当前已发布策略和该项目仓库约定为准，仍不把仓库数据自动当成已批准规则。

### 真实适配器

在 AMOS 检出中确认动作通道：`adapters/pi/normalize.ts`、`adapters/opencode-v2/normalize.ts`、`host/contracts.ts`，以及 `strict/`、`autonomy/` 的实际 operation 映射。只有真实发出的 operation 和目标类型才能写成硬规则。

### 证据、采样与优先级

先做映射，再写 YAML。每条规则至少填清楚：

| 依据 | 动作 / 精确范围 | 判定 / 建议 | 正反例 |
| --- | --- | --- | --- |
| 文件、约定或已确认需求 | operation + target 模式 | `deny`/`confirm`/`log` 或 `advisory_only` | 一个应命中、一个不应命中 |

- 只采样与任务相关的工具/路径/约定；其余记录为“已见，不据此设规则”。
- 优先级：明确用户需求或已采用约定 > 仓库固定文件 > 机器探测。冲突时报告，不静默选择更严规则。
- 已批准需求与仓库数据分开：需求决定要不要规则；代码、配置和文档只提供动作范围与风险证据。没有证据的约束写 `advisory_only`。

### 取已有版本

- 当前已发布策略：`GET /api/projects/:id/policy`。
- 精确导出的完整 YAML：`GET /api/projects/:id/versions/:n/export?format=yaml`；`n` 从 `GET /api/projects/:id/versions` 获取。
- `GET /api/seed` 只是仓库 seed candidate，不是全部已发布策略；不要把它当成当前策略或完整基线。
- 导出是编辑基线；保留原规则、顺序、`activation` 与 `injection`。没有导出就把“需要原稿”作为阻塞项返给用户。

## 硬规则与建议规则

硬规则只在动作真实发出时产生效果。当前可可靠匹配：

| operation | 实际来源 |
| --- | --- |
| `file_read` | `read` |
| `file_write` | `write`、`edit`；strict 的 `delete` 与 autonomy 文件创建也走此通道 |
| `file_search` | `grep`、`find`、`glob` |
| `file_list` | `ls`、`list` |
| `file_mkdir` / `file_delete` | managed file host |
| `shell_input` | 仅 coverage，**不会命中规则** |

`file_stat`、`file_rename`、`file_perm`、`file_link`、`file_truncate`、`http_request` 等可被 Schema 接受，但当前普通适配器不发出；可作为未来基线，现状下是惰性的。保护当前行为只使用上表 operation，不宣称已拦截其余动作。

以下写成 `advisory_only`：工程规范、设计偏好、Shell/网络/文件内容约束、MCP/进程执行、还没有可执行检查点的约定。建议规则固定 `predicate: null`、`enforcement: advisory`，只进入 narrative 注入。

**硬规则选择流程**（有/无 hard rule 的对照）：

1. 有明确用户需求或已采用的全局约定？没有就只写 `advisory_only`。
2. 是跨仓库共通的路径/动作？不是就写项目层，并在 `boundary-report.md` 说明。
3. 是当前真实发出的 operation？不是就写建议。
4. 能用窄 predicate 表达，并给出一个应命中、一个不应命中的动作？不能就写建议。
5. 全满足才写 `deny`/`confirm`/`log`；只写用户明确列出或仓库共通约定支持的窄规则，不因探测到工具自动新增。

Shell、联网、MCP、进程执行等当前不可结构化匹配的通道一律优先建议；不要用广泛黑名单宣称覆盖完整。

## 文档与字段

完整文档包含 `format: amos-project-policy`、`version: 1`、`activation`、`injection`、`rules`。真实写法直接看 `examples/global.yaml` 与 `examples/project.yaml`，不要在正文复制大段示例。

- 草案省略 `project` 块，保持跨机器可移植；出现时其 `id`/`workspace` 必须与目标一致。
- `activation.includedRuleIds` 为 `null` 时选中全部已知规则；为数组时只选这些 ID。`excludedRuleIds` 与显式 included 不能重叠。
- `injection.baselineRuleIds` 和 `injection.routes` 只影响建议投影的选择与排序，不改变裁决。
- 规则 14 个字段全部必填：`id`、`dimension`、`level`、`expressivity`、`predicate`、`narrative`、`on_hit`、`scope`、`ttl`、`owner`、`source`、`probe`、`policy_class`、`enforcement`。
- `source` 可为 `null`；填对象时 `repo`、`rev`、`path`、`section`、`lines`、`quote` 都需非空。没有可靠出处用 `null`，不编造，不把秘密或长命令放进 `quote`。
- `deterministic`：`predicate` 非空，`enforcement` 为 `runtime`/`ci`/`server`。`advisory_only`：`predicate: null`，`enforcement: advisory`。
- `level` 只影响建议排序/审计优先级；`scope` 只是审计元数据，不筛选规则，也不决定继承。继承由项目绑定与激活集合决定。

## 匹配与判定

七个算子：`op_in`、`target_glob`、`target_outside_glob`、`url_query_key_in`、`target_scheme_in`、`target_host_in`、`target_host_not_in`。`predicate` 数组内逐项 AND，没有 OR、正则或自定义算子。

- 文件 target 必须是规范化绝对路径；引擎不做 realpath、不展开 `~`。
- `file_search`/`file_list` 的 target 是检索/列举的根路径，不是每个结果文件。
- `*` 不跨 `/`，`**` 跨层级；`**/.netrc` 命中任意层级的 `.netrc`，不命中 `.netrc.bak`。
- allow 列表（如 `target_outside_glob`）只列探测并模拟验证过的路径；不要用宽泛黑名单假装覆盖完整。
- `${SESSION_WORKSPACE}` 只用于 `target_glob`/`target_outside_glob` 的项目规则。全局项目自身 workspace 为 `/`，全局规则推荐不用该 token；实际组合/模拟按目标项目 workspace 展开。
- URL 算子要求绝对 URL；把 `op_in` 放第一条，否则 URL 规则遇到文件 action 会让整次评估 fail-closed。
- `ttl` 为 `null`、`YYYY-MM-DD` 或精确 UTC ISO 时间；到期不参与评估，发布不会清理或续期。
- 命中多个规则时按 `deny > confirm > log > allow`。重复 ID、空表、未知字段、非法参数或非规范化路径都会 fail-closed。

## 全局与项目

- 全局项目固定 ID `global`，规则继承到所有工作区项目；只写跨仓库共通的路径/动作，不写某个仓库的目录或绝对路径。项目专属目录、语言、发布流程等写项目层，并在 `boundary-report.md` 说明。
- 项目候选只保存补充规则。不要复制全局规则；同 ID 不同定义会被拒绝。
- 生效集合 = 固定基线 ∪ 选中项（全局 + 项目）− 合法排除。项目不能排除固定基线或已激活的全局规则。
- 全局发布影响所有已发布项目；只提醒人类影响面，不代替发布。

## 编辑既有策略

1. 读取用户指定的草稿，或用 `/policy` 和 `/versions/:n/export?format=yaml` 取已发布原稿；没有原稿就停。
2. 只改用户指定的规则。其余规则保持原顺序和原字段，`activation` 与 `injection` 一并保留；直接文本编辑时连注释和排版也保留。
3. 新增规则：显式 `includedRuleIds` 数组要加入新 ID；`null` 会自动选中。新 ID 不得与 `excludedRuleIds` 重叠。
4. 删除规则：从 `includedRuleIds`、`excludedRuleIds`、`injection.baselineRuleIds`、每条 route 的 `add` 和 `suppress` 清理该 ID，并确保有效集合不为空。
5. 不改历史版本、不改 ID 既有含义、不把项目规则“升级”成全局规则；改全局规则回 `global` 做。
6. 只有用户明确授权时才提交提案，body 明确包含 `{ title, rationale, baseVersion, candidate }`；`baseVersion` 取当前版本，首次发布为 `0`，`candidate` 用 `/import` 返回值。

## 验证

完整文档只有一条入口：**先 `/import`，再用返回的 `candidate`**。

1. `POST /api/projects/:id/import`，body `{"yaml":"<完整文档>"}`；它保留 `injection`，返回 `{valid, issues, candidate}`。`valid` 不为 `true` 就停止。
2. 后续接口都传 `candidate`：
   - `POST /api/projects/:id/validate`：`{"candidate": <candidate>}`
   - `POST /api/projects/:id/simulate`：`{"candidate": <candidate>, "action": {...}}`
3. 至少一个应命中、一个不应命中；核对 `matched_rule_ids` 与 `verdict`。修改既有策略时抽查未改规则，确认激活和注入未被整体改写。
4. 无服务时可离线调用原生 `parsePolicyDocument` 与 `createRuleValidator` / `validateAndCompileCandidate`，检查完整文档和工作区展开；没有在线固定 catalog / 已发布全局层时，引用其规则 ID 的草案无法完整校验。必须说明未在线导入/模拟。

**不要**把完整文档的裸 `{yaml}` 直接发给 `/validate` 或 `/proposals`：该简写只读 `rules` 和 `activation`，会静默丢掉 `injection`。

## 人工发布界线

- 草案、导入结果、模拟结果都只是验证材料；生效策略来自人工批准的发布版本。
- Agent 可以读取、校验、模拟；只有明确授权才提交提案。**不得审批、发布、回滚或改数据库/激活产物**。
- `POST /api/proposals/:id/decision` 是人工操作；`owner: agent` 只表示来源，权限由人工审批决定。
- 不手改 `.pi/activation.json`、`.pi/amos.json` 或运行回执。发布与重新加载由控制台和宿主完成。
- 即使已发布，普通适配器仍可能是 `log-only`；只陈述规则和验证结果，不宣称已形成运行期强制或安全边界。

## 输出

同时交付：

- `inventory.md`：非秘密盘点，含 OS/arch、相关工具与版本、skills 目录名、agent settings 的 hook/plugin/MCP 结构、读过的仓库固定文件；不含 secret 值。
- `policy.yaml`：一份完整 YAML，不拼接多份文档，不留占位符。
- `boundary-report.md`：需求到规则的映射、全局/项目边界、hard/advisory 选择理由、正反例结果、未运行/未验证项，以及未读取 secret 的说明。

缺少目标、原稿或服务连接时写明缺什么；不用“已生效”“已发布”替代未完成的验证。
