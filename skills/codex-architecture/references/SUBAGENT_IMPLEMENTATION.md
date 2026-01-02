# 子代理实现逻辑详解

## 概述

子代理（SubAgent）是 Codex 项目中实现任务分解和并行处理的核心机制。它允许主代理创建专门的子代理来处理特定任务，如代码审查（review）和对话压缩（compact），从而提高整体效率和专注度。

## 核心组件

### 1. 协议定义 (`codex-rs/protocol/src/protocol.rs`)

```rust
pub enum SessionSource {
    Cli,
    VSCode,
    Exec,
    Mcp,
    SubAgent(SubAgentSource),  // 子代理会话源
    Unknown,
}

pub enum SubAgentSource {
    Review,      // 代码审查子代理
    Compact,     // 压缩子代理
    Other(String), // 自定义子代理类型
}
```

**作用**：
- 定义会话来源类型，区分不同类型的代理会话
- 通过 `SubAgentSource` 枚举支持不同类型的子代理任务
- 在 HTTP 请求中通过 `x-openai-subagent` 头部传递子代理类型信息

### 2. 子代理委托器 (`codex-rs/core/src/codex_delegate.rs`)

#### 交互式子代理会话 (`codex-rs/core/src/codex_delegate.rs`)

```rust
pub(crate) async fn run_codex_conversation_interactive(
    config: Config,
    auth_manager: Arc<AuthManager>,
    models_manager: Arc<ModelsManager>,
    parent_session: Arc<Session>,
    parent_ctx: Arc<TurnContext>,
    cancel_token: CancellationToken,
    initial_history: Option<InitialHistory>,
) -> Result<Codex, CodexErr>
```

**功能**：
- 创建独立的子代理 Codex 实例
- 设置双向通信通道（ops_tx 用于发送操作，events_rx 用于接收事件）
- 自动处理审批请求（exec/patch），不向消费者暴露
- 支持父会话取消传播

#### 一次性子代理会话 (`codex-rs/core/src/codex_delegate.rs`)

```rust
pub(crate) async fn run_codex_conversation_one_shot(
    config: Config,
    auth_manager: Arc<AuthManager>,
    models_manager: Arc<ModelsManager>,
    input: Vec<UserInput>,
    parent_session: Arc<Session>,
    parent_ctx: Arc<TurnContext>,
    cancel_token: CancellationToken,
    initial_history: Option<InitialHistory>,
) -> Result<Codex, CodexErr>
```

**特点**：
- 封装交互式会话，立即提交初始输入
- 自动桥接事件，在任务完成或中止时关闭
- 返回只读 Codex 实例，防止后续操作

### 3. 事件转发机制 (`codex-rs/core/src/codex_delegate.rs`)

```rust
async fn forward_events(
    codex: Arc<Codex>,
    tx_sub: Sender<Event>,
    parent_session: Arc<Session>,
    parent_ctx: Arc<TurnContext>,
    cancel_token: CancellationToken,
)
```

**处理逻辑**：
1. **过滤**：忽略 `AgentMessageDelta`、`AgentReasoningDelta`、`SessionConfigured` 等中间事件
2. **审批处理**：
   - `ExecApprovalRequest` → 调用父会话的 `request_command_approval`
   - `ApplyPatchApprovalRequest` → 调用父会话的 `request_patch_approval`
3. **转发**：其他事件通过通道发送给消费者

### 4. 任务特定实现

#### 代码审查任务 (`codex-rs/core/src/tasks/review.rs`)

```rust
pub(crate) struct ReviewTask;

impl ReviewTask {
    pub(crate) fn new() -> Self {
        Self
    }
}
```

**执行流程**：
1. **配置子代理**：
   - 清除用户指令（`user_instructions = None`）
   - 禁用项目文档加载（`project_doc_max_bytes = 0`）
   - 禁用 Web 搜索和图像查看工具
   - 设置审查专用提示词（`REVIEW_PROMPT`）
   - 使用专用审查模型

2. **启动子代理**：调用 `run_codex_conversation_one_shot`

3. **处理结果**：
   - 解析子代理返回的 JSON 格式审查输出
   - 生成结构化的 `ReviewOutputEvent`
   - 退出审查模式并记录结果

#### 压缩任务 (`codex-rs/core/src/tasks/compact.rs`)

```rust
pub(crate) struct CompactTask;
```

**执行流程**：
1. **判断执行方式**：
   - OpenAI 提供商 + 启用远程压缩特性 → 使用远程压缩
   - 否则使用本地压缩

2. **远程压缩** (`compact_remote.rs`)：
   - 直接调用 API 的压缩接口
   - 保留历史中的 `GhostSnapshot`

3. **本地压缩** (`compact.rs`)：
   - 使用 `SUMMARIZATION_PROMPT` 提示词
   - 通过流式 API 生成摘要
   - 构建新的历史记录：
     - 保留初始上下文
     - 添加最近的用户消息（受 token 限制）
     - 添加摘要后缀

### 5. HTTP 头部集成 (`codex-rs/core/src/client.rs`)

```rust
// client.rs
if let SessionSource::SubAgent(sub) = &self.session_source {
    let subagent = if let crate::protocol::SubAgentSource::Other(label) = sub {
        label.clone()
    } else {
        serde_json::to_value(sub)
            .ok()
            .and_then(|v| v.as_str().map(std::string::ToString::to_string))
            .unwrap_or_else(|| "other".to_string())
    };
    if let Ok(val) = HeaderValue::from_str(&subagent) {
        extra_headers.insert("x-openai-subagent", val);
    }
}
```

**作用**：向后端 API 标识子代理类型，用于统计和优化。

## 数据流架构

```
父会话
  ↓
创建子代理配置
  ↓
run_codex_conversation_one_shot
  ↓
┌─────────────────────────────────┐
│  子代理 Codex 实例               │
│  - 独立的配置                    │
│  - 独立的历史记录                │
│  - 独立的工具集                  │
└─────────────────────────────────┘
  ↓
事件流
  ↓
┌─────────────────────────────────┐
│  forward_events (过滤/转发)      │
│  - 过滤中间事件                  │
│  - 处理审批请求                  │
│  - 转发关键事件                  │
└─────────────────────────────────┘
  ↓
父会话接收处理结果
```

## 关键设计决策

### 1. 审批隔离
- 子代理的执行/补丁审批请求自动路由到父会话
- 用户无需感知子代理的存在
- 保持统一的审批体验

### 2. 配置隔离
- 子代理使用独立的配置副本
- 根据任务类型调整配置（如审查任务禁用某些工具）
- 避免配置污染

### 3. 事件过滤
- 中间事件（delta）被过滤，减少噪音
- 关键事件（完成、中止、错误）被保留
- 保持事件流的简洁性

### 4. 资源管理
- 使用 `CancellationToken` 支持取消传播
- 自动清理子代理资源
- 防止资源泄漏

## 使用场景

### 代码审查
```rust
// 在父会话中启动审查
let review_task = ReviewTask::new();
review_task.run(session, ctx, input, cancel_token).await;
```

### 对话压缩
```rust
// 自动压缩长对话
if should_use_remote_compact_task(session, provider) {
    run_remote_compact_task(session, ctx).await;
} else {
    run_compact_task(session, ctx, input).await;
}
```

## 测试验证

### 头部测试 (`codex-rs/core/tests/responses_headers.rs`)
验证子代理类型正确传递到 HTTP 头部：
- `SessionSource::SubAgent(SubAgentSource::Review)` → `x-openai-subagent: review`
- `SessionSource::SubAgent(SubAgentSource::Other("my-task"))` → `x-openai-subagent: my-task`

### 委托器测试 (`codex-rs/core/src/codex_delegate.rs`)
验证事件转发和取消处理：
- 取消时正确关闭子代理
- 审批请求正确路由
- 事件过滤逻辑正确

## 扩展性

### 添加新子代理类型
1. 在 `SubAgentSource` 中添加新枚举值
2. 创建对应的 `SessionTask` 实现
3. 在任务调度逻辑中注册新任务类型

### 自定义子代理
使用 `SubAgentSource::Other(String)` 支持动态子代理类型：
```rust
SessionSource::SubAgent(SubAgentSource::Other("custom-task".to_string()))
```

## 性能考虑

- **轻量级**：子代理共享父会话的认证和模型管理器
- **并行性**：多个子代理可以同时运行（通过独立的 Codex 实例）
- **资源效率**：使用通道而非回调，避免阻塞
- **取消响应**：CancellationToken 支持快速终止

## 总结

子代理系统通过：
1. **协议层**：定义标准化的子代理类型
2. **委托层**：管理子代理生命周期和通信
3. **任务层**：实现特定任务的业务逻辑
4. **集成层**：与现有系统无缝集成

实现了任务分解、并行处理和资源隔离，同时保持了用户体验的一致性。