# 全局基线从零生成 — 完整步骤

本指南适用于为 AMOS 编写**新的全局基线策略**（不是编辑既有策略或项目规则）。

## 第 0 步：环境准备与目标确认

1. **确认 AMOS 检出可用**
   - 本机 `$AMOS_ROOT` 或 `git clone` 访问 `project-amos` 仓库
   - 若使用在线 AMOS 服务，记录 `AMOS_PORT`、`AMOS_AGENT_TOKEN` 等环境变量

2. **明确输出目标**
   - 全局基线项目 ID 固定为 `global`
   - 交付物：`policy.yaml`（完整 YAML）+ `boundary-report.md`（规则依据与缺口说明）
   - 仅生成与导入验证；**不提交提案、不发布**

## 第 1 步：本机与系统盘点

创建 `inventory.md`，记录（仅供内部参考，不写进 policy.yaml）：

```markdown
# 环境盘点

## 系统
- OS: $(uname -s) $(uname -r)
- Arch: $(uname -m)
- Node: $(node --version 2>/dev/null || echo "absent")
- npm: $(npm --version 2>/dev/null || echo "absent")
- Git: $(git --version)

## 检测到的工具
- $(which rg 2>/dev/null && echo "ripgrep: present" || echo "ripgrep: absent")
- $(which docker 2>/dev/null && echo "Docker: present" || echo "Docker: absent")
- ... 其他与任务相关的工具

## 仓库结构
- 当前仓库：$(pwd)
- AMOS 检出：$AMOS_ROOT（若有）
- 基线位置：$([[ -f "$AMOS_ROOT/constraints/schema.json" ]] && echo "found" || echo "not found")

## 已有规则（若 AMOS 有已发布全局策略）
- $(curl -s http://127.0.0.1:$AMOS_PORT/api/projects/global/policy 2>/dev/null | jq '.rules | length' || echo "online service unavailable")
```

## 第 2 步：扫描 project-amos 仓库——提取真实约定

### 2a. 适配器与 operation 映射

读以下文件，确认**真实发出的 operation**：

```bash
# 核心适配器
grep -r "operation:" $AMOS_ROOT/adapters/pi/normalize.ts         # PI 适配器发出什么
grep -r "operation:" $AMOS_ROOT/adapters/opencode-v2/normalize.ts # OpenCode 发出什么
grep -r "'file_" $AMOS_ROOT/host/contracts.ts                     # 通用 operation 契约

# 结论记进 inventory.md
```

**规则**：只能在已确认发出的 operation 上写硬规则：
- ✅ `file_read`、`file_write`、`file_search`、`file_list`（普通）
- ✅ `file_mkdir`、`file_delete`（managed file host）
- ❌ `file_stat`、`file_perm`、`http_request` 等（当前不发出，不可用于硬规则）

### 2b. 现有全局规则（若有）

```bash
# 若有已发布全局策略
curl -s -H "Authorization: Bearer $AMOS_AGENT_TOKEN" \
  "http://127.0.0.1:$AMOS_PORT/api/projects/global/policy" | jq '.' > existing-global.json

# 若无在线服务，检查 AMOS 源树
grep -A 50 "rules:" $AMOS_ROOT/examples/global.yaml 2>/dev/null || echo "no example"
```

在 `boundary-report.md` 注明是否继承现有规则 ID（通常新建时为空，全部是新增）。

### 2c. 治理约定与风险清单

逐一读取，提取**跨仓库的**强制规则和建议：

```bash
# 核心文档
cat $AMOS_ROOT/CONTRIBUTING.md                    # 贡献指南，可能有秘密存储、CI 要求等
cat $AMOS_ROOT/README.md                          # 项目概览与使用规范
ls -la $AMOS_ROOT/.github/workflows/               # CI 流程（哪些操作在 CI 阶段需要审查）
grep -r "secret\|token\|key" $AMOS_ROOT/docs --include="*.md" | head -20  # 秘密管理指南

# 存储与同步约定
grep -r "\.env\|\.netrc\|\.ssh" $AMOS_ROOT --include="*.md" --include="*.ts" | head -10
```

**关键**：从这些文件提取的信息是 **source.quote** 的出处。若文件说"secrets go in `.env` and are gitignored"，这就是一条关于禁止 `file_write` 到 `.env` 的硬规则的依据。

### 2d. 目录结构与公私边界

```bash
# 敏感目录
ls -la $AMOS_ROOT/ | grep -E "^\." | awk '{print $NF}'  # .git, .github, .env* 等
find $AMOS_ROOT -maxdepth 2 -type d -name "private\|internal\|secret" 2>/dev/null

# 包与依赖
cat $AMOS_ROOT/package.json | jq '.scripts, .dependencies' | head -50
```

## 第 3 步：建立映射表

用 Markdown 表格记录每条规则的来源、动作、判定、例子：

```markdown
| ID | 依据文件 | operation | target | 判定 | 正例 | 反例 | 备注 |
|---|---|---|---|---|---|---|---|
| global-secret-files | CONTRIBUTING.md L42 | file_write | `.env*`, `.netrc`, `*.key` | deny | 写 `.env.local` 时确认 | 写 `config.json` | 只能agent确认，不阻止 |
| global-dotgit-read | README.md | file_read | `/.git/**` | log | 读 `.git/config` | 读 `.gitignore` | 建议记日志，不deny |
| global-ci-logs | .github/workflows/\*.yml | file_write | `/ci-logs/**`, `*.log` in temp | advisory_only | — | — | 未来 CI 可靠检测后升级 |
```

**重点**：
- 每条规则都要有 **source 出处**（文件 + 行号）
- operation 只选已确认的类型
- 判定（deny/confirm/log/advisory_only）要与仓库现状和人工流程一致
- 记下缺口或需人工审查的项

## 第 4 步：编写 policy.yaml

基于映射表，用 `examples/global.yaml` 作起点，填充：

```yaml
format: amos-project-policy
version: 1

activation:
  includedRuleIds: null  # 新建时 null 或明确列出新增规则 ID
  excludedRuleIds: []

injection:
  baselineRuleIds: []     # 若非继承现有全局，通常为空
  routes: []              # 全局规则通常无注入路由

rules:
  - id: global-secret-files
    dimension: "confidentiality"
    level: "critical"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args:
          operations: ["file_write"]
      - op: target_glob
        args:
          patterns: ["/.env*", "/.netrc", "/**/*.key"]
    
    narrative: |
      禁止写入秘密存储文件（密钥、令牌、连接字符串等）。
      所有秘密必须通过安全的环境变量或密钥管理服务注入。
      
    on_hit: "confirm"  # 若是 deny，改为 deny；通常从容地 confirm
    scope: "operation:file_write"
    ttl: null
    owner: "governance-team"
    
    source:
      repo: "project-amos"
      rev: "HEAD"
      path: "CONTRIBUTING.md"
      section: "Secret Management"
      lines: [42, 48]
      quote: "All secrets must be stored in .env files (never committed) or injected at runtime via secure environment variables or a secrets manager."
    
    probe: "fs_writeable"
    policy_class: "mandatory"
    enforcement: "confirm"  # 或 "runtime"、"ci"、"server"
  
  - id: global-git-internals
    # ... 
    predicate:
      - op: op_in
        args:
          operations: ["file_read"]
      - op: target_glob
        args:
          patterns: ["/.git/**"]
    enforcement: "log"
    # ...

  - id: global-ci-best-practices
    # 建议规则：无 predicate，只有 narrative
    predicate: null
    enforcement: "advisory"
    narrative: |
      CI 流程中的日志、临时文件、工作区清理应遵循以下规范：
      1. 不输出秘密到 stdout/stderr
      2. 临时目录在 CI 完成后清理
      3. ...
    source: null  # 建议规则通常没有单一出处
```

**14 个必填字段不能缺**：id、dimension、level、expressivity、predicate、narrative、on_hit、scope、ttl、owner、source、probe、policy_class、enforcement。

## 第 5 步：导入验证

### 在线验证（若有 AMOS 服务）

```bash
export DRAFT="./policy.yaml"
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"

# Step 1: Import（必须先做）
IMPORT_RESULT=$(node --input-type=module << 'EOF'
import fs from "fs";
const yaml = fs.readFileSync(process.env.DRAFT, "utf-8");
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/global/import`,
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

# 检查 valid: true
echo "$IMPORT_RESULT" | jq '.valid'  # 必须是 true
CANDIDATE=$(echo "$IMPORT_RESULT" | jq '.candidate')
```

### 离线验证（若无在线服务）

```bash
# 需要 AMOS 检出已构建
cd $AMOS_ROOT && npm ci && npm run build

node --input-type=module << 'EOF'
import { parsePolicyDocument } from "$AMOS_ROOT/dist/control-plane/document.js";
import fs from "fs";

const yaml = fs.readFileSync("./policy.yaml", "utf-8");
const doc = JSON.parse(require("js-yaml").dump(yaml)); // 或用 yaml 库解析
const parsed = parsePolicyDocument(doc);

console.log("Parsed rules:", parsed.rules.map(r => r.id));
console.log("Activation:", parsed.activation);
// 若无异常则通过
EOF
```

## 第 6 步：模拟测试（至少 2 条规则）

选 2 条你写的规则，各测试 1 个应命中 + 1 个不应命中的动作：

```bash
# 假设导入成功，取得 candidate 对象

# 规则 1: global-secret-files
# 应命中：写 .env.local
node --input-type=module << 'EOF'
const candidate = /* from import */;
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/global/simulate`,
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
        target: "/.env.local",
        content: "SECRET=..."
      }
    })
  }
);
const result = await resp.json();
console.log("Should HIT global-secret-files:", result.matched_rule_ids);
// 期望：matched_rule_ids 包含 "global-secret-files"
EOF

# 不应命中：写 config.json
node --input-type=module << 'EOF'
// 类似上面，但 target: "/config.json"
// 期望：matched_rule_ids 不包含 "global-secret-files"（可能包含其他规则或为空）
EOF
```

**记录结果**（每条规则测试）：
- ✅ 应命中 → 返回预期 rule ID
- ✅ 不应命中 → 未返回该 rule ID
- ❌ 一方失败 → 检查 predicate 和 target glob

## 第 7 步：编写 boundary-report.md

汇总所有规则、依据、验证结果、缺口：

```markdown
# 全局基线 — 验证报告

## 规则总结

| ID | 类型 | operation | 判定 | 验证 | 缺口 |
|---|---|---|---|---|---|
| global-secret-files | hard | file_write | confirm | ✅ 在线导入+模拟 | 无 |
| global-git-internals | hard | file_read | log | ✅ 在线导入 | 未模拟反例（git 操作需要真实 repo） |
| global-ci-best-practices | advisory | — | advisory_only | ✅ 离线解析 | 无（建议规则不可验证） |

## 验证结果

- 导入：✅ valid=true（AMOS 在线）
- global-secret-files：
  - ✅ 写 `.env.local` → matched_rule_ids 包含该规则，verdict=confirm
  - ✅ 写 `config.json` → matched_rule_ids 不包含该规则
- global-git-internals：
  - ✅ 读 `/.git/config` → matched_rule_ids 包含该规则，verdict=log
  - ✅ 读 `README.md` → matched_rule_ids 不包含该规则

## 已知缺口

1. HTTP 重定向检查（`target_host_in`）未运行模拟，需要在线 URL 调用
2. 文件权限规则（`file_perm`）当前 AMOS 适配器不发出，建议标记为 future baseline
3. Shell 命令检查（`shell_input`）不可作硬规则，仅作 advisory

## 跨仓库可用性

✅ 所有规则使用通用 glob 和 URL 模式，不硬编码特定仓库路径
✅ 未使用 `${SESSION_WORKSPACE}`（全局规则约束）
✅ 可应用于任意工作区项目

## 交付清单

- [ ] policy.yaml 通过导入验证
- [ ] 至少 2 条规则各有完整正反例模拟
- [ ] 所有 source 对象都有真实出处或标记为 null
- [ ] 无秘密信息在 quote 中
- [ ] 无 `...` 占位符或 TODO
- [ ] 准备就绪（但不提交提案）
```

## 第 8 步：交付

打包三份文件到输出目录：

```bash
# 交付清单
policy.yaml              # 完整、可导入的 YAML
boundary-report.md       # 规则与验证报告
inventory.md            # 环境与假设盘点（可选，仅供参考）
```

**严禁包含**：
- 秘密信息（token、密钥、API key）
- 完整的环境转储或历史日志
- `...` 占位符或未完成的规则
- 对未验证项的"已验证"声称

---

## 参考资料

- `SKILL.md` — 完整工作流与硬规则约束
- `references/authoring.md` — 操作符详解与验证脚本
- `examples/global.yaml` — 完整全局策略示例
- `examples/project.yaml` — 项目规则与 `${SESSION_WORKSPACE}` 示例
