# 工具编排系统 (Tool Orchestration)

## 概述

工具编排系统是 Codex 项目的核心组件，负责管理、路由和执行各种工具调用。该系统基于 Rust 的异步编程模型和 trait 系统，采用分层架构设计，提供灵活、安全且可扩展的工具管理能力。

## 架构设计

### 整体架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                    工具定义层 (spec.rs)                      │
│  - ToolSpec: 工具规范定义 (Function, LocalShell, etc.)      │
│  - JsonSchema: JSON Schema 子集用于参数验证                 │
│  - 工具构建函数: create_shell_tool(), create_apply_patch()  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    注册层 (registry.rs)                      │
│  - ToolRegistry: 工具处理器注册表                           │
│  - ToolHandler: 工具处理器 trait                            │
│  - ToolRegistryBuilder: 构建器模式                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    路由层 (router.rs)                        │
│  - ToolRouter: 工具调用路由和分发                           │
│  - ToolCall: 工具调用封装                                   │
│  - MCP 工具名称解析 (server/tool)                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    执行层 (handlers/)                        │
│  - ShellHandler: Shell 命令执行                             │
│  - ApplyPatchHandler: 文件编辑                              │
│  - McpHandler: MCP 工具调用                                 │
│  - UnifiedExecHandler: 交互式执行                           │
│  - 其他工具处理器                                           │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件

#### 1. 工具规范 (ToolSpec)

```rust
pub enum ToolSpec {
    Function(ResponsesApiTool),  // 标准函数工具
    LocalShell {},                // 本地Shell工具
    WebSearch {},                 // 网络搜索工具
    Freeform(FreeformTool),       // 自由格式工具
}
```

工具规范定义了：
- 工具名称和描述
- 参数的 JSON Schema
- 是否支持并行调用

#### 2. 工具处理器 (ToolHandler)

```rust
#[async_trait]
pub trait ToolHandler: Send + Sync {
    fn kind(&self) -> ToolKind;  // Function 或 Mcp
    fn matches_kind(&self, payload: &ToolPayload) -> bool;
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool;
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError>;
}
```

关键方法：
- `kind()`: 返回处理器类型
- `matches_kind()`: 检查负载类型是否匹配
- `is_mutating()`: 判断是否为修改性操作（需要等待工具门控）
- `handle()`: 执行工具调用的核心逻辑

#### 3. 工具调用流程

```
用户输入 → 模型响应 → 解析工具调用 → 路由分发 → 权限检查 → 执行处理 → 返回结果
```

详细步骤：
1. **构建工具调用**：从模型响应中解析 `ResponseItem`
2. **路由分发**：根据工具名称查找处理器
3. **权限检查**：通过 `is_mutating` 判断是否需要等待门控
4. **执行处理**：调用处理器的 `handle` 方法
5. **结果转换**：将输出转换为模型可理解的格式

## 关键特性

### 1. 并行支持

某些工具支持并行调用，通过 `supports_parallel_tool_calls` 标志控制：

```rust
builder.push_spec_with_parallel_support(create_grep_files_tool(), true);
```

支持并行的工具：
- `grep_files`
- `list_dir`
- `read_file`
- `test_sync_tool`
- `list_mcp_resources`
- `list_mcp_resource_templates`
- `read_mcp_resource`

### 2. 沙箱安全

通过 `SandboxPermissions` 控制命令执行权限：

```rust
pub enum SandboxPermissions {
    UseDefault,          // 使用默认沙箱策略
    RequireEscalated,    // 请求提升权限（需要用户批准）
}
```

沙箱类型：
- **macOS**: Seatbelt (`/usr/bin/sandbox-exec`)
- **Linux**: Landlock + Seccomp
- **Windows**: Restricted Token

### 3. MCP 集成

支持 Model Context Protocol 工具：

```rust
// 工具名称格式: server/tool_name
if let Some((server, tool)) = session.parse_mcp_tool_name(&name).await {
    // 调用 MCP 工具
}
```

MCP 特性：
- 动态工具发现
- OAuth/Bearer Token 认证
- Stdio/HTTP 传输支持
- 沙箱状态通知

### 4. 工具门控 (Tool Gate)

对于可能修改系统状态的工具，通过 `ReadinessFlag` 实现同步：

```rust
if handler.is_mutating(&invocation).await {
    tracing::trace!("waiting for tool gate");
    invocation.turn.tool_call_gate.wait_ready().await;
    tracing::trace!("tool gate released");
}
```

确保工具按顺序执行，避免竞争条件。

## 工具分类

### 内置工具

#### Shell 工具
- **`shell`**: 执行 shell 命令（数组参数）
- **`shell_command`**: 执行 shell 命令（字符串参数）
- **`local_shell`**: 本地 Shell 工具
- **`exec_command`**: 统一 exec（PTY 支持）
- **`write_stdin`**: 向交互式会话写入输入

#### 文件操作
- **`read_file`**: 读取文件（支持切片和缩进模式）
- **`list_dir`**: 列出目录内容
- **`grep_files`**: 文件内容搜索
- **`apply_patch`**: 文件编辑（自由格式或 JSON）

#### 其他工具
- **`view_image`**: 查看图像
- **`update_plan`**: 规划工具
- **`web_search`**: 网络搜索
- **`test_sync_tool`**: 测试同步工具

#### MCP 工具
- **`list_mcp_resources`**: 列出 MCP 资源
- **`list_mcp_resource_templates`**: 列出资源模板
- **`read_mcp_resource`**: 读取资源

### 动态工具

通过 MCP 协议动态加载外部工具，支持工具发现和调用。

## 配置驱动

工具可用性通过 `ToolsConfig` 和 `Features` 系统控制：

```rust
pub struct ToolsConfig {
    pub shell_type: ConfigShellToolType,
    pub apply_patch_tool_type: Option<ApplyPatchToolType>,
    pub web_search_request: bool,
    pub include_view_image_tool: bool,
    pub experimental_supported_tools: Vec<String>,
}
```

### 功能标志

```rust
pub enum Feature {
    ShellTool,           // Shell 工具
    UnifiedExec,         // 统一 exec
    ApplyPatchFreeform,  // 自由格式 apply_patch
    WebSearchRequest,    // 网络搜索
    ViewImageTool,       // 图像查看工具
    // ... 其他功能
}
```

### 模型类型决定工具集

不同模型有不同的默认工具集：
- **gpt-5-codex**: shell_command, apply_patch, view_image
- **codex-mini-latest**: local_shell, view_image
- **gpt-5**: shell, view_image

## 安全机制

### 1. 命令安全检查

```rust
fn is_known_safe_command(command: &[String]) -> bool {
    // 检查是否为已知安全命令
}
```

### 2. 执行策略

```rust
pub struct ExecPolicy {
    pub allowed_prefixes: Vec<String>,
    pub denied_prefixes: Vec<String>,
}
```

### 3. 审批策略

```rust
pub enum AskForApproval {
    Never,        // 从不请求批准
    OnRequest,    // 按需请求
    OnFailure,    // 仅在失败时
    UnlessTrusted, // 除非受信任
}
```

## 执行流程示例

### Shell 命令执行

```rust
// 1. 解析工具调用
let params: ShellCommandToolCallParams = serde_json::from_str(&arguments)?;

// 2. 转换为执行参数
let exec_params = ShellHandler::to_exec_params(params, session, turn);

// 3. 检查审批需求
let approval_req = session.services.exec_policy
    .create_exec_approval_requirement_for_command(...);

// 4. 创建工具上下文
let tool_ctx = ToolCtx { session, turn, call_id, tool_name };

// 5. 执行命令
let mut orchestrator = ToolOrchestrator::new();
let mut runtime = ShellRuntime::new();
let out = orchestrator.run(&mut runtime, &req, &tool_ctx, &turn, approval_policy).await;

// 6. 返回结果
Ok(ToolOutput::Function { content, success: Some(true) })
```

### MCP 工具调用

```rust
// 1. 解析 MCP 工具名称
if let Some((server, tool)) = session.parse_mcp_tool_name(&name).await {
    // 2. 调用 MCP 服务器
    let result = session.services.mcp_connection_manager
        .read().await
        .call_tool(server, tool, arguments).await;
    
    // 3. 返回结果
    Ok(ToolOutput::Function { content: result.content, ... })
}
```

### 交互式执行 (Unified Exec)

```rust
// 1. 创建或重用 PTY 会话
let session_id = UnifiedExecSessionManager::exec_command(...);

// 2. 执行命令
let mut runtime = UnifiedExecRuntime::new();
let out = orchestrator.run(&mut runtime, &req, &tool_ctx, &turn, approval_policy).await;

// 3. 返回会话 ID 和输出
Ok(ToolOutput::UnifiedExec { session_id, output })
```

## 错误处理

### 错误类型

```rust
pub enum FunctionCallError {
    Fatal(String),              // 致命错误
    RespondToModel(String),     // 可返回给模型的错误
}
```

### 错误传播

1. **工具执行错误**：通过 `ToolOutput::Function { success: Some(false) }` 返回
2. **解析错误**：返回 `RespondToModel`，提示模型修正参数
3. **系统错误**：返回 `Fatal`，终止当前任务

## 性能优化

### 1. 自动压缩

当令牌使用量超过限制时，自动压缩对话历史：
- 使用 `compact_conversation_history()` 端点
- 保留关键上下文，减少令牌消耗

### 2. 输出截断

```rust
pub struct TruncationPolicy {
    pub max_output_tokens: Option<usize>,
    pub max_lines: Option<usize>,
}
```

### 3. 门控机制

通过 `ReadinessFlag` 避免并发工具调用的竞争条件。

## 测试策略

### 单元测试

```rust
#[test]
fn test_shell_command_handler_to_exec_params() {
    // 测试参数转换
}

#[tokio::test]
async fn test_tool_execution() {
    // 测试异步工具执行
}
```

### 集成测试

- 模拟 MCP 服务器
- 沙箱环境测试
- 并行工具调用测试

### 快照测试

验证工具规范和输出格式的稳定性。

## 关键文件

| 文件 | 说明 |
|------|------|
| `core/src/tools/spec.rs` | 工具定义和 JSON Schema |
| `core/src/tools/registry.rs` | 工具注册和处理器管理 |
| `core/src/tools/router.rs` | 工具路由和分发 |
| `core/src/tools/handlers/` | 具体工具处理器实现 |
| `core/src/tools/runtimes/` | 工具运行时（沙箱、审批） |
| `core/src/tools/sandboxing.rs` | 沙箱和审批协调 |

## 总结

工具编排系统通过分层架构、trait 系统和异步编程模型，提供了：
- **灵活性**：支持多种工具类型和动态加载
- **安全性**：多层沙箱和审批机制
- **可扩展性**：易于添加新工具和处理器
- **性能**：并行支持、自动压缩和输出截断

该系统是 Codex 项目的核心，为所有工具调用提供了统一、安全且高效的执行环境。