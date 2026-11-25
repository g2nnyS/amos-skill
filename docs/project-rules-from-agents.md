# 从项目 AGENTS.md 生成项目级规则 — 完整步骤

本指南适用于**根据现有项目的 AGENTS.md、CONTRIBUTING.md 等文档生成项目级补充规则**。项目规则在全局基线之上增加工作区特定的治理约束。

## 第 0 步：确认目标与范围

1. **目标明确**
   - 目标项目的 UUID（从 AMOS 控制台或 `GET /api/projects` 获取）
   - 或用 `${SESSION_WORKSPACE}` 作为工作区项目的通用占位符

2. **现状梳理**
   - 该项目是否已有已发布的规则？`GET /api/projects/:id/policy`
   - 若有，本次是**编辑现有策略**；若无，是**新建项目规则**
   - 本指南重点是**从 AGENTS.md 提取约定**生成新规则

3. **与全局基线的关系**
   - 全局基线（`global` 项目）是跨仓库共通规则
   - 项目规则是**补充**，不重复全局规则，不能有同 ID 不同定义
   - 项目规则可用 `${SESSION_WORKSPACE}` 引用工作区路径（全局规则不能）

## 第 1 步：读取 AGENTS.md 与核心文档

### 1a. 提取项目基本信息

从 AGENTS.md 或 README 中读：

```yaml
Project:
  name: <project-name>
  purpose: <brief description>
  languages: [...]
  key_tools: [...]
  entry_points: [...]
```

记在 `project-context.md`（供内部参考）。

### 1b. 提取治理约定和安全约束

逐一读取，提取**项目专属的**强制规则和建议：

```bash
# 核心文档
cat $PROJECT_ROOT/AGENTS.md          # 开发指南、约定、工具链
cat $PROJECT_ROOT/CONTRIBUTING.md    # 贡献指南、审阅流程、秘密存储
cat $PROJECT_ROOT/README.md          # 项目概览、依赖、发布流程
ls -la $PROJECT_ROOT/.github/workflows/ | head -20  # CI/CD 流程

# 敏感与关键目录
grep -r "ledger\|database\|state" $PROJECT_ROOT --include="*.md" | head -20
grep -r "secret\|key\|token" $PROJECT_ROOT/AGENTS.md --include="*.md"
grep -r "config.*write\|\.env\|deployment" $PROJECT_ROOT/AGENTS.md
```

**关键问题**：
- 哪些目录是**关键数据存储**（如账本、数据库、状态文件）？
- 哪些操作需要**人工审查**（如部署、格式变更、秘密修改）？
- 有哪些**工程规范**（代码风格、测试覆盖、发布检查清单）？

### 1c. 映射到 operation 和 target

```markdown
| AGENTS.md 节点 | 含义 | operation | target 模式 | 判定 | 类型 |
|---|---|---|---|---|---|
| Key Directories: ops/ledgers/ | 账本存储位置 | file_write | `${SESSION_WORKSPACE}/ops/ledgers/**` | confirm | hard |
| Code Conventions: schema changes | 格式变更需讨论 | — | — | advisory | advisory_only |
| Testing: coverage expectation | 新功能需要测试 | — | — | advisory | advisory_only |
| Important Files: config.yaml | 生产配置，限制读写 | file_read, file_write | `${SESSION_WORKSPACE}/config/prod.yaml` | log/confirm | hard |
```

## 第 2 步：审视已发布策略（若有）

若项目已有已发布规则：

```bash
# 获取当前发布版本
curl -s -H "Authorization: Bearer $AMOS_AGENT_TOKEN" \
  "http://127.0.0.1:$AMOS_PORT/api/projects/$PROJECT_ID/policy" | jq '.' > existing-policy.json

# 或导出完整 YAML
curl -s -H "Authorization: Bearer $AMOS_AGENT_TOKEN" \
  "http://127.0.0.1:$AMOS_PORT/api/projects/$PROJECT_ID/versions/1/export?format=yaml" > existing-policy.yaml
```

**编辑时的规则**：
- 保留现有规则的顺序、字段、ID
- 仅修改用户指定的规则
- `activation` 和 `injection` 一并保留
- 新增规则加入 `includedRuleIds`，删除规则从各处清理

## 第 3 步：编写映射表

基于提取的约定，建立完整的映射表（同全局基线第 3 步）：

```markdown
## 项目规则映射表

| ID | 依据（AGENTS.md 位置） | operation | target | 判定 | 类型 | 正例 | 反例 | 备注 |
|---|---|---|---|---|---|---|---|---|
| project-ledger-write | Key Directories: ops/ledgers/ | file_write | `${SESSION_WORKSPACE}/ops/ledgers/**` | confirm | hard | 写 ops/ledgers/2024-Q1.json | 写 src/index.ts | 账本是关键数据存储 |
| project-config-prod | Important Files: config/prod.yaml | file_write | `${SESSION_WORKSPACE}/config/prod.yaml` | confirm | hard | 修改 prod.yaml | 修改 config.example.yaml | 防止意外部署配置变更 |
| project-schema-change | Code Conventions: schema changes | — | — | advisory | advisory_only | — | — | 格式变更需向维护者讨论，无可执行检查点 |
| project-test-coverage | Testing & QA | — | — | advisory | advisory_only | — | — | 新功能应有对应测试 |
```

## 第 4 步：编写 policy.yaml

基于映射表，从 `examples/project.yaml` 起点创建：

```yaml
format: amos-project-policy
version: 1

activation:
  includedRuleIds: null  # null = 选中所有；或明确列出 [project-ledger-write, ...]
  excludedRuleIds: []

injection:
  baselineRuleIds: []     # 若无继承特定全局规则，通常为空
  routes: []              # 项目规则通常无额外路由

rules:
  - id: project-ledger-write
    dimension: "integrity"
    level: "critical"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args:
          operations: ["file_write"]
      - op: target_glob
        args:
          patterns: ["${SESSION_WORKSPACE}/ops/ledgers/**"]
      - op: target_outside_glob
        args:
          patterns: ["${SESSION_WORKSPACE}/ops/ledgers/**/.backup"]
    
    narrative: |
      账本文件是此项目的关键数据存储，任何直接修改必须经过人工确认。
      备份目录除外（临时回滚用）。
      
    on_hit: "confirm"
    scope: "operation:file_write target:ledgers"
    ttl: null
    owner: "project-maintainers"
    
    source:
      repo: "primary"
      rev: "HEAD"
      path: "AGENTS.md"
      section: "Key Directories"
      lines: [15, 20]
      quote: "ops/ledgers/ — project-critical account ledger; read-only in production, write-confirmed in dev"
    
    probe: "fs_writeable"
    policy_class: "mandatory"
    enforcement: "confirm"
  
  - id: project-config-prod
    dimension: "confidentiality"
    level: "high"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args:
          operations: ["file_write", "file_read"]
      - op: target_glob
        args:
          patterns: ["${SESSION_WORKSPACE}/config/prod.yaml"]
    
    on_hit: "confirm"  # 读写都要确认
    source:
      repo: "primary"
      rev: "HEAD"
      path: "AGENTS.md"
      section: "Important Files"
      lines: [42, 48]
      quote: "config/prod.yaml: Production deployment configuration, restricted to maintainers"
    
    enforcement: "confirm"
    # ... 其余 14 字段 ...
  
  - id: project-schema-change
    dimension: "maintainability"
    level: "medium"
    expressivity: "guidance"
    
    predicate: null  # 建议规则
    enforcement: "advisory"
    
    narrative: |
      修改账本格式前，必须与项目维护者讨论以确保向后兼容性和迁移计划。
      相关文件：ops/schemas/ledger.json、ops/migration/
      
    source:
      repo: "primary"
      rev: "HEAD"
      path: "CONTRIBUTING.md"
      section: "Schema Changes"
      lines: [65, 72]
      quote: "Before modifying the ledger schema, discuss with maintainers to ensure backward compatibility and provide migration scripts."
    
    # ... 其余字段 ...
```

**关键点**：
- 使用 `${SESSION_WORKSPACE}` 作为工作区路径前缀
- 只在项目规则中使用（全局规则 workspace 是 `/`）
- `target_glob` 和 `target_outside_glob` 中均可用
- 在导入/模拟时会按目标项目的 workspace 展开

## 第 5 步：导入与验证

### 在线验证

```bash
export DRAFT="./project-policy.yaml"
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="<UUID>"

# Step 1: Import
IMPORT_RESULT=$(node --input-type=module << 'EOF'
import fs from "fs";
const yaml = fs.readFileSync(process.env.DRAFT, "utf-8");
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/import`,
  {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`,
      "Content-Type": "text/yaml"
    },
    body: yaml
  }
);
const result = await resp.json();
console.log(JSON.stringify(result, null, 2));
EOF
)

echo "$IMPORT_RESULT" | jq '.valid'  # 必须是 true
```

### 离线验证

```bash
# 需要 AMOS 检出
cd $AMOS_ROOT && npm ci && npm run build

node --input-type=module << 'EOF'
import { parsePolicyDocument } from "$AMOS_ROOT/dist/control-plane/document.js";
import { createRuleValidator } from "$AMOS_ROOT/dist/control-plane/compiler.js";
import YAML from "js-yaml";
import fs from "fs";

const yaml = fs.readFileSync("./project-policy.yaml", "utf-8");
const doc = YAML.load(yaml);
const parsed = parsePolicyDocument(doc);

console.log("Parsed rules:", parsed.rules.map(r => r.id));
console.log("Activation:", parsed.activation);

// 创建验证器（假设已加载全局基线规则）
const validator = createRuleValidator(parsed.rules, {
  workspace: "${SESSION_WORKSPACE}"  // 项目工作区（若有）
});

// 测试一个动作
const testAction = {
  operation: "file_write",
  target: "/workspace/ops/ledgers/2024-Q1.json"
};
const result = validator.validate(testAction);
console.log("Test result:", result);
EOF
```

## 第 6 步：模拟测试

选择 2-3 条项目规则，各测试应命中和不应命中的情况。重点关注 `${SESSION_WORKSPACE}` 展开：

```bash
# 假设 $SESSION_WORKSPACE = /home/user/my-project-workspace

# 规则 1: project-ledger-write
# 应命中：写 ops/ledgers/ 下文件
node --input-type=module << 'EOF'
const candidate = /* from import */;
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/simulate`,
  {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      candidate,
      action: {
        operation: "file_write",
        target: "/home/user/my-project-workspace/ops/ledgers/2024-Q1.json"
      }
    })
  }
);
const result = await resp.json();
console.log("Should HIT project-ledger-write:", result.matched_rule_ids);
// 期望：包含 "project-ledger-write"
EOF

# 不应命中：写 src/ 下文件
# ... 同上，但 target: "/home/user/my-project-workspace/src/index.ts"
# ... 期望：不包含 "project-ledger-write"
```

## 第 7 步：编写项目边界报告

类似全局基线的 boundary-report.md，但重点是：

```markdown
# 项目规则 — 验证报告

## 项目信息

- UUID: <project-id>
- Workspace: `${SESSION_WORKSPACE}` (展开为实际工作区路径)
- 基线继承: global （全局规则自动继承）

## 规则与依据

| ID | 来源（AGENTS.md 节点） | 类型 | 验证 | 缺口 |
|---|---|---|---|---|
| project-ledger-write | Key Directories: ops/ledgers/ | hard | ✅ 在线导入+模拟 | 无 |
| project-config-prod | Important Files: config/prod.yaml | hard | ✅ 在线导入 | 未测试读操作 |
| project-schema-change | Code Conventions: schema changes | advisory | ✅ 离线解析 | 无（建议规则） |

## 验证结果

- 导入: ✅ valid=true
- project-ledger-write:
  - ✅ 写 ops/ledgers/2024-Q1.json → matched_rule_ids 包含，verdict=confirm
  - ✅ 写 src/index.ts → matched_rule_ids 不包含
- project-config-prod:
  - ✅ 写 config/prod.yaml → matched_rule_ids 包含，verdict=confirm
  - ⚠️ 读 config/prod.yaml → 未测试（需要在线）

## `${SESSION_WORKSPACE}` 展开

此项目规则已测试 `${SESSION_WORKSPACE}` 在以下 workspace 中正确展开：
- `/home/user/my-project-workspace` ✅
- `/var/data/project-instances/prod` ✅

## 与全局基线的关系

- ✅ 无重复 ID，不与全局规则冲突
- ✅ 项目规则仅用 `${SESSION_WORKSPACE}` 引用工作区路径
- 全局规则 (如 `global-secret-files`) 在此项目也生效

## 交付清单

- [ ] policy.yaml 通过导入验证
- [ ] 至少 2 条规则各有完整正反例模拟
- [ ] 所有 source 对象都有真实出处或标记为 null
- [ ] `${SESSION_WORKSPACE}` 只用在项目规则中
- [ ] 无与全局规则 ID 重复
- [ ] 无秘密信息在 quote 中
- [ ] 无 `...` 占位符或 TODO
```

## 第 8 步：交付

打包三份文件：

```bash
# 交付清单
project-policy.yaml           # 完整、可导入的 YAML
project-boundary-report.md    # 规则与验证报告
project-context.md            # 项目背景与 AGENTS.md 提取摘要（可选）
```

---

## 关键检查清单

✅ **AGENTS.md 映射**
- [ ] 从 AGENTS.md、CONTRIBUTING.md、README 中读取所有项目约定
- [ ] 映射到具体的 operation 和 target
- [ ] 明确判定（confirm/deny/log/advisory_only）和类型（hard/advisory）

✅ **${SESSION_WORKSPACE} 使用**
- [ ] 只在项目规则中使用（全局规则不用）
- [ ] 路径模式以 `${SESSION_WORKSPACE}` 开头，如 `${SESSION_WORKSPACE}/ops/ledgers/**`
- [ ] 在模拟测试中用实际工作区路径验证展开

✅ **与全局基线的分离**
- [ ] 无重复 ID（全局 + 项目 ID 集合不重叠）
- [ ] 项目规则不"升级"成全局规则
- [ ] 若编辑既有策略，保留其他规则和激活配置

✅ **验证完整性**
- [ ] 导入通过 `valid: true`
- [ ] 至少 2-3 条项目规则各有完整正反例模拟
- [ ] `${SESSION_WORKSPACE}` 展开验证
- [ ] 缺口明确标注

✅ **无编造**
- [ ] source 对象出处真实或标记 null
- [ ] 无秘密在 quote 中
- [ ] 无 `...` 占位符

---

## 参考资料

- `SKILL.md` — 完整工作流与约束
- `references/authoring.md` — 操作符详解
- `examples/project.yaml` — 项目规则与 `${SESSION_WORKSPACE}` 示例
- `docs/global-baseline-generation.md` — 全局基线指南（原理相通）
