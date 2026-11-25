---
type: llm
focus: trace
weight: 2
---

# 评分：项目规则范围与依据

检查生成的规则是否有实际依据、覆盖真实的项目治理需求：

1. **AGENTS.md 映射的真实性**
   - [ ] 每条规则都能在目标项目的 AGENTS.md、CONTRIBUTING.md、README 中找到对应证据
   - [ ] source 对象的 `path`、`section`、`lines`、`quote` 都指向真实文件与准确位置
   - [ ] 若 `source: null`，narrative 明确说明为什么没有单一出处（如"多处约定聚合"）

2. **硬规则与建议规则的区分**
   - [ ] 硬规则（3-5 条）：关键目录、关键文件、敏感操作，确实需要人工确认或拦截
   - [ ] 建议规则（2-3 条）：工程规范、设计偏好、工作流建议，当前无可执行检查点
   - [ ] 判定与类型一致：硬规则有非空 predicate 和 `enforcement: runtime/ci/server`；建议规则 `predicate: null` + `enforcement: advisory`

3. **operation 和 target 的有效性**
   - [ ] operation 仅用已验证的类型：`file_read`、`file_write`、`file_search`、`file_list` 等
   - [ ] 不用未验证的操作（如 `http_request`，除非有特别理由且标记为 advisory）
   - [ ] target 路径模式真实存在或通用（如 `${SESSION_WORKSPACE}/data/**`、`${SESSION_WORKSPACE}/config/prod.yaml`）

4. **${SESSION_WORKSPACE} 的正确使用**
   - [ ] 所有项目工作区路径以 `${SESSION_WORKSPACE}` 开头
   - [ ] 不硬编码绝对路径（如 `/home/user/my-project`）
   - [ ] `${SESSION_WORKSPACE}` 只在项目规则中出现（不在全局规则中）
   - [ ] glob 模式规范：`*` 不跨 `/`，`**` 跨层级
   - [ ] 在 target_outside_glob 中也正确使用（如排除备份目录）

5. **正反例覆盖**
   - [ ] 至少 1-2 条硬规则各有 1 个应命中 + 1 个不应命中的测试用例
   - [ ] 正例演示规则何时触发（确认会命中）
   - [ ] 反例演示规则何时不触发（确认不会误报）
   - [ ] 例子具体且可复现

6. **与全局基线的分离**
   - [ ] 没有与全局规则 ID 重复（如 `global-secret-files` 不出现在项目规则中）
   - [ ] 项目规则专注于项目内部约定，不试图重新定义全局强制
   - [ ] 若需要继承全局规则，通过 `injection.baselineRuleIds` 明确列出，不重新定义
   - [ ] boundary-report.md 明确说明全局 + 项目规则如何协作

7. **规则判定的合理性**
   - [ ] `confirm` 规则：确实值得人工审查的操作
   - [ ] `deny` 规则：项目内部的强制安全边界
   - [ ] `log` 规则：便于审计的操作
   - [ ] `advisory` 规则：明确为工程偏好或尚无检查点
   - [ ] 判定与依据（AGENTS.md 节点）一致

8. **缺口与限制的说明**
   - [ ] 明确指出哪些需求无法在项目规则中覆盖（如跨项目的共享资源）
   - [ ] 说明离线验证的局限（如"HTTP 请求需要在线 API"）
   - [ ] 不隐瞒、不过度声称

**评分**：
- 5-8 条规则有实据、硬规则与建议规则清晰分工、至少 1-2 条规则有完整正反例测试、明确缺口说明、与全局基线分离清楚 → 优秀
- 4-6 条规则、依据大多可追溯、1 条规则测试完整、缺口说明清楚 → 合格
- <4 条规则、依据不清、无测试或与全局基线混淆 → 需改进
- 包含猜测约定、无追溯的需求、ID 重复、硬编码绝对路径 → 不及格
