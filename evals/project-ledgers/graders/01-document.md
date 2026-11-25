---
type: llm
focus: trace
weight: 1
---

# 评分：项目规则文档有效性

检查生成的 `policy.yaml` 是否满足 AMOS 项目的导入要求：

1. **文档结构**
   - [ ] 包含 `format: amos-project-policy` 和 `version: 1`
   - [ ] 包含 `activation`（`includedRuleIds` 或 `null`，`excludedRuleIds`）
   - [ ] 包含 `injection`（`baselineRuleIds`、`routes`）
   - [ ] 包含 `rules` 数组

2. **规则完整性**（每条规则 14 个字段）
   - [ ] `id`：非空字符串，遵循 `project-*` 或 `<project-name>-*` 命名
   - [ ] `dimension`、`level`、`expressivity`、`scope`：有效值
   - [ ] `predicate`：非空数组（硬规则）或 `null`（建议规则）
   - [ ] `narrative`：清晰的中文说明，不空
   - [ ] `on_hit`：`deny`/`confirm`/`log`/`allow`，与规则类型匹配
   - [ ] `ttl`：`null`、日期格式或 ISO 时间
   - [ ] `owner`：非空字符串（如项目名、团队、维护者）
   - [ ] `source`：完整对象（`repo`/`rev`/`path`/`section`/`lines`/`quote` 都非空）或 `null`
   - [ ] `probe`、`policy_class`、`enforcement`：有效枚举值

3. **谓词有效性**（硬规则）
   - [ ] `predicate` 不为 `null` 时：`enforcement` 是 `runtime`/`ci`/`server`
   - [ ] 所有算子合法：`op_in`、`target_glob`、`target_outside_glob`、`url_query_key_in`、`target_scheme_in`、`target_host_in`、`target_host_not_in`
   - [ ] 文件路径使用 `${SESSION_WORKSPACE}` 作为工作区前缀
   - [ ] 路径模式规范化且可靠（如 `${SESSION_WORKSPACE}/ops/ledgers/**`，不是 `~/project/...` 或绝对路径）
   - [ ] `op_in` 在数组首位（若存在）

4. **${SESSION_WORKSPACE} 使用**
   - [ ] 只在项目规则中出现（全局规则不用）
   - [ ] 路径模式以 `${SESSION_WORKSPACE}` 开头，如 `${SESSION_WORKSPACE}/data/**`
   - [ ] 不硬编码绝对路径（除非通用 glob 如 `/.git/**`）
   - [ ] 在 target_glob 和 target_outside_glob 中规范使用

5. **建议规则有效性**（advisory_only）
   - [ ] `predicate: null`
   - [ ] `enforcement: advisory`

6. **与全局基线的分离**
   - [ ] 所有项目规则 ID 不与全局规则重复
   - [ ] `activation.includedRuleIds` 中的 ID 都在 `rules` 中定义
   - [ ] `excludedRuleIds` 与 `includedRuleIds` 无重叠
   - [ ] `injection.baselineRuleIds` 只引用全局规则 ID（不引用本项目规则）

7. **交叉检查**
   - [ ] 无重复 ID（项目内部、与全局规则）
   - [ ] 无空字段或省略必填字段
   - [ ] 无未知字段或不符合 schema 的值

8. **可导入性**
   - [ ] 通过 AMOS 的 `POST /import` 返回 `valid: true`（若有在线服务）
   - [ ] 或通过离线 `parsePolicyDocument` 解析成功，无结构错误
   - [ ] 若编辑既有策略，`activation` 与 `injection` 配置一致

**评分**：符合上述全部项则完全合格；缺一项则扣分；无法导入或关键字段为空则不及格。
