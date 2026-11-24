# AMOS 规则 YAML 参考

本文件补充 `SKILL.md` 的操作细节。机器契约以 AMOS 检出为准：

- `constraints/schema.json`：规则表 JSON Schema。
- `constraints/vocabulary.md`：七个算子及路径/URL/TTL 口径。
- `control-plane/document.ts`：完整 YAML 文档解析。
- `control-plane/compiler.ts`：候选、激活、注入与全局冲突检查。
- `docs/configuration-one-shot.md`、`docs/local-governance.md`：导入、提案、发布边界。
- 示例：`examples/global.yaml`、`examples/project.yaml`。

## 1. 文档与激活

导入接受旧式规则数组或完整文档。新草案用完整文档：

```yaml
format: amos-project-policy
version: 1
activation:
  includedRuleIds: null      # null = 全部已知；数组 = 只选这些
  excludedRuleIds: []
injection:
  baselineRuleIds: []
  routes: []
rules:
  - id: ...
    # 14 个规则字段
```

- 省略 `project` 块保持可移植；出现时 `id`/`workspace` 必须匹配目标。
- `published` 是导出派生信息，导入忽略，可删除。
- `includedRuleIds: null` 选中固定 catalog、当前全局层和项目规则；显式数组只选列出的 ID。未知、重复或与排除集重叠会失败。
- 项目候选只保存补充规则；同 ID 不同定义会被拒绝。
- 删除规则时同步清理 `includedRuleIds`、`excludedRuleIds`、`injection.baselineRuleIds` 和每条 route 的 `add`/`suppress`。

规则字段、硬/建议约束见 `SKILL.md` 与 `examples/`。`source` 可为 `null`；填对象时六个字段都必须非空，没有真实出处不要编造。

## 2. 动作与匹配

| op | args 的键（值为非空字符串数组） | 匹配条件 |
| --- | --- | --- |
| `op_in` | `operations` | operation 精确匹配，通常放第一条 |
| `target_glob` | `patterns` | 文件路径命中任一 glob |
| `target_outside_glob` | `patterns` | 文件路径在所有 glob 之外 |
| `url_query_key_in` | `keys` | URL 查询键匹配，不读取值 |
| `target_scheme_in` | `schemes` | 绝对 URL 的 scheme 命中 |
| `target_host_in` | `hosts` | 绝对 URL 的 hostname 命中 |
| `target_host_not_in` | `hosts` | 绝对 URL 的 hostname 不在列表中 |

- 当前普通适配器可发出的 operation：`file_read`、`file_write`、`file_search`、`file_list`；managed file host 另有 `file_mkdir`、`file_delete`。`shell_input` 只是不可评估的 coverage。
- `file_stat`、`file_rename`、`file_perm`、`file_link`、`file_truncate`、`http_request` 等可被 Schema 接受，但当前不发出；写在基线里也是惰性的，不能描述为已经拦截。
- 文件 target 必须是规范化绝对路径；不做 realpath、不展开 `~`。`*` 不跨 `/`，`**` 跨层级。
- `file_search`/`file_list` 的 target 是检索/列举根路径，不是每个结果文件。
- URL 规则必须先用 `op_in` 限定网络操作；否则文件 action 会让 URL 解析失败并导致整次评估 fail-closed。
- `${SESSION_WORKSPACE}` 只用于项目规则的 glob。全局项目自身 workspace 为 `/`，全局草案推荐不用它；实际组合/模拟按目标项目 workspace 展开。
- 无命中默认值由调用方决定，不要依赖“无命中即 allow”。多规则命中时按 `deny > confirm > log > allow`。
- `advisory_only` 不产生裁决，只进入 narrative 投影。`level` 不改变裁决；`scope` 只是审计元数据。

## 3. 只读调研

### 3.1 本机盘点（全局目标）

全局基线先了解运行环境，只读、只采样与任务相关的部分：

```bash
uname -s
uname -m
command -v node npm git rg docker python3 2>/dev/null || true
node --version 2>/dev/null || true
```

- skills：只列客户端 skill 根下的目录名，例如存在时 `ls -1 "$HOME/.claude/skills"`；不要批量读取 skill 内容。
- agent settings：只检查 hook、plugin、MCP 的文件名、顶层键、服务名和启用状态。**不要 `cat` 整个配置，不读取、输出或复制 `apiKey`、`token`、`env` 等 secret 值**；只记录键名和计数。
- 把 OS/arch、工具存在性、skills 名称、设置结构写进 `inventory.md`，标 present/absent。检测到工具只说明“已见”，不自动生成禁令。

### 3.2 仓库与固定目录

- 读当前仓库的 `AGENTS.md`、`CONTRIBUTING.md`、`README*`、包/构建清单，以及 `.github/`、`docs/`、`scripts/` 等固定目录中与任务相关的文件。
- 全局目标：这些只是共通约定样本，不等价于项目全部约定。只把跨仓库共通的路径/动作写进全局；项目专属目录、语言、发布流程写项目层，并写进 `boundary-report.md`。
- 项目目标：以已发布策略和该项目仓库约定为准，不把仓库数据自动当成已批准规则。

### 3.3 真实动作

读 AMOS 的 `adapters/pi/normalize.ts`、`adapters/opencode-v2/normalize.ts`、`host/contracts.ts`，确认 operation 和 target 形状；只有真实发出的 operation 才写成硬规则。

### 3.4 证据、采样与优先级

每条规则写成“依据 → operation + target → 判定/建议 → 正反例”。

- 只采样与任务相关的工具/路径/约定；其余记为“已见，不据此设规则”。
- 优先级：明确用户需求或已采用约定 > 仓库固定文件 > 机器探测；冲突时报告，不静默取更严。
- 已批准需求决定是否要规则；仓库文件只证明动作范围和风险。没有证据的约束写 `advisory_only`。

### 3.5 原稿与 seed

编辑指定草稿时读取该文件；修改已发布策略时：

- `GET /api/projects/:id/policy` 取当前版本；
- `GET /api/projects/:id/versions/:n/export?format=yaml` 取完整 YAML；
- `GET /api/seed` 只是仓库 seed candidate，不是全部已发布策略，不能替代当前版本。

### 3.6 硬规则选择流程

| 条件 | 结果 |
| --- | --- |
| 没有明确需求或已采用约定 | 只写 `advisory_only` |
| 不是跨仓库共通路径/动作 | 写项目层，并记录到 `boundary-report.md` |
| 不是当前真实 operation | 写建议 |
| 不能用窄 predicate 表达，或给不出正反例 | 写建议 |
| 以上都满足 | 写窄 `deny`/`confirm`/`log`，只覆盖明确列出的范围 |

探测到的工具、MCP、Shell、联网、进程执行不自动变成硬规则；这些通道优先写建议。不要用宽泛黑名单宣称覆盖完整；allow 列表只列探测并模拟验证过的路径。

## 4. 线上验证：Node 26 内存 fetch

只接受可信本机端口：固定 `http://127.0.0.1:<port>`，调用者只提供端口和项目 ID；不接受完整 URL、额外路径、查询串或用户信息；禁止重定向；token 只从环境读取，不打印、不写入文件或参数。

```bash
: "${AMOS_PORT:?local console port}"
: "${AMOS_AGENT_TOKEN:?agent bearer token from environment}"
: "${PROJECT_ID:?global or project id}"
: "${DRAFT:?path to full policy yaml}"
: "${ACTION_JSON:?JSON action object}"

AMOS_PORT="$AMOS_PORT" AMOS_AGENT_TOKEN="$AMOS_AGENT_TOKEN" \
PROJECT_ID="$PROJECT_ID" DRAFT="$DRAFT" ACTION_JSON="$ACTION_JSON" \
node --input-type=module <<'JS'
import { readFileSync } from 'node:fs';

const fail = (message) => { throw new Error(message); };
const port = Number(process.env.AMOS_PORT);
if (!Number.isInteger(port) || port < 1 || port > 65535) {
  fail('AMOS_PORT must be an integer from 1 to 65535');
}
const token = process.env.AMOS_AGENT_TOKEN;
if (typeof token !== 'string' || token.length === 0) fail('AMOS_AGENT_TOKEN is required');
const project = process.env.PROJECT_ID;
if (
  typeof project !== 'string' ||
  (project !== 'global' &&
    !/^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/.test(project))
) {
  fail('PROJECT_ID must be global or a UUID');
}
const draft = process.env.DRAFT;
if (typeof draft !== 'string' || draft.length === 0) fail('DRAFT is required');
let action;
try {
  action = JSON.parse(process.env.ACTION_JSON ?? '');
} catch {
  fail('ACTION_JSON must be valid JSON');
}
if (action === null || typeof action !== 'object' || Array.isArray(action)) {
  fail('ACTION_JSON must be an object');
}

const base = `http://127.0.0.1:${port}`;
const headers = {
  authorization: `Bearer ${token}`,
  'content-type': 'application/json',
};

async function post(pathname, body) {
  const response = await fetch(base + pathname, {
    method: 'POST',
    headers,
    body: JSON.stringify(body),
    redirect: 'error',
    signal: AbortSignal.timeout(10_000),
  });
  const data = await response.json();
  if (!response.ok) {
    fail(`${pathname} failed: HTTP ${response.status} ${JSON.stringify(data)}`);
  }
  return data;
}

const imported = await post(
  `/api/projects/${encodeURIComponent(project)}/import`,
  { yaml: readFileSync(draft, 'utf8') },
);

if (imported.valid !== true) {
  console.error(JSON.stringify({ phase: 'import', issues: imported.issues }, null, 2));
  process.exitCode = 1;
} else {
  const simulated = await post(
    `/api/projects/${encodeURIComponent(project)}/simulate`,
    { candidate: imported.candidate, action },
  );
  console.log(JSON.stringify({
    phase: 'simulate',
    verdict: simulated.verdict,
    matched_rule_ids: simulated.matched_rule_ids,
  }, null, 2));
}
JS
```

运行两次：一个应命中，一个不应命中。`imported.valid` 不为 `true` 时不会调用 simulate。

## 5. 离线验证：原生文档管线

先构建 AMOS 检出（`npm ci && npm run build`），再运行：

```bash
: "${AMOS_ROOT:?AMOS checkout}"
: "${DRAFT:?path to full policy yaml}"
: "${WORKSPACE:?normalized absolute workspace; use / for global}"

AMOS_ROOT="$AMOS_ROOT" DRAFT="$DRAFT" WORKSPACE="$WORKSPACE" \
node --input-type=module <<'JS'
import { readFileSync } from 'node:fs';
import { pathToFileURL } from 'node:url';

const root = process.env.AMOS_ROOT;
if (!root) throw new Error('AMOS_ROOT is required');
const workspace = process.env.WORKSPACE;
if (typeof workspace !== 'string' || !workspace.startsWith('/')) {
  throw new Error('WORKSPACE must be an absolute path');
}

const document = await import(
  pathToFileURL(`${root}/dist/control-plane/document.js`).href
);
const compiler = await import(
  pathToFileURL(`${root}/dist/control-plane/compiler.js`).href
);

const parsed = document.parsePolicyDocument(readFileSync(process.env.DRAFT, 'utf8'));
const validator = compiler.createRuleValidator(
  readFileSync(`${root}/constraints/schema.json`, 'utf8'),
);
const result = compiler.validateAndCompileCandidate(
  parsed.candidate,
  workspace,
  validator,
  null,
);

if (!result.result.valid) {
  console.error(JSON.stringify(result.result.issues, null, 2));
  process.exitCode = 1;
} else {
  console.log(`valid: ${result.compiled.effectiveRuleIds.length} effective rules`);
}
JS
```

这里调用的是导入同源的 `parsePolicyDocument`、`createRuleValidator` 和 `validateAndCompileCandidate`，会解析完整文档并按 `workspace` 展开项目 glob。

`external` 传 `null` 时，检查不会装载固定目录或已发布全局层；只在这些层存在的激活 ID 可能报未知，同名定义冲突也可能漏检。用目标项目的 `/import` 校验组合，再用 `/simulate` 验证动作。

## 6. 交付检查单

- [ ] 目标明确；编辑任务已取得草稿或已发布原稿。
- [ ] `inventory.md`、`policy.yaml`、`boundary-report.md` 同时存在；盘点不含 secret 值。
- [ ] 完整文档；无拼接、`...`、TODO。
- [ ] 新增 ID 已进入显式 include；删除 ID 已从 excluded、baseline、route add/suppress 清理。
- [ ] 未改规则保持原顺序与字段；`activation`、`injection` 未被重置。
- [ ] 硬规则只用当前真实 operation；Shell/HTTP/MCP/进程约束写成建议。
- [ ] hard rule 只覆盖明确要求/共通路径；项目专属约定留在项目层。
- [ ] allow 列表只列探测并验证过的路径；没有用宽黑名单宣称完整。
- [ ] 全局草案不含本机路径，推荐不用 `${SESSION_WORKSPACE}`。
- [ ] 线上或离线验证结果已记录；未运行项已标记。
- [ ] 未审批、未发布、未改数据库或激活产物。
