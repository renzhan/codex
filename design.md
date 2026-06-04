# Technical Design: Runtime Human Approval

## Overview

本设计基于 codex 原生的 `approval_policy` + `prefix rules`（execpolicy）机制，为 agent 运行时的 shell 命令执行添加人类审批能力。核心思路是：**不改变 skill 的执行方式**（仍然是 `python3 scripts/xxx.py` 的 shell 命令），而是通过 codex 的命令前缀匹配规则在运行时拦截高风险命令。

> **Codex 源码验证说明**：本设计已对照 codex-rs 源码验证，关键实现位于：
> - `codex-rs/execpolicy/` — prefix rule 解析与匹配引擎（Starlark DSL）
> - `codex-rs/core/src/exec_policy.rs` — rules 文件加载逻辑
> - `codex-rs/app-server-protocol/src/protocol/v2/item.rs` — 审批请求/响应协议
> - `codex-rs/protocol/src/protocol.rs` — `AskForApproval` 枚举定义

## Design Principles

1. **最小侵入**：不改变 skill 的执行方式，不要求 skill 改造为 MCP Server
2. **利用原生能力**：基于 codex 已有的 `approval_policy` + `.codex/rules/*.rules` execpolicy 机制
3. **双运行时兼容**：同时支持 agent-factory 的 HttpSseProtocol 和 aiop-agent-ide 的 CodexCLIClient
4. **向后兼容**：未配置审批策略的 agent 保持 `approvalPolicy: "never"` 行为不变

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Build Time                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Skill Developer                                                     │
│       │                                                              │
│       ▼                                                              │
│  tool-approvals.yaml                                                 │
│  ┌──────────────────────────────────────┐                           │
│  │ approvals:                            │                           │
│  │   - tool_name: shell                  │                           │
│  │     suggested_prefix_rules:           │                           │
│  │       - pattern: [python3, scripts/   │                           │
│  │           analyze_sku_overlap.py]     │                           │
│  │         decision: prompt              │                           │
│  │         reason: "读取生产数据库"       │                           │
│  │         risk_level: high              │                           │
│  └──────────────────────────────────────┘                           │
│                                                                      │
│  Agent Creator (Frontend)                                            │
│       │                                                              │
│       ▼                                                              │
│  Agent Approval Policy (DB)                                          │
│  ┌──────────────────────────────────────┐                           │
│  │ mode: rules-based                     │                           │
│  │ timeout: 30min                        │                           │
│  │ additional_rules: [...]               │                           │
│  │ auto_allow_rules: [...]               │                           │
│  └──────────────────────────────────────┘                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        Task Startup                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SandboxTaskService.create_and_stream()                              │
│       │                                                              │
│       ├─── 1. 读取 agent 的 approval_policy                         │
│       ├─── 2. 读取 agent 关联 skills 的 tool-approvals.yaml         │
│       ├─── 3. 合并生成 .codex/rules/default.rules → 写入 workdir  │
│       ├─── 4. 设置 approvalPolicy 参数为 "granular"              │
│       │       ({ rules: true, sandboxApproval: false, ... })      │
│       ▼                                                              │
│  Codex Runtime 启动                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        Runtime Approval Flow                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Codex 执行 agent turn                                               │
│       │                                                              │
│       ▼                                                              │
│  Agent 决定执行: python3 scripts/analyze_sku_overlap.py              │
│       │                                                              │
│       ▼                                                              │
│  Codex 匹配 .codex/rules/default.rules → decision="prompt"       │
│       │                                                              │
│       ▼                                                              │
│  Codex 暂停执行，发出审批 JSON-RPC request                           │
│  (method: "item/commandExecution/requestApproval")                   │
│       │                                                              │
│       ├─── Path A (HttpSseProtocol): SSE event → StreamEvent        │
│       │         │                                                    │
│       │         ▼                                                    │
│       │    agent-factory backend                                     │
│       │         │                                                    │
│       │         ├── 创建 ApprovalRequest 记录                        │
│       │         ├── 更新 SandboxTask status → 5 (PendingApproval)   │
│       │         ├── 推送 approval.required 事件给前端                 │
│       │         └── 等待审批决策...                                   │
│       │                                                              │
│       └─── Path B (CodexCLIClient): JSON-RPC request                │
│                 │  (需要 response 回复，不是 notification)            │
│                 │                                                    │
│                 ▼                                                    │
│            aiop-agent-ide backend                                    │
│                 │                                                    │
│                 ├── 推送 WebSocket 消息给前端                         │
│                 └── 等待审批决策...                                   │
│                                                                      │
│  Approver 在前端做出决策 (approve/reject)                             │
│       │                                                              │
│       ▼                                                              │
│  Backend 将决策回传给 Codex                                           │
│       │                                                              │
│       ├─── Path A: POST /codecli/agent/{task_id}/reply               │
│       └─── Path B: JSON-RPC response                                 │
│                                                                      │
│       ▼                                                              │
│  Codex 恢复执行（或跳过命令）                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Model

### 1. ApprovalPolicy（Agent 级别配置）

存储在 `agent_config` 表的 `config_data` JSON 字段中，新增 `approval_policy` key：

```json
{
  "approval_policy": {
    "enabled": true,
    "mode": "rules-based",
    "timeout_minutes": 30,
    "additional_rules": [
      {"pattern": ["rm", "-rf"], "decision": "prompt", "reason": "危险删除操作", "risk_level": "critical"}
    ],
    "auto_allow_rules": [
      {"pattern": ["python3", "scripts/query_*.py"]}
    ],
    "approvers": [123, 456]
  }
}
```

### 2. ApprovalRequest 表（新建）

```sql
CREATE TABLE approval_request (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       VARCHAR(64) NOT NULL,
    task_id         BIGINT NOT NULL REFERENCES sandbox_task(id),
    agent_id        BIGINT NOT NULL,
    thread_id       VARCHAR(128),
    
    -- 审批内容
    command         TEXT NOT NULL,
    matched_rule    TEXT,
    risk_level      VARCHAR(16) DEFAULT 'medium',
    reason          TEXT,
    
    -- 状态
    status          SMALLINT NOT NULL DEFAULT 1,  -- 1=pending, 2=approved, 3=rejected, 4=expired
    
    -- 决策信息
    decided_by      BIGINT,
    decided_at      TIMESTAMP WITH TIME ZONE,
    decision_reason TEXT,
    
    -- 审计
    requested_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    is_deleted      SMALLINT NOT NULL DEFAULT 0,
    
    -- 索引
    CONSTRAINT idx_approval_request_task UNIQUE (task_id, command, requested_at)
);

CREATE INDEX idx_approval_request_status ON approval_request(tenant_id, status);
CREATE INDEX idx_approval_request_agent ON approval_request(agent_id, requested_at);
```

### 3. tool-approvals.yaml 格式（Skill 构建产物）

```yaml
# tool-approvals.yaml — skill 构建时生成
version: "1.0"
approvals:
  - tool_name: shell
    approval_type: prefix_rule
    reason: "分析SKU重叠数据，会读取生产数据库"
    risk_level: high
    suggested_prefix_rules:
      - pattern: ["python3", "scripts/analyze_sku_overlap.py"]
        decision: prompt
      - pattern: ["python3", "scripts/delete_records.py"]
        decision: prompt
  - tool_name: shell
    approval_type: prefix_rule
    reason: "发送通知，会调用外部 API"
    risk_level: medium
    suggested_prefix_rules:
      - pattern: ["python3", "scripts/send_notification.py"]
        decision: prompt
```

### 4. .codex/rules/default.rules 文件格式（运行时生成）

> **重要**：Codex 的 execpolicy 使用 **Starlark DSL** 语法（类 Python），不是纯文本格式。
> 规则文件必须放在 `.codex/rules/` 目录下，扩展名为 `.rules`。
> Codex 加载时会扫描该目录下所有 `*.rules` 文件并合并解析。

```python
# .codex/rules/default.rules
# Auto-generated from tool-approvals.yaml + agent approval policy
# DO NOT EDIT MANUALLY

# From skill: sku-analysis
prefix_rule(pattern=["python3", "scripts/analyze_sku_overlap.py"], decision="prompt")
prefix_rule(pattern=["python3", "scripts/delete_records.py"], decision="prompt")

# From skill: notification
prefix_rule(pattern=["python3", "scripts/send_notification.py"], decision="prompt")

# From agent-level additional rules
prefix_rule(pattern=["rm", "-rf"], decision="forbidden")

# Auto-allow rules (override skill declarations)
prefix_rule(pattern=["python3", "scripts/query_safe.py"], decision="allow")
prefix_rule(pattern=["cat"], decision="allow")
prefix_rule(pattern=["ls"], decision="allow")
prefix_rule(pattern=["grep"], decision="allow")
```

> **注意 `BANNED_PREFIX_SUGGESTIONS`**：Codex 内置了一个禁止列表，不允许过于宽泛的 allow 规则。
> 例如 `prefix_rule(pattern=["python3"], decision="allow")` 会被拒绝，因为 `["python3"]` 在禁止列表中。
> 必须提供更具体的路径，如 `["python3", "scripts/query_safe.py"]`。
>
> 禁止列表包括：`python3`, `python`, `bash`, `sh`, `zsh`, `node`, `git`, `sudo`, `env` 等。

## Component Design

### Component 1: ApprovalRulesGenerator

**位置**: `agent-factory/backend/app/services/approval_rules.py`

负责将 `tool-approvals.yaml` + agent 级别策略合并生成 `.codex/rules` 文件内容。

```python
class ApprovalRulesGenerator:
    """将 skill 声明和 agent 策略合并为 codex execpolicy rules 文件（Starlark DSL）。"""

    def generate_rules_content(
        self,
        *,
        tool_approvals: list[dict],      # 从 tool-approvals.yaml 解析的规则
        agent_policy: dict | None,        # agent 级别的 approval_policy
    ) -> str:
        """生成 .codex/rules/default.rules 文件内容（Starlark 格式）。
        
        合并逻辑：
        1. 收集所有 skill 的 suggested_prefix_rules (decision=prompt)
        2. 追加 agent_policy.additional_rules
        3. 追加 agent_policy.auto_allow_rules (decision=allow)
        4. 去重
        5. 输出为 Starlark prefix_rule() 函数调用格式
        """

    def _format_prefix_rule(self, pattern: list[str], decision: str) -> str:
        """格式化单条规则为 Starlark 语法。
        
        Example:
            _format_prefix_rule(["python3", "scripts/foo.py"], "prompt")
            → 'prefix_rule(pattern=["python3", "scripts/foo.py"], decision="prompt")'
        """
        tokens = ", ".join(f'"{p}"' for p in pattern)
        return f'prefix_rule(pattern=[{tokens}], decision="{decision}")'

    def resolve_approval_policy_param(
        self, agent_policy: dict | None
    ) -> dict | str:
        """决定传给 codex 的 approvalPolicy 参数值。
        
        - agent_policy 为 None 或 enabled=False → "never"
        - mode="full-auto" → "never"
        - mode="rules-based" → granular config (推荐，仅 rules 触发审批)
        - mode="all-commands" → "on-request" (所有命令都需审批)
        
        推荐使用 granular 模式，只让 prefix rules 触发审批：
        {
            "sandboxApproval": false,
            "rules": true,
            "skillApproval": false,
            "requestPermissions": false,
            "mcpElicitations": false
        }
        """
```

### Component 2: 运行时配置注入

#### Path A: HttpSseProtocol（agent-factory sandbox task）

修改 `SandboxTaskService.create_and_stream()` 和 `_build_execute_kwargs()`：

```python
# sandbox_task.py 中新增逻辑

async def _prepare_approval_config(
    db: AsyncSession,
    *,
    agent_id: int,
    tenant_id: str,
) -> tuple[str | None, str]:
    """准备审批配置。
    
    Returns:
        (rules_content, approval_policy_param)
        - rules_content: .codex/rules 文件内容，None 表示不需要审批
        - approval_policy_param: 传给 codex 的 approvalPolicy 值
    """
    # 1. 读取 agent 的 approval_policy 配置
    agent_policy = await _load_agent_approval_policy(db, agent_id, tenant_id)
    
    # 2. 读取 agent 关联 skills 的 tool-approvals.yaml
    tool_approvals = await _load_skill_tool_approvals(db, agent_id, tenant_id)
    
    # 3. 生成 rules
    generator = ApprovalRulesGenerator()
    rules_content = generator.generate_rules_content(
        tool_approvals=tool_approvals,
        agent_policy=agent_policy,
    )
    approval_policy = generator.resolve_approval_policy_param(agent_policy)
    
    return rules_content, approval_policy
```

**问题**：HttpSseProtocol 通过 HTTP 调用中间层，无法直接写入 workdir 的 `.codex/rules/`。需要：
- 方案 A：将 rules 内容作为 form_data 参数传给中间层，由中间层写入
- 方案 B：将 rules 打包进 agent.zip 中（作为 `.codex/rules/default.rules` 文件）
- **推荐方案 B**：在打包 agent.zip 时将 `.codex/rules/default.rules` 文件包含进去，codex 启动时会自动扫描 workdir 下的 `.codex/rules/*.rules` 文件并加载

#### Path B: CodexCLIClient（aiop-agent-ide builder session）

修改 `_generate_mcp_config()` 或新增 `_generate_approval_rules()`：

```python
# ws_api.py 中新增

def _generate_approval_rules(workdir: str, rules_content: str) -> None:
    """将审批规则写入 workdir/.codex/rules/default.rules 文件。"""
    rules_dir = os.path.join(workdir, ".codex", "rules")
    os.makedirs(rules_dir, exist_ok=True)
    rules_path = os.path.join(rules_dir, "default.rules")
    with open(rules_path, "w") as f:
        f.write(rules_content)
```

修改 `CodexCLIClient.start_thread()` 和 `run_turn()`：

```python
# 将 approvalPolicy 从硬编码 "never" 改为参数化
async def start_thread(
    self,
    workdir: str,
    *,
    approval_policy: str | dict = "never",  # 新增参数，支持字符串或 granular dict
    ...
) -> str:
    result = await self._request(
        "thread/start",
        {
            "cwd": os.path.abspath(workdir),
            "approvalPolicy": approval_policy,  # 动态传入
            ...
        },
    )
```

> **推荐 `approvalPolicy` 值**：对于 rules-based 模式，使用 granular 配置：
> ```python
> approval_policy = {
>     "sandboxApproval": False,
>     "rules": True,          # 只有 prefix rules 匹配时才触发审批
>     "skillApproval": False,
>     "requestPermissions": False,
>     "mcpElicitations": False,
> }
> ```
> 这样只有 `.codex/rules/default.rules` 中 `decision="prompt"` 的规则匹配时才会暂停请求审批，
> 其他命令按照 sandbox 策略正常执行，不会弹出额外的审批请求。

### Component 3: 审批事件处理

> **关键澄清**：Codex 的审批机制使用的是 **JSON-RPC request**（需要 response 回复），
> 而不是 notification（单向通知）。method 为 `item/commandExecution/requestApproval`。
> Codex 发出 request 后会阻塞等待 response，收到 response 后才继续或中止执行。

#### Codex 审批请求协议（已验证）

```typescript
// Server → Client: JSON-RPC Request
// method: "item/commandExecution/requestApproval"
interface CommandExecutionRequestApprovalParams {
    threadId: string;
    turnId: string;
    itemId: string;
    approvalId?: string;       // 用于区分同一 item 的多个审批回调
    reason?: string;           // 审批原因说明
    command?: string;          // 待执行的命令
    cwd?: string;              // 命令工作目录
    commandActions?: CommandAction[];  // 解析后的命令动作（用于前端展示）
    proposedExecpolicyAmendment?: ExecPolicyAmendment;  // 建议的规则修改
}

// Client → Server: JSON-RPC Response
interface CommandExecutionRequestApprovalResponse {
    decision: CommandExecutionApprovalDecision;
}

// 决策选项
type CommandExecutionApprovalDecision =
    | "accept"                  // 批准执行
    | "acceptForSession"        // 批准并在本次会话中记住（后续相同命令不再询问）
    | { acceptWithExecpolicyAmendment: { execpolicyAmendment: ... } }  // 批准并持久化规则
    | "decline"                 // 拒绝（agent 继续 turn，跳过此命令）
    | "cancel";                 // 拒绝并中断整个 turn
```

#### Path A: HttpSseProtocol 事件处理

扩展 `StreamEvent` 模型和 `_handle_event` 逻辑：

```python
# schemas/third_party.py — 扩展 StreamEventType
class StreamEventType:
    ...
    APPROVAL_REQUIRED = "approval.required"
    APPROVAL_RESOLVED = "approval.resolved"

# 审批事件来自 codex 的 JSON-RPC request，中间层会将其转换为 SSE event。
# 中间层转发的 SSE event payload 结构：
# {
#   "type": "item/commandExecution/requestApproval",
#   "request_id": 42,           # JSON-RPC request id，回复时需要
#   "params": {
#     "threadId": "...",
#     "turnId": "...",
#     "itemId": "item-xxx",
#     "command": "python3 scripts/analyze_sku_overlap.py",
#     "cwd": "/workspace",
#     "reason": "approval required by policy rule",
#     "approvalId": "...",      # 可选
#   }
# }
```

在 `_stream_events` 中处理审批事件：

```python
async def _handle_approval_event(
    db: AsyncSession,
    event: StreamEvent,
    task_id: int,
    agent_id: int,
    tenant_id: str,
) -> None:
    """处理审批请求事件：创建 ApprovalRequest 记录，更新 task 状态。"""
    # 1. 创建 ApprovalRequest 记录
    # 2. 更新 SandboxTask status → 5 (PendingApproval)
    # 3. 事件会通过 SSE 推送给前端（在 _stream_events 的 yield 中）
```

#### Path B: CodexCLIClient 事件处理

在 JSON-RPC 消息处理中新增审批 request 分支：

```python
# codex_cli_client.py — 消息处理循环中

# 审批请求是 JSON-RPC request（有 "id" 字段），不是 notification
# 需要回复 JSON-RPC response 才能让 codex 继续执行
if method == "item/commandExecution/requestApproval":
    request_id = msg.get("id")  # JSON-RPC request id，必须用此 id 回复
    params = msg.get("params", {})
    approval_info = {
        "thread_id": params.get("threadId"),
        "turn_id": params.get("turnId"),
        "item_id": params.get("itemId"),
        "approval_id": params.get("approvalId"),
        "command": params.get("command"),
        "cwd": params.get("cwd"),
        "reason": params.get("reason"),
        "request_id": request_id,  # 保存用于后续回复
    }
    if on_activity:
        await on_activity({
            "method": "approval.required",
            "params": approval_info,
        })
    # 不自动回复，等待外部调用 respond_to_approval()
```

新增审批决策方法：

```python
class CodexCLIClient:
    async def respond_to_approval(
        self,
        *,
        request_id: int,
        decision: str,  # "approve" | "approve_session" | "reject" | "cancel"
    ) -> None:
        """回复审批请求（JSON-RPC response）。
        
        Codex 收到 response 后会根据 decision 继续或中止执行。
        
        Decision 映射：
        - "approve" → {"decision": "accept"}
        - "approve_session" → {"decision": "acceptForSession"}
        - "reject" → {"decision": "decline"}  (agent 继续 turn，跳过此命令)
        - "cancel" → {"decision": "cancel"}   (中断整个 turn)
        """
        decision_map = {
            "approve": "accept",
            "approve_session": "acceptForSession",
            "reject": "decline",
            "cancel": "cancel",
        }
        self._send({
            "jsonrpc": "2.0",
            "id": request_id,
            "result": {
                "decision": decision_map.get(decision, "decline"),
            },
        })
```

### Component 4: 审批 API Endpoints

**位置**: `agent-factory/backend/app/api/v1/endpoints/approval.py`

```python
@router.get("/approvals", response_model=ApprovalListOut)
async def list_approvals(
    status: int | None = None,  # 1=pending, 2=approved, 3=rejected, 4=expired
    agent_id: int | None = None,
    task_id: int | None = None,
    skip: int = 0,
    limit: int = 20,
):
    """查询审批请求列表。"""

@router.post("/approvals/{approval_id}/approve")
async def approve_request(
    approval_id: int,
    body: ApprovalDecisionIn,  # { reason?: string }
):
    """批准审批请求。"""
    # 1. 验证权限
    # 2. 更新 ApprovalRequest 状态
    # 3. 将决策回传给 codex runtime
    # 4. 更新 SandboxTask status → 2 (Running)

@router.post("/approvals/{approval_id}/reject")
async def reject_request(
    approval_id: int,
    body: ApprovalDecisionIn,
):
    """拒绝审批请求。"""

@router.post("/approvals/batch")
async def batch_decision(
    body: BatchApprovalIn,  # { ids: [1,2,3], decision: "approve"|"reject", reason?: string }
):
    """批量审批。"""
```

### Component 5: 审批超时处理

**位置**: `agent-factory/backend/app/services/approval_timeout.py`

使用定时任务（或 Redis 延迟队列）检查超时的审批请求：

```python
class ApprovalTimeoutChecker:
    """定期检查超时的审批请求，自动拒绝并通知。"""

    async def check_expired(self) -> None:
        """扫描超时的 pending 审批请求。
        
        执行逻辑：
        1. 查询 status=1 且 requested_at + timeout < now() 的记录
        2. 批量更新为 status=4 (expired)
        3. 对每个过期请求，发送 reject 决策给 codex
        4. 通知 agent creator
        """
```

### Component 6: Agent Zip 打包增强

**位置**: 修改现有的 agent zip 打包逻辑

在 `_load_agent_files()` 或 zip 构建过程中，将 `.codex/rules/default.rules` 文件注入到 zip 包中：

```python
async def _inject_approval_rules_to_zip(
    zip_bytes: bytes,
    rules_content: str,
) -> bytes:
    """将 .codex/rules/default.rules 文件注入到 agent zip 包中。
    
    codex 解压 zip 后会自动扫描 workdir/.codex/rules/ 目录下的 *.rules 文件并加载。
    规则文件使用 Starlark DSL 语法。
    """
    import io, zipfile
    
    buf = io.BytesIO(zip_bytes)
    with zipfile.ZipFile(buf, 'a') as zf:
        zf.writestr(".codex/rules/default.rules", rules_content)
    return buf.getvalue()
```

## Sequence Diagram: 完整审批流程

```
Agent Creator          Frontend           Backend              Codex Runtime
     │                    │                   │                      │
     │  配置 approval     │                   │                      │
     │  policy            │                   │                      │
     ├───────────────────►│                   │                      │
     │                    │  save policy      │                      │
     │                    ├──────────────────►│                      │
     │                    │                   │                      │
     │                    │                   │                      │
User │  发起任务          │                   │                      │
     ├───────────────────►│                   │                      │
     │                    │  create_and_stream│                      │
     │                    ├──────────────────►│                      │
     │                    │                   │  1. load policy      │
     │                    │                   │  2. load tool-approvals
     │                    │                   │  3. generate rules   │
     │                    │                   │  4. inject into zip  │
     │                    │                   │  5. set approvalPolicy=granular    │
│                    │                   │     { rules: true }                │
     │                    │                   │                      │
     │                    │                   │  execute_stream()    │
     │                    │                   ├─────────────────────►│
     │                    │                   │                      │
     │                    │                   │  ... agent working...│
     │                    │                   │                      │
     │                    │                   │  agent wants to run: │
     │                    │                   │  python3 scripts/    │
     │                    │                   │  analyze_sku.py      │
     │                    │                   │                      │
     │                    │                   │  matches rule →      │
     │                    │                   │  PAUSE               │
     │                    │                   │◄─────────────────────┤
     │                    │                   │  JSON-RPC request:   │
     │                    │                   │  item/commandExecution│
     │                    │                   │  /requestApproval    │
     │                    │                   │                      │
     │                    │  create           │                      │
     │                    │  ApprovalRequest  │                      │
     │                    │  update task→5    │                      │
     │                    │                   │                      │
     │                    │◄─────────────────┤│                      │
     │                    │  SSE: approval.   │                      │
     │                    │  required         │                      │
     │                    │                   │                      │
Approver                  │                   │                      │
     │  approve           │                   │                      │
     ├───────────────────►│                   │                      │
     │                    │  POST /approve    │                      │
     │                    ├──────────────────►│                      │
     │                    │                   │  update record       │
     │                    │                   │  update task→2       │
     │                    │                   │                      │
     │                    │                   │  JSON-RPC response   │
     │                    │                   │  {decision: "accept"}│
     │                    │                   ├─────────────────────►│
     │                    │                   │                      │
     │                    │                   │  RESUME execution    │
     │                    │                   │◄─────────────────────┤
     │                    │                   │  ... continues ...   │
```

## Key Technical Decisions

### 1. 为什么用 `.codex/rules` 而不是 MCP tool approval

当前 skill 是普通 Python 脚本通过 shell 执行，不是 MCP Server。改造成 MCP Server 成本高且不必要。Codex 的 execpolicy prefix rules 天然支持 shell 命令级别的审批拦截，且已经是 codex 的稳定功能。规则文件使用 Starlark DSL 语法，支持 `allow`、`prompt`、`forbidden` 三种 decision。

### 2. 为什么将 rules 注入 zip 而不是通过 API 参数传递

HttpSseProtocol 通过 HTTP 调用中间层 `/codecli/agent/runs`，中间层的接口参数有限。将 `.codex/rules/default.rules` 打包进 agent.zip 是最简单的方式——codex 解压后自动扫描 workdir 下的 `.codex/rules/*.rules` 文件并加载，无需中间层做任何改动。

### 3. 审批决策如何回传给 codex

Codex 的审批机制是 **JSON-RPC request/response** 模式（不是 notification）：
- Codex 发出 `item/commandExecution/requestApproval` request 后会阻塞等待
- 客户端必须回复一个带有相同 `id` 的 JSON-RPC response
- response 中的 `decision` 字段决定后续行为

具体路径：
- **HttpSseProtocol 路径**：通过 reply endpoint (`/codecli/agent/{task_id}/reply`) 发送 JSON-RPC response。需要确认中间层是否支持在 request 等待期间接收 reply。
- **CodexCLIClient 路径**：通过 stdin 直接写入 JSON-RPC response（与现有的 `mcpServer/elicitation/request` 处理方式一致）。

### 4. 为什么推荐 Granular approval policy 而非 on-request

`AskForApproval` 支持以下值：
- `"never"` — 从不请求审批（当前默认）
- `"on-request"` — 模型决定何时请求审批（过于宽泛，会导致非 rules 匹配的命令也可能触发审批）
- `"untrusted"` — 所有非已知安全命令都需审批（过于严格）
- `"granular"` — 细粒度控制（**推荐**）

Granular 模式允许精确控制哪些类型的审批会触发：
```json
{
  "sandboxApproval": false,      // 不触发 sandbox 相关审批
  "rules": true,                 // 只有 prefix rules 匹配 decision="prompt" 时触发
  "skillApproval": false,        // 不触发 skill 审批
  "requestPermissions": false,   // 不触发权限请求审批
  "mcpElicitations": false       // 不触发 MCP elicitation 审批
}
```
这样可以确保只有我们在 `.codex/rules/default.rules` 中声明的 `decision="prompt"` 规则才会触发审批，其他命令正常执行。

### 5. 审批期间 SSE 连接管理

HttpSseProtocol 的 SSE 连接在审批期间需要保持打开。Codex 在发出 JSON-RPC request 后会阻塞等待 response，SSE 流不会关闭。如果连接超时断开：
- ApprovalRequest 已持久化到数据库
- 审批决策通过独立的 API 提交
- 决策提交后通过新的 reply 请求恢复 codex 执行
- 需要确认中间层是否支持长时间保持连接（可能需要心跳机制）

### 6. 与现有 PendingApproval(5) 状态的关系

现有的 `SandboxTask.task_status = 5` 是任务级别的审批（整个任务需要审批才能开始执行）。运行时审批是命令级别的（任务执行过程中某个命令需要审批）。两者复用同一个状态值，但语义不同：
- 任务级审批：task 创建后直接进入 status=5，approve 后才开始执行
- 运行时审批：task 执行过程中从 status=2 切换到 status=5，approve 后恢复执行

建议：保持复用 status=5，通过 ApprovalRequest 表区分是哪种审批。

## Migration Plan

1. **Phase 1**: 数据模型 + API（创建 approval_request 表，实现 CRUD 和 API endpoints）
2. **Phase 2**: Rules 生成（实现 ApprovalRulesGenerator，修改 zip 打包逻辑注入 rules）
3. **Phase 3**: 运行时集成（修改 approvalPolicy 参数，处理审批事件，实现决策回传）
4. **Phase 4**: 前端集成（连接真实数据，实现审批操作 UI）
5. **Phase 5**: 超时处理 + 权限控制

## Open Questions

1. **中间层 reply 机制**：HttpSseProtocol 的中间层 `/codecli/agent/{task_id}/reply` 是否支持在 JSON-RPC request 等待期间接收审批决策 response？需要确认中间层的实现。
2. ~~**Codex 审批 notification 的具体 method 名**~~：**已确认**。Codex 使用 JSON-RPC request（不是 notification），method 为 `item/commandExecution/requestApproval`，参数结构为 `CommandExecutionRequestApprovalParams`。
3. **SSE 连接超时**：审批可能需要等待数分钟甚至数十分钟，HTTP SSE 连接是否会超时？是否需要心跳或重连机制？注意 Codex 在等待 response 期间不会主动关闭连接。
4. **批量命令审批**：Codex 是逐个命令暂停的——每个匹配 `decision="prompt"` 的命令都会独立发出一个 `requestApproval` request，等待 response 后才继续执行下一个命令。不支持批量收集后一次性审批。
5. **`acceptForSession` 的影响**：如果前端选择 `acceptForSession`，Codex 会在当前会话中缓存该决策，后续相同命令不再询问。需要评估这是否符合安全策略要求。
6. **Granular policy 的 app-server 协议支持**：需要确认 HttpSseProtocol 中间层是否支持传递 granular 格式的 `approvalPolicy`（对象而非字符串）。如果不支持，可以退而使用 `"on-request"`，但会导致更多非预期的审批请求。
haha
