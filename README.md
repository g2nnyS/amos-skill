# amos-skill

[中文](#中文) | [English](#english)

---

<a name="中文"></a>

# 中文

给 Agent 用的 [AMOS](https://github.com/g2nnyS/project-amos) 规则编写指南。产物是一份可导入控制台的完整 YAML 草案：全局基线写给内置 `global` 项目；项目补充规则写给实际工作区。审批和发布由人完成。

## 安装

### 选项 1：npm / npx（推荐）

通过 npm 包管理器快速安装：

```bash
# 使用 npx 一键运行（无需安装）
npx skills install @g2nnyS/amos-skill

# 或通过 npm 安装到 Claude Code skills 目录
npm install -g @g2nnyS/amos-skill
```

或手动复制到 skills 目录：
```bash
npm install @g2nnyS/amos-skill ~/.claude/skills/amos-skill
```

### 选项 2：Claude Code（Git 方式）

1. 克隆或下载本仓库：
   ```bash
   git clone https://github.com/g2nnyS/amos-skill.git
   ```

2. 将整个 `amos-skill` 目录放入 Claude Code 的 skills 目录：
   ```bash
   mkdir -p ~/.claude/skills
   cp -r amos-skill ~/.claude/skills/
   ```

3. 重启 Claude Code，该 skill 会自动注册为可用。

### 选项 3：通过 Skill Tool 直接使用

如果你的 Agent 客户端支持 `Skill` 工具，直接告诉 Agent：

> 使用 amos-skill 为 AMOS 编写规则。首先读取 `SKILL.md`，然后按指南生成 `policy.yaml`。

Agent 会自动加载本仓库的 `SKILL.md` 作为入口。

### 选项 4：本地编辑

无需 Agent 客户端集成，直接：

1. Clone 本仓库到本地
2. 读 [`SKILL.md`](SKILL.md) 理解工作流
3. 按 [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) 或 [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) 的步骤手工编写规则
4. 用 Node.js 验证脚本验证 YAML（见下文）

---

## 快速开始

### 任务 1：从零生成全局基线

**谁适合**：为 AMOS 编写跨仓库共通的治理规则。

**输入**：
- project-amos 仓库的检出（或在线 AMOS 服务）
- 本机环境信息

**输出**：
- `policy.yaml` — 完整、可导入的规则文档
- `boundary-report.md` — 规则依据、验证结果、缺口说明
- `inventory.md` — 环境盘点（可选）

**步骤**：
```bash
# 1. 告诉 Agent：为 AMOS 全局项目从零生成基线
# "为 AMOS 的全局项目编写一份新基线策略。
#  请参考 docs/global-baseline-generation.md 完整指南。
#  目标是 10+ 条规则，混合硬规则和建议规则，
#  每条都有完整的 source 出处或 null。
#  扫描 project-amos 仓库提取真实的 operation 映射和治理约定。
#  仅生成与导入验证，不提交提案。"

# 2. 或自己按指南手工生成，然后验证
export DRAFT="./policy.yaml"
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="global"

# 执行指南中的 Node.js 验证脚本
```

详见 [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md)（371 行完整指南，含环境设置、仓库扫描、映射表、导入验证、模拟测试、缺口说明）。

---

### 任务 2：从项目 AGENTS.md 生成项目级规则

**谁适合**：为某个具体项目编写补充规则，基于该项目的 AGENTS.md 和贡献指南。

**输入**：
- 目标项目的 UUID 或 `${SESSION_WORKSPACE}` 占位符
- 项目仓库的 AGENTS.md、CONTRIBUTING.md 等文档
- 项目的已发布策略（若编辑现有规则）

**输出**：
- `policy.yaml` — 完整、可导入的项目级规则
- `project-boundary-report.md` — 规则与全局基线的分工、`${SESSION_WORKSPACE}` 展开验证、缺口说明
- `project-context.md` — AGENTS.md 提取摘要（可选）

**步骤**：
```bash
# 1. 告诉 Agent：从项目 AGENTS.md 生成项目规则
# "为项目 UUID=<uuid> 编写补充规则。
#  请扫描该项目的 AGENTS.md、CONTRIBUTING.md、README 文档。
#  参考 docs/project-rules-from-agents.md 完整指南。
#  生成 5-8 条规则（硬规则 3-5、建议规则 2-3），
#  使用 \${SESSION_WORKSPACE} 引用工作区路径。
#  确保无与全局规则的 ID 重复。
#  仅生成与导入验证，不提交提案。"

# 2. 或自己按指南手工生成并验证
```

详见 [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md)（394 行完整指南，含项目文档扫描、`${SESSION_WORKSPACE}` 使用、全局基线分离、导入验证、模拟测试）。

---

## 完整工作流

### 1. 阅读入口文档

| 文件 | 用途 |
|------|------|
| [`SKILL.md`](SKILL.md) | **必读**。完整的工作流、约束、14 个必填字段、7 个算子、全局 vs 项目模型、人工发布边界 |
| [`references/authoring.md`](references/authoring.md) | 操作符详解、验证脚本、交付清单 |
| [`examples/global.yaml`](examples/global.yaml) | 全局基线模板 |
| [`examples/project.yaml`](examples/project.yaml) | 项目规则模板 |

### 2. 选择指南

- **生成全局基线**？→ [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md)
- **生成项目规则**？→ [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md)

### 3. 编写规则

按指南的步骤：
1. 扫描仓库文档、提取约定
2. 建立映射表（依据 → operation → 判定 → 类型）
3. 编写 YAML（所有 14 字段必填）
4. 导入验证（`POST /import` 或离线 `parsePolicyDocument`）
5. 模拟测试（至少 1-2 条规则各有完整正反例）
6. 编写边界报告（总结、验证结果、缺口）

### 4. 验证与交付

**在线验证**（需要运行的 AMOS 服务）：
```bash
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="global"
export DRAFT="./policy.yaml"

node --input-type=module << 'EOF'
import fs from "fs";
const yaml = fs.readFileSync(process.env.DRAFT, "utf-8");
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/import`,
  { method: "POST", headers: { "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`, "Content-Type": "text/yaml" }, body: yaml }
);
console.log(JSON.stringify(await resp.json(), null, 2));
EOF
```

**离线验证**（无需在线服务，需要 AMOS 检出）：
```bash
export AMOS_ROOT="/path/to/project-amos"
cd $AMOS_ROOT && npm ci && npm run build

node --input-type=module << 'EOF'
import { parsePolicyDocument } from "$AMOS_ROOT/dist/control-plane/document.js";
import YAML from "js-yaml";
import fs from "fs";

const yaml = fs.readFileSync("./policy.yaml", "utf-8");
const doc = YAML.load(yaml);
const parsed = parsePolicyDocument(doc);
console.log("Parsed rules:", parsed.rules.map(r => r.id));
EOF
```

**交付清单**：
- ✅ policy.yaml（完整 YAML，所有 14 字段，无占位符）
- ✅ boundary-report.md（规则总结、验证结果、缺口说明）
- ✅ 源文档（inventory.md 或 project-context.md）
- ✅ 验证证据（导入结果、模拟输出）

---

## 核心概念

### 硬规则 vs 建议规则

| 类型 | predicate | enforcement | 何时使用 |
|------|-----------|-------------|---------|
| **硬规则** | 非空数组 | `runtime`/`ci`/`server` | 确定的操作，可检验，需要拦截或确认 |
| **建议规则** | `null` | `advisory` | 工程规范、工作流建议、尚无检查点的约定 |

### 全局 vs 项目

| 维度 | 全局基线 | 项目规则 |
|------|--------|--------|
| **项目 ID** | `global` | 具体 UUID 或 `${SESSION_WORKSPACE}` |
| **路径** | 通用 glob（`/.git/**`） | `${SESSION_WORKSPACE}/...` |
| **信息源** | project-amos 仓库 + 本机环境 | 目标项目的 AGENTS.md |
| **规则数** | 10+ 条 | 5-8 条 |
| **约束** | 跨仓库可复用 | 项目专属 |
| **分离** | 独立 | 无与全局规则的 ID 重复 |

### ${SESSION_WORKSPACE}

工作区路径抽象（仅用于项目规则）：
- 在编写时保持可移植（如 `${SESSION_WORKSPACE}/data/**`）
- 在导入/模拟时按目标项目的实际工作区展开
- **不能用于全局规则**（全局 workspace 是 `/`）

---

## 完整示例

### 示例 1：全局基线中的一条规则

```yaml
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
  
  on_hit: "confirm"
  scope: "operation:file_write target:secrets"
  ttl: null
  owner: "governance-team"
  
  source:
    repo: "primary"
    rev: "HEAD"
    path: "CONTRIBUTING.md"
    section: "Secret Management"
    lines: [42, 48]
    quote: "All secrets must be stored in .env files (never committed) or injected at runtime via secure environment variables."
  
  probe: "fs_writeable"
  policy_class: "mandatory"
  enforcement: "confirm"
```

### 示例 2：项目规则中使用 ${SESSION_WORKSPACE}

```yaml
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
  
  narrative: |
    账本文件是此项目的关键数据存储，任何直接修改必须经过人工确认。
  
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
    quote: "ops/ledgers/ — project-critical account ledger; write-confirmed in dev"
  
  probe: "fs_writeable"
  policy_class: "mandatory"
  enforcement: "confirm"
```

在模拟时，`${SESSION_WORKSPACE}` 被展开为实际路径（如 `/home/user/my-project-workspace`）。

---

## 常见问题

### Q: 我能直接用示例覆盖已有策略吗？

**不能。** 示例是起点模板。若有既有策略，导出它用 `GET /api/projects/:id/versions/:n/export?format=yaml`，然后编辑。

### Q: 如何确保没有与全局规则 ID 重复？

在 `evals/global-write/` 评估中查看已有全局规则 ID，或在 AMOS 控制台查看 `GET /api/projects/global/policy`。项目规则 ID 避免使用 `global-*` 前缀，改用 `project-*` 或项目名前缀。

### Q: 我能拼接多个 YAML 文件吗？

**不能。** 每份 policy.yaml 必须是单独完整的文档。若需组合，用 `activation.includedRuleIds` 和 `injection.baselineRuleIds` 在同一文件内选择。

### Q: 没有在线 AMOS 服务怎么办？

用离线验证脚本（需要 AMOS 检出已构建）。见 `docs/global-baseline-generation.md` 或 `docs/project-rules-from-agents.md` 的"离线验证"部分。

### Q: 秘密信息怎么处理？

**绝不要**在 `source.quote` 中包含秘密。若必须引用（如"密钥文件 `.env`"），用文件名或关键词，不要包含值。导出的 `inventory.md` 也不输出 secret 值。

### Q: 我能提交提案吗？

**不能。** Agent 只能草稿、导入、验证、模拟。审批和发布是人工操作（`POST /api/proposals/:id/decision` 由人执行）。

---

## 资源

| 资源 | 描述 |
|------|------|
| [`SKILL.md`](SKILL.md) | 完整工作流与约束（入口文档） |
| [`AGENTS.md`](AGENTS.md) | 项目指南与文件说明 |
| [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) | 全局基线从零生成的 8 步完整指南 |
| [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) | 项目规则从 AGENTS.md 生成的 8 步完整指南 |
| [`references/authoring.md`](references/authoring.md) | 操作符参考、验证脚本、交付清单 |
| [`examples/global.yaml`](examples/global.yaml) | 全局基线模板 |
| [`examples/project.yaml`](examples/project.yaml) | 项目规则模板 |
| [`evals/global-write/`](evals/global-write/) | 全局基线评估案例与评分标准 |
| [`evals/project-ledgers/`](evals/project-ledgers/) | 项目规则评估案例与评分标准 |
| [AMOS 仓库](https://github.com/g2nnyS/project-amos) | 官方 AMOS 控制平面实现 |

---

## 许可

本 Skill 是 [project-amos](https://github.com/g2nnyS/project-amos) 的配套指南包。

遵循 MIT License（见 [LICENSE](LICENSE)）。

---

<a name="english"></a>

# English

AI Agent skill for authoring [AMOS](https://github.com/g2nnyS/project-amos) governance rules. The output is a complete, importable YAML policy draft: global baselines are written to the built-in `global` project; project-level supplementary rules apply to specific workspaces. Approval and publication are always human-only operations.

## Installation

### Option 1: npm / npx (Recommended)

Quick installation via npm package manager:

```bash
# One-command npx installation (no local install needed)
npx skills install @g2nnyS/amos-skill

# Or install globally
npm install -g @g2nnyS/amos-skill
```

Or install to skills directory manually:
```bash
npm install @g2nnyS/amos-skill ~/.claude/skills/amos-skill
```

### Option 2: Claude Code (Git)

1. Clone or download this repository:
   ```bash
   git clone https://github.com/g2nnyS/amos-skill.git
   ```

2. Place the `amos-skill` directory in Claude Code's skills folder:
   ```bash
   mkdir -p ~/.claude/skills
   cp -r amos-skill ~/.claude/skills/
   ```

3. Restart Claude Code—the skill will auto-register.

### Option 3: Skill Tool Direct Use

If your Agent client supports the `Skill` tool, tell the Agent directly:

> Use amos-skill to author AMOS rules. Start by reading `SKILL.md`, then follow the guides to generate `policy.yaml`.

The Agent will automatically load `SKILL.md` as the entry point.

### Option 4: Local Editing

No Agent client required:

1. Clone this repository locally
2. Read [`SKILL.md`](SKILL.md) to understand the workflow
3. Follow [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) or [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) to author rules manually
4. Use Node.js validation scripts to verify YAML (see below)

---

## Quick Start

### Task 1: Generate Global Baseline (10+ rules)

**Use when**: You need to author cross-repository governance rules for AMOS.

**Inputs**:
- project-amos repository checkout (or running AMOS service)
- Local environment info

**Outputs**:
- `policy.yaml` — complete, importable rule document
- `boundary-report.md` — rule justifications, verification results, gaps
- `inventory.md` — environment inventory (optional)

**Steps**:
```bash
# 1. Tell the Agent to generate a global baseline
# "Author a new global baseline policy for AMOS.
#  Reference docs/global-baseline-generation.md for complete guidance.
#  Target 10+ rules mixing hard rules and advisory rules,
#  each with complete source attribution or null.
#  Scan project-amos repository for real operation mappings and governance conventions.
#  Generate and validate imports only; do not submit proposals."

# 2. Or generate manually, then verify
export DRAFT="./policy.yaml"
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="global"

# Run Node.js validation script from docs/
```

See [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) (371 lines: environment setup, repo scanning, mapping tables, import validation, simulation testing, gap analysis).

---

### Task 2: Generate Project-Level Rules from AGENTS.md (5-8 rules)

**Use when**: You need project-specific supplementary rules based on a project's AGENTS.md and contribution guidelines.

**Inputs**:
- Target project UUID or `${SESSION_WORKSPACE}` placeholder
- Project AGENTS.md, CONTRIBUTING.md, README
- Published project policy (if editing existing rules)

**Outputs**:
- `policy.yaml` — complete, importable project rules
- `project-boundary-report.md` — division of labor with baseline, `${SESSION_WORKSPACE}` expansion verification, gaps
- `project-context.md` — AGENTS.md extraction summary (optional)

**Steps**:
```bash
# 1. Tell the Agent to generate project rules
# "Author supplementary rules for project UUID=<uuid>.
#  Scan the project's AGENTS.md, CONTRIBUTING.md, README documents.
#  Reference docs/project-rules-from-agents.md for complete guidance.
#  Generate 5-8 rules (3-5 hard, 2-3 advisory),
#  using ${SESSION_WORKSPACE} for workspace-specific paths.
#  Ensure no ID conflicts with global rules.
#  Generate and validate imports only; do not submit proposals."

# 2. Or generate manually, then verify
```

See [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) (394 lines: project doc scanning, `${SESSION_WORKSPACE}` usage, baseline separation, import validation, simulation testing).

---

## Full Workflow

### 1. Read Entry Documentation

| File | Purpose |
|------|---------|
| [`SKILL.md`](SKILL.md) | **Required reading.** Complete workflow, constraints, 14 required fields, 7 operators, global vs. project model, human approval boundary |
| [`references/authoring.md`](references/authoring.md) | Operator reference, validation scripts, delivery checklist |
| [`examples/global.yaml`](examples/global.yaml) | Global baseline template |
| [`examples/project.yaml`](examples/project.yaml) | Project rules template |

### 2. Choose Your Guide

- **Generating a global baseline?** → [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md)
- **Generating project rules?** → [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md)

### 3. Author Rules

Follow the guide's steps:
1. Scan repository docs, extract conventions
2. Build mapping table (source → operation → verdict → type)
3. Write YAML (all 14 fields required)
4. Validate import (`POST /import` or offline `parsePolicyDocument`)
5. Test simulation (≥1-2 rules, each with positive + negative cases)
6. Write boundary report (summary, results, gaps)

### 4. Verify and Deliver

**Online verification** (requires running AMOS service):
```bash
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="global"
export DRAFT="./policy.yaml"

node --input-type=module << 'EOF'
import fs from "fs";
const yaml = fs.readFileSync(process.env.DRAFT, "utf-8");
const resp = await fetch(
  `http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/import`,
  { method: "POST", headers: { "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`, "Content-Type": "text/yaml" }, body: yaml }
);
console.log(JSON.stringify(await resp.json(), null, 2));
EOF
```

**Offline verification** (no service needed; requires AMOS checkout built):
```bash
export AMOS_ROOT="/path/to/project-amos"
cd $AMOS_ROOT && npm ci && npm run build

node --input-type=module << 'EOF'
import { parsePolicyDocument } from "$AMOS_ROOT/dist/control-plane/document.js";
import YAML from "js-yaml";
import fs from "fs";

const yaml = fs.readFileSync("./policy.yaml", "utf-8");
const doc = YAML.load(yaml);
const parsed = parsePolicyDocument(doc);
console.log("Parsed rules:", parsed.rules.map(r => r.id));
EOF
```

**Delivery checklist**:
- ✅ policy.yaml (complete YAML, all 14 fields, no placeholders)
- ✅ boundary-report.md (rule summary, results, gaps)
- ✅ source docs (inventory.md or project-context.md)
- ✅ verification evidence (import & simulation output)

---

## Core Concepts

### Hard Rules vs. Advisory Rules

| Type | predicate | enforcement | Use when |
|------|-----------|-------------|----------|
| **Hard rule** | non-empty array | `runtime`/`ci`/`server` | Deterministic operations, verifiable, need interception or confirmation |
| **Advisory rule** | `null` | `advisory` | Engineering conventions, workflow suggestions, no executable checkpoint yet |

### Global vs. Project

| Dimension | Global Baseline | Project Rules |
|-----------|-----------------|---------------|
| **Project ID** | `global` | Specific UUID or `${SESSION_WORKSPACE}` |
| **Paths** | Universal globs (`/.git/**`) | `${SESSION_WORKSPACE}/...` |
| **Source** | project-amos repo + local env | Target project's AGENTS.md |
| **Rule count** | 10+ rules | 5-8 rules |
| **Scope** | Cross-repo reusable | Project-specific |
| **Separation** | Independent | No ID conflicts with global rules |

### ${SESSION_WORKSPACE}

Workspace path abstraction (project rules only):
- At authoring time: stays portable (e.g., `${SESSION_WORKSPACE}/data/**`)
- At verification time: expands to target project's actual workspace
- **Cannot be used in global rules** (global workspace is `/`)

---

## Full Examples

### Example 1: A Rule in Global Baseline

```yaml
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
    Prohibit writing to secret storage files (keys, tokens, connection strings).
    All secrets must be injected via secure environment variables or a secrets manager.
  
  on_hit: "confirm"
  scope: "operation:file_write target:secrets"
  ttl: null
  owner: "governance-team"
  
  source:
    repo: "primary"
    rev: "HEAD"
    path: "CONTRIBUTING.md"
    section: "Secret Management"
    lines: [42, 48]
    quote: "All secrets must be stored in .env files (never committed) or injected at runtime via secure environment variables."
  
  probe: "fs_writeable"
  policy_class: "mandatory"
  enforcement: "confirm"
```

### Example 2: Project Rule Using ${SESSION_WORKSPACE}

```yaml
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
  
  narrative: |
    Ledger files are project-critical data stores; any direct modification requires human confirmation.
  
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
    quote: "ops/ledgers/ — project-critical account ledger; write-confirmed in dev"
  
  probe: "fs_writeable"
  policy_class: "mandatory"
  enforcement: "confirm"
```

At simulation time, `${SESSION_WORKSPACE}` expands to the actual path (e.g., `/home/user/my-project-workspace`).

---

## FAQ

### Q: Can I directly use templates to overwrite existing policies?

**No.** Templates are starting points. If a policy exists, export it with `GET /api/projects/:id/versions/:n/export?format=yaml`, then edit.

### Q: How do I ensure no ID conflicts with global rules?

Check global rule IDs in `evals/global-write/` eval results, or query `GET /api/projects/global/policy` on the AMOS console. For project rules, avoid `global-*` prefix; use `project-*` or project-name prefix instead.

### Q: Can I concatenate multiple YAML files?

**No.** Each policy.yaml must be a single, complete document. For combining, use `activation.includedRuleIds` and `injection.baselineRuleIds` within one file.

### Q: What if I don't have a running AMOS service?

Use offline validation scripts (requires built AMOS checkout). See "Offline Verification" in `docs/global-baseline-generation.md` or `docs/project-rules-from-agents.md`.

### Q: How do I handle secrets?

**Never** include secrets in `source.quote`. If you must reference them (e.g., "key file `.env`"), use names or keywords—not values. Never export secret values in `inventory.md` either.

### Q: Can I submit proposals?

**No.** Agents can only draft, import, validate, and simulate. Approval and publication are human-only (`POST /api/proposals/:id/decision` requires human action).

---

## Resources

| Resource | Description |
|----------|-------------|
| [`SKILL.md`](SKILL.md) | Complete workflow & constraints (entry point) |
| [`AGENTS.md`](AGENTS.md) | Project guidelines & file reference |
| [`docs/global-baseline-generation.md`](docs/global-baseline-generation.md) | 8-step guide for generating global baseline from scratch |
| [`docs/project-rules-from-agents.md`](docs/project-rules-from-agents.md) | 8-step guide for generating project rules from AGENTS.md |
| [`references/authoring.md`](references/authoring.md) | Operator reference, validation scripts, checklist |
| [`examples/global.yaml`](examples/global.yaml) | Global baseline template |
| [`examples/project.yaml`](examples/project.yaml) | Project rules template |
| [`evals/global-write/`](evals/global-write/) | Global baseline evaluation case & graders |
| [`evals/project-ledgers/`](evals/project-ledgers/) | Project rules evaluation case & graders |
| [AMOS Repository](https://github.com/g2nnyS/project-amos) | Official AMOS control plane implementation |

---

## License

This Skill is a companion package to [project-amos](https://github.com/g2nnyS/project-amos).

Licensed under MIT (see [LICENSE](LICENSE)).
