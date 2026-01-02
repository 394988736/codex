# Codex 项目 Prompt、消息及工具调用结果组装发送全流程分析

## 概述

本文档详细分析了 Codex 项目中从用户输入到模型响应，再到工具调用和结果返回的完整流程。该流程涉及多个核心模块的协同工作，包括核心会话管理、模型客户端、工具系统、事件处理等。

## 核心组件

### 1. 核心会话管理 (`codex-rs/core/src/codex.rs`)

- **Codex**: 高级接口，提供 `submit()` 和 `next_event()` 方法
- **Session**: 会话状态管理，包含：
  - `SessionConfiguration`: 会话配置（模型、提供者、策略等）
  - `TurnContext`: 单次任务的上下文（工作目录、工具配置等）
  - `SessionServices`: 共享服务（MCP 连接、审批存储、遥测等）
- **run_task()**: 主要任务执行循环，处理模型响应和工具调用
- **run_turn()**: 单次模型调用，处理流式响应和并行工具调用

### 2. 模型客户端 (`codex-rs/core/src/client.rs`)

- **ModelClient**: 封装与后端 API 的通信
  - `stream()`: 根据提供者使用 Chat 或 Responses API 流式处理响应
  - `compact_conversation_history()`: 使用压缩端点压缩对话历史
- 支持 `WireApi::Chat` 和 `WireApi::Responses` 两种协议
- 处理认证刷新和错误映射

### 3. 工具系统 (`codex-rs/core/src/tools/`)

#### 3.1 工具定义 (`spec.rs`)
- `exec_command`: 执行命令（统一 exec 或 shell）
- `shell`: 执行 shell 命令（数组参数）
- `shell_command`: 执行 shell 命令（字符串参数）
- `apply_patch`: 文件编辑工具（自由格式或 JSON）
- `view_image`: 查看图像
- `grep_files`: 文件搜索
- `read_file`: 文件读取
- `plan`: 规划工具

#### 3.2 工具路由 (`router.rs`)
- 将工具调用分派到相应的处理器
- 处理 MCP 工具名称解析（`mcp__server__tool`）

#### 3.3 工具注册 (`registry.rs`)
- 注册工具处理器
- 处理工具调用的生命周期（日志、遥测、错误处理）

#### 3.4 工具处理器 (`handlers/`)
- `ShellHandler`/`ShellCommandHandler`: 执行 shell 命令
- `ApplyPatchHandler`: 处理文件编辑
- `McpHandler`: 调用 MCP 工具
- `UnifiedExecHandler`: 管理交互式 PTY 会话

#### 3.5 工具运行时 (`runtimes/`)
- `ShellRuntime`: 执行 shell 命令的运行时（处理沙箱和审批）
- `ApplyPatchRuntime`: 执行 apply_patch 的运行时
- `UnifiedExecRuntime`: 管理交互式 exec 会话的运行时

#### 3.6 沙箱和审批 (`sandboxing.rs`)
- `ToolOrchestrator`: 协调审批、沙箱选择和重试
- `SandboxAttempt`: 代表一次沙箱执行尝试
- `ExecApprovalRequirement`: 定义审批需求

### 4. 事件处理 (`codex-rs/core/src/stream_events_utils.rs`)

- **handle_output_item_done()**: 处理完成的输出项，记录并排队任何工具执行
- **handle_non_tool_response_item()**: 将消息/推理转换为 turn items
- **ToolEmitter**: 用于发出工具事件的枚举

### 5. 并行工具执行 (`codex-rs/core/src/tools/parallel.rs`)

- **ToolCallRuntime**: 管理工具调用的执行
- 支持并行和串行工具执行
- 处理工具调用的取消

## 完整流程分析

### 阶段 1: 用户输入到模型调用

#### 1.1 用户提交操作
```rust
// codex-rs/core/src/codex.rs
pub async fn submit(&self, op: Op) -> CodexResult<String> {
    let id = self.next_id.fetch_add(1, Ordering::SeqCst).to_string();
    let sub = Submission { id: id.clone(), op };
    self.submit_with_id(sub).await?;
    Ok(id)
}
```

#### 1.2 提交循环处理
```rust
// codex-rs/core/src/codex.rs
async fn submission_loop(sess: Arc<Session>, config: Arc<Config>, rx_sub: Receiver<Submission>) {
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::UserTurn { .. } | Op::UserInput { .. } => {
                handlers::user_input_or_turn(&sess, sub.id.clone(), sub.op, &mut previous_context).await;
            }
            // ... 其他操作处理
        }
    }
}
```

#### 1.3 创建轮次上下文
```rust
// codex-rs/core/src/codex.rs
pub async fn new_turn_with_sub_id(
    &self,
    sub_id: String,
    updates: SessionSettingsUpdate,
) -> ConstraintResult<Arc<TurnContext>> {
    // 应用配置更新
    let session_configuration = { /* 从状态获取并更新 */ };
    // 创建新的 TurnContext
    Ok(self.new_turn_from_configuration(sub_id, session_configuration, ...).await)
}
```

### 阶段 2: Prompt 构建与模型调用

#### 2.1 Prompt 构建
```rust
// codex-rs/core/src/client_common.rs
pub struct Prompt {
    pub input: Vec<ResponseItem>,           // 对话历史
    pub tools: Vec<ToolSpec>,               // 可用工具
    pub parallel_tool_calls: bool,          // 是否支持并行调用
    pub base_instructions_override: Option<String>, // 基础指令覆盖
    pub output_schema: Option<Value>,       // 输出模式
}

// 获取完整指令
pub fn get_full_instructions<'a>(&'a self, model: &'a ModelFamily) -> Cow<'a, str> {
    let base = self.base_instructions_override.as_deref().unwrap_or(model.base_instructions.deref());
    // 根据模型和工具存在性添加 apply_patch 指令
    if model.needs_special_apply_patch_instructions && !is_apply_patch_tool_present {
        Cow::Owned(format!("{base}\n{APPLY_PATCH_TOOL_INSTRUCTIONS}"))
    } else {
        Cow::Borrowed(base)
    }
}
```

#### 2.2 工具规范生成
```rust
// codex-rs/core/src/tools/spec.rs
pub fn create_tools_json_for_responses_api(tools: &[ToolSpec]) -> Result<Vec<serde_json::Value>> {
    // 将 ToolSpec 转换为 API 兼容的 JSON
}

pub fn create_tools_json_for_chat_completions_api(tools: &[ToolSpec]) -> Result<Vec<serde_json::Value>> {
    // 转换为 Chat Completions API 格式
}
```

#### 2.3 模型流式调用
```rust
// codex-rs/core/src/client.rs
pub async fn stream(&self, prompt: &Prompt) -> Result<ResponseStream> {
    match self.provider.wire_api {
        WireApi::Responses => self.stream_responses_api(prompt).await,
        WireApi::Chat => self.stream_chat_completions(prompt).await,
    }
}

// Responses API 实现
async fn stream_responses_api(&self, prompt: &Prompt) -> Result<ResponseStream> {
    let tools_json = create_tools_json_for_responses_api(&prompt.tools)?;
    let api_prompt = build_api_prompt(prompt, instructions, tools_json);
    // 调用 API 并返回流
}
```

### 阶段 3: 流式响应处理

#### 3.1 响应事件类型
```rust
// codex-api crate
pub enum ResponseEvent {
    Created,
    OutputItemAdded(ResponseItem),
    OutputItemDone(ResponseItem),
    OutputTextDelta(String),
    ReasoningSummaryDelta { delta: String, summary_index: usize },
    ReasoningContentDelta { delta: String, content_index: usize },
    RateLimits(RateLimitSnapshot),
    Completed { response_id: String, token_usage: Option<TokenUsage> },
}
```

#### 3.2 流处理循环
```rust
// codex-rs/core/src/codex.rs
async fn try_run_turn(...) -> CodexResult<TurnRunResult> {
    let mut stream = turn_context.client.clone().stream(prompt).await?;
    let tool_runtime = ToolCallRuntime::new(...);
    let mut in_flight: FuturesOrdered<BoxFuture<'static, CodexResult<ResponseInputItem>>> = FuturesOrdered::new();
    
    loop {
        match stream.next().await {
            Some(ResponseEvent::OutputItemDone(item)) => {
                // 处理完成的输出项
                let output_result = handle_output_item_done(&mut ctx, item, ...).await?;
                if let Some(tool_future) = output_result.tool_future {
                    in_flight.push_back(tool_future);
                }
            }
            Some(ResponseEvent::OutputItemAdded(item)) => {
                // 处理新增输出项
                if let Some(turn_item) = handle_non_tool_response_item(&item).await {
                    sess.emit_turn_item_started(&turn_context, &turn_item).await;
                }
            }
            Some(ResponseEvent::Completed { token_usage, .. }) => {
                // 更新令牌使用量并退出
                sess.update_token_usage_info(&turn_context, token_usage.as_ref()).await;
                break Ok(TurnRunResult { needs_follow_up, last_agent_message });
            }
            // ... 其他事件处理
        }
    }
    
    // 等待所有并行工具执行完成
    drain_in_flight(&mut in_flight, sess.clone(), turn_context.clone()).await?;
}
```

### 阶段 4: 工具调用识别与路由

#### 4.1 工具调用识别
```rust
// codex-rs/core/src/stream_events_utils.rs
pub async fn handle_output_item_done(
    ctx: &mut HandleOutputCtx,
    item: ResponseItem,
    previously_active_item: Option<TurnItem>,
) -> Result<OutputItemResult> {
    match ToolRouter::build_tool_call(ctx.sess.as_ref(), item.clone()).await {
        Ok(Some(call)) => {
            // 是工具调用
            ctx.sess.record_conversation_items(&ctx.turn_context, std::slice::from_ref(&item)).await;
            
            // 创建工具执行 future
            let tool_future: InFlightFuture<'static> = Box::pin(
                ctx.tool_runtime.clone().handle_tool_call(call, cancellation_token),
            );
            
            output.needs_follow_up = true;
            output.tool_future = Some(tool_future);
        }
        Ok(None) => {
            // 不是工具调用，处理为普通消息
            if let Some(turn_item) = handle_non_tool_response_item(&item).await {
                ctx.sess.emit_turn_item_completed(&ctx.turn_context, turn_item).await;
            }
            ctx.sess.record_conversation_items(&ctx.turn_context, std::slice::from_ref(&item)).await;
            output.last_agent_message = last_assistant_message_from_item(&item);
        }
        Err(err) => {
            // 错误处理
            // ...
        }
    }
}
```

#### 4.2 工具调用构建
```rust
// codex-rs/core/src/tools/router.rs
pub async fn build_tool_call(
    session: &Session,
    item: ResponseItem,
) -> Result<Option<ToolCall>, FunctionCallError> {
    match item {
        ResponseItem::FunctionCall { name, arguments, call_id, .. } => {
            if let Some((server, tool)) = session.parse_mcp_tool_name(&name).await {
                Ok(Some(ToolCall {
                    tool_name: name,
                    call_id,
                    payload: ToolPayload::Mcp { server, tool, raw_arguments: arguments },
                }))
            } else {
                Ok(Some(ToolCall {
                    tool_name: name,
                    call_id,
                    payload: ToolPayload::Function { arguments },
                }))
            }
        }
        ResponseItem::CustomToolCall { name, input, call_id, .. } => {
            Ok(Some(ToolCall {
                tool_name: name,
                call_id,
                payload: ToolPayload::Custom { input },
            }))
        }
        ResponseItem::LocalShellCall { id, call_id, action, .. } => {
            // 转换为 ShellToolCallParams
            // ...
        }
        _ => Ok(None),
    }
}
```

### 阶段 5: 工具执行与结果处理

#### 5.1 工具执行调度
```rust
// codex-rs/core/src/tools/parallel.rs
pub fn handle_tool_call(
    self,
    call: ToolCall,
    cancellation_token: CancellationToken,
) -> impl Future<Output = Result<ResponseInputItem, CodexErr>> {
    let supports_parallel = self.router.tool_supports_parallel(&call.tool_name);
    
    // 根据是否支持并行获取读锁或写锁
    let _guard = if supports_parallel {
        Either::Left(lock.read().await)
    } else {
        Either::Right(lock.write().await)
    };
    
    // 分派工具调用
    router.dispatch_tool_call(session, turn, tracker, call.clone()).await
}
```

#### 5.2 工具分派与处理
```rust
// codex-rs/core/src/tools/registry.rs
pub async fn dispatch(&self, invocation: ToolInvocation) -> Result<ResponseInputItem, FunctionCallError> {
    let handler = self.handler(tool_name.as_ref())
        .ok_or_else(|| FunctionCallError::RespondToModel(unsupported_tool_call_message(...)))?;
    
    // 检查负载类型匹配
    if !handler.matches_kind(&invocation.payload) {
        return Err(FunctionCallError::Fatal(...));
    }
    
    // 等待工具门控（用于串行执行）
    if handler.is_mutating(&invocation).await {
        invocation.turn.tool_call_gate.wait_ready().await;
    }
    
    // 执行工具
    let output = handler.handle(invocation).await?;
    
    // 转换为响应项
    Ok(output.into_response(&call_id_owned, &payload_for_response))
}
```

#### 5.3 Shell 工具处理示例
```rust
// codex-rs/core/src/tools/handlers/shell.rs
#[async_trait]
impl ToolHandler for ShellHandler {
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        let params: ShellToolCallParams = serde_json::from_str(&arguments)?;
        let exec_params = Self::to_exec_params(params, turn.as_ref());
        
        // 检查是否为 apply_patch 拦截
        if let Some(output) = intercept_apply_patch(...).await? {
            return Ok(output);
        }
        
        // 发出开始事件
        let emitter = ToolEmitter::shell(...);
        emitter.begin(event_ctx).await;
        
        // 创建执行请求
        let req = ShellRequest { ... };
        let mut orchestrator = ToolOrchestrator::new();
        let mut runtime = ShellRuntime::new();
        
        // 执行命令（包含审批和沙箱逻辑）
        let out = orchestrator.run(&mut runtime, &req, &tool_ctx, &turn, turn.approval_policy).await;
        
        // 发出结束事件并格式化输出
        let content = emitter.finish(event_ctx, out).await?;
        Ok(ToolOutput::Function { content, content_items: None, success: Some(true) })
    }
}
```

#### 5.4 沙箱与审批协调
```rust
// codex-rs/core/src/tools/sandboxing.rs
pub struct ToolOrchestrator;

impl ToolOrchestrator {
    pub async fn run(
        &mut self,
        runtime: &mut dyn ToolRuntime,
        req: &dyn ExecRequest,
        ctx: &ToolCtx,
        turn: &TurnContext,
        approval_policy: AskForApproval,
    ) -> Result<ExecToolCallOutput, ToolError> {
        // 1. 检查审批需求
        let approval_req = self.check_approval_requirement(req, ctx, turn, approval_policy).await?;
        
        // 2. 如果需要审批，请求用户
        if approval_req.requires_approval() {
            let decision = session.request_command_approval(...).await;
            if decision == ReviewDecision::Denied {
                return Err(ToolError::Rejected("rejected by user".to_string()));
            }
        }
        
        // 3. 选择沙箱类型
        let sandbox = SandboxManager::select_initial(...);
        
        // 4. 尝试执行
        let attempt = sandbox.env_for(req, ctx).await?;
        let result = runtime.execute(req, &attempt.env).await;
        
        // 5. 如果沙箱被拒绝且策略允许，重试无沙箱
        if let Err(ToolError::Codex(CodexErr::Sandbox(SandboxErr::Denied { .. }))) = &result {
            if self.should_retry_without_sandbox(turn) {
                let no_sandbox_attempt = SandboxAttempt::no_sandbox(req, ctx).await?;
                return runtime.execute(req, &no_sandbox_attempt.env).await;
            }
        }
        
        result
    }
}
```

### 阶段 6: 工具结果组装与返回

#### 6.1 工具输出转换
```rust
// codex-rs/core/src/tools/context.rs
impl ToolOutput {
    pub fn into_response(self, call_id: &str, payload: &ToolPayload) -> ResponseInputItem {
        match self {
            ToolOutput::Function { content, content_items, success } => {
                if matches!(payload, ToolPayload::Custom { .. }) {
                    ResponseInputItem::CustomToolCallOutput {
                        call_id: call_id.to_string(),
                        output: content,
                    }
                } else {
                    ResponseInputItem::FunctionCallOutput {
                        call_id: call_id.to_string(),
                        output: FunctionCallOutputPayload {
                            content,
                            content_items,
                            success,
                        },
                    }
                }
            }
            ToolOutput::Mcp { result } => {
                let output = match result {
                    Ok(call_tool_result) => FunctionCallOutputPayload::from(call_tool_result),
                    Err(err) => FunctionCallOutputPayload {
                        content: err,
                        success: Some(false),
                        ..Default::default()
                    },
                };
                ResponseInputItem::FunctionCallOutput {
                    call_id: call_id.to_string(),
                    output,
                }
            }
        }
    }
}
```

#### 6.2 事件发出
```rust
// codex-rs/core/src/tools/events.rs
impl ToolEmitter {
    pub async fn finish(
        &self,
        ctx: ToolEventCtx<'_>,
        out: Result<ExecToolCallOutput, ToolError>,
    ) -> Result<String, FunctionCallError> {
        let (event, result) = match out {
            Ok(output) => {
                let content = self.format_exec_output_for_model(&output, ctx);
                let event = ToolEventStage::Success(output);
                let result = if output.exit_code == 0 {
                    Ok(content)
                } else {
                    Err(FunctionCallError::RespondToModel(content))
                };
                (event, result)
            }
            Err(ToolError::Rejected(msg)) => {
                let normalized = if msg == "rejected by user" {
                    "exec command rejected by user".to_string()
                } else {
                    msg
                };
                let event = ToolEventStage::Failure(ToolEventFailure::Message(normalized.clone()));
                let result = Err(FunctionCallError::RespondToModel(normalized));
                (event, result)
            }
            // ... 其他错误处理
        };
        
        // 发出事件
        self.emit(ctx, event).await;
        result
    }
}
```

#### 6.3 结果记录到对话历史
```rust
// codex-rs/core/src/codex.rs
pub async fn record_conversation_items(
    &self,
    turn_context: &TurnContext,
    items: &[ResponseItem],
) {
    // 1. 添加到内存对话历史
    self.record_into_history(items, turn_context).await;
    
    // 2. 持久化到 rollout
    self.persist_rollout_response_items(items).await;
    
    // 3. 发送原始响应项给客户端
    self.send_raw_response_items(turn_context, items).await;
}
```

### 阶段 7: 并行工具执行完成与合并

#### 7.1 等待所有并行工具完成
```rust
// codex-rs/core/src/codex.rs
async fn drain_in_flight(
    in_flight: &mut FuturesOrdered<BoxFuture<'static, CodexResult<ResponseInputItem>>>,
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
) -> CodexResult<()> {
    while let Some(res) = in_flight.next().await {
        match res {
            Ok(response_input) => {
                // 记录工具输出到对话历史
                sess.record_conversation_items(&turn_context, &[response_input.into()]).await;
            }
            Err(err) => {
                error_or_panic(format!("in-flight tool future failed during drain: {err}"));
            }
        }
    }
    Ok(())
}
```

#### 7.2 轮次差异事件
```rust
// codex-rs/core/src/codex.rs
if should_emit_turn_diff {
    let unified_diff = {
        let mut tracker = turn_diff_tracker.lock().await;
        tracker.get_unified_diff()
    };
    if let Ok(Some(unified_diff)) = unified_diff {
        let msg = EventMsg::TurnDiff(TurnDiffEvent { unified_diff });
        sess.clone().send_event(&turn_context, msg).await;
    }
}
```

### 阶段 8: 循环继续或结束

#### 8.1 检查是否需要继续
```rust
// codex-rs/core/src/codex.rs
let TurnRunResult {
    needs_follow_up,
    last_agent_message: turn_last_agent_message,
} = turn_output;

if !needs_follow_up {
    last_agent_message = turn_last_agent_message;
    sess.notifier().notify(&UserNotification::AgentTurnComplete { ... });
    break; // 任务完成
}
continue; // 继续下一轮
```

#### 8.2 自动压缩检查
```rust
// codex-rs/core/src/codex.rs
let total_usage_tokens = sess.get_total_token_usage().await;
let token_limit_reached = total_usage_tokens >= auto_compact_limit;

if token_limit_reached && needs_follow_up {
    run_auto_compact(&sess, &turn_context).await;
    continue;
}
```

## 关键数据流

### 1. Prompt 构建数据流
```
用户输入 + 对话历史 + 工具规范 + 基础指令 → Prompt
    ↓
Prompt 转换为 API 格式 → API 请求
```

### 2. 响应处理数据流
```
API 响应流 → ResponseEvent
    ↓
OutputItemDone → ToolRouter::build_tool_call → ToolCall
    ↓
ToolCallRuntime::handle_tool_call → 工具执行
    ↓
ToolOutput → ResponseInputItem → 记录到历史
```

### 3. 工具执行数据流
```
ToolCall → ToolRouter::dispatch_tool_call → ToolHandler::handle
    ↓
ToolOrchestrator::run → 审批 + 沙箱 + 执行
    ↓
ExecToolCallOutput → ToolEmitter::finish → 格式化输出
    ↓
ToolOutput → ResponseInputItem
```

## 事件类型

### 主要事件类型
- **SessionConfiguredEvent**: 会话配置完成
- **TaskStartedEvent**: 任务开始
- **ItemStartedEvent** / **ItemCompletedEvent**: Turn item 生命周期
- **RawResponseItemEvent**: 原始响应项
- **AgentMessageContentDeltaEvent**: 助手消息增量
- **ReasoningContentDeltaEvent**: 推理内容增量
- **ExecCommandBeginEvent** / **ExecCommandEndEvent**: 命令执行
- **PatchApplyBeginEvent** / **PatchApplyEndEvent**: 补丁应用
- **ExecApprovalRequestEvent**: 执行审批请求
- **ApplyPatchApprovalRequestEvent**: 补丁审批请求
- **TokenCountEvent**: 令牌使用统计
- **TurnDiffEvent**: 轮次差异
- **ErrorEvent**: 错误事件
- **ShutdownComplete**: 关闭完成

## 错误处理

### 错误类型
- **CodexErr**: 集中式错误类型
- **FunctionCallError**: 工具调用错误
- **ToolError**: 工具运行时错误

### 错误传播
1. 工具执行错误 → ToolError → FunctionCallError
2. FunctionCallError → 转换为 ResponseInputItem
3. ResponseInputItem → 记录到对话历史
4. 错误事件 → 发送给客户端

## 性能优化

### 1. 并行工具执行
- 支持并行的工具使用读锁，串行工具使用写锁
- 使用 `FuturesOrdered` 保持结果顺序

### 2. 自动压缩
- 当令牌使用量超过限制时自动压缩对话历史
- 支持本地和远程压缩

### 3. 流式处理
- 使用异步流处理模型响应
- 增量处理和事件发出

## 安全考虑

### 1. 沙箱执行
- 平台特定沙箱（Seatbelt、Landlock、Windows Restricted Token）
- 沙箱策略配置

### 2. 审批策略
- `Never`: 从不审批
- `OnRequest`: 总是审批
- `OnFailure`: 失败时审批
- `UnlessTrusted`: 除非受信任

### 3. 命令安全检查
- `is_safe_command()`: 检查已知安全命令
- `ExecPolicy`: 基于前缀的命令执行策略

## 总结

Codex 项目的 prompt、消息及工具调用结果组装发送流程是一个复杂的异步系统，涉及多个层次的协同工作：

1. **输入层**: 用户输入 → 对话历史 → Prompt 构建
2. **模型层**: API 调用 → 流式响应 → 事件解析
3. **工具层**: 工具识别 → 路由 → 执行 → 结果处理
4. **输出层**: 结果组装 → 事件发出 → 历史记录

整个流程通过事件驱动、异步处理和并行执行来保证高效性和响应性，同时通过沙箱和审批机制确保安全性。