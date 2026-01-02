# apply_patch 工具实现详解

## 概述

`apply_patch` 是 Codex 项目中的核心文件编辑工具，支持自由格式和 JSON 格式的补丁应用。本文档详细分析其实现过程，包括定义、解析、验证、执行和完成的完整流程。

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    工具定义层                                │
│  - ToolSpec: 定义 apply_patch 工具规范                      │
│  - Freeform/Function 两种格式                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    注册层                                    │
│  - ToolRegistry: 注册 apply_patch 处理器                   │
│  - ApplyPatchHandler: 工具处理器实现                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    解析验证层                                │
│  - codex_apply_patch::maybe_parse_apply_patch_verified     │
│  - 语法解析、路径验证、安全性检查                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    安全评估层                                │
│  - assess_patch_safety: 评估补丁安全性                      │
│  - 决定是否需要用户批准                                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    执行层                                    │
│  - ApplyPatchRuntime: 运行时实现                            │
│  - ToolOrchestrator: 协调执行流程                           │
│  - 独立进程执行 (--codex-run-as-apply-patch)               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    事件层                                    │
│  - ToolEmitter: 发出应用开始/结束事件                       │
│  - TurnDiffTracker: 跟踪文件变更                            │
└─────────────────────────────────────────────────────────────┘
```

## 1. 工具定义 (Tool Definition)

### 1.1 模型可用性

`apply_patch` 工具并非对所有模型都可用，其可用性取决于**模型家族配置**和**特性标志**的组合。

#### 1.1.1 模型家族配置

**文件引用**: `d:\codex\codex-rs\core\src\models_manager\model_family.rs`

不同模型家族通过 `apply_patch_tool_type` 字段决定支持情况：

| 模型系列 | apply_patch_tool_type | 说明 |
|---------|----------------------|------|
| `gpt-5-codex`, `gpt-5.1-codex`, `codex-*` | `Some(Freeform)` | Codex 系列，支持 Freeform 格式 |
| `gpt-5.1`, `gpt-5.2`, `gpt-5.2-codex` | `Some(Freeform)` | GPT-5 系列，支持 Freeform 格式 |
| `exp-*`, `bengalfox`, `boomslang` | `Some(Freeform)` | 实验性模型，支持 Freeform 格式 |
| `gpt-5.1-codex-max` | `Some(Freeform)` | Codex Max 版本，支持 Freeform 格式 |
| `gpt-oss`, `openai/gpt-oss` | `Some(Function)` | GPT-OSS 系列，支持 Function 格式 |
| `deepseek-chat`, `deepseek/deepseek-chat` | `Some(Function)` | DeepSeek 系列，支持 Function 格式 |
| `mimo-v2-flash`, `mi-mo-v2-flash`, `MiMo-V2-Flash` | `Some(Freeform)` | MiMo-V2-Flash，支持 Freeform 格式 |
| `gpt-5` | `None` | 默认不支持，需要特性标志 |
| `o3`, `o4-mini` | `None` | 推理模型，不支持 |
| `gpt-4.1`, `gpt-4o`, `gpt-3.5` | `None` | 旧版 GPT 模型，不支持 |
| `codex-mini-latest` | `None` | Mini 版本，默认不支持 |

#### 1.1.2 特性标志控制

**文件引用**: `d:\codex\codex-rs\core\src\features.rs`

即使模型家族支持，还需要通过 `Feature::ApplyPatchFreeform` 特性标志控制：

```rust
// 特性定义
FeatureSpec {
    id: Feature::ApplyPatchFreeform,
    key: "apply_patch_freeform",
    stage: Stage::Experimental,
    default_enabled: false,  // 默认关闭
}
```

#### 1.1.3 工具注册逻辑

**文件引用**: `d:\codex\codex-rs\core\src\tools\spec.rs`

```rust
// 在 ToolsConfig::new() 中决定是否启用
let include_apply_patch_tool = features.enabled(Feature::ApplyPatchFreeform);

let apply_patch_tool_type = match model_family.apply_patch_tool_type {
    Some(ApplyPatchToolType::Freeform) => Some(ApplyPatchToolType::Freeform),
    Some(ApplyPatchToolType::Function) => Some(ApplyPatchToolType::Function),
    None => {
        if include_apply_patch_tool {
            Some(ApplyPatchToolType::Freeform)  // 通过特性标志启用
        } else {
            None
        }
    }
};

// 在 build_specs 中注册工具
if let Some(apply_patch_tool_type) = &config.apply_patch_tool_type {
    match apply_patch_tool_type {
        ApplyPatchToolType::Freeform => {
            builder.push_spec(create_apply_patch_freeform_tool());
        }
        ApplyPatchToolType::Function => {
            builder.push_spec(create_apply_patch_json_tool());
        }
    }
    builder.register_handler("apply_patch", apply_patch_handler);
}
```

#### 1.1.4 历史配置兼容性

**文件引用**: `d:\codex\codex-rs\core\src\features\legacy.rs`

支持旧版配置选项，自动映射到新特性标志：

```rust
const ALIASES: &[Alias] = &[
    Alias {
        legacy_key: "experimental_use_freeform_apply_patch",
        feature: Feature::ApplyPatchFreeform,
    },
    Alias {
        legacy_key: "include_apply_patch_tool",
        feature: Feature::ApplyPatchFreeform,
    },
];
```

#### 1.1.5 测试验证示例

**文件引用**: `d:\codex\codex-rs\core\src\tools\spec.rs`

```rust
// gpt-5-codex 默认包含 apply_patch
assert_model_tools(
    "gpt-5-codex",
    &Features::with_defaults(),
    &["shell_command", "apply_patch", "view_image", ...],
);

// codex-mini-latest 默认不包含 apply_patch
assert_model_tools(
    "codex-mini-latest",
    &Features::with_defaults(),
    &["local_shell", "view_image", ...],  // 无 apply_patch
);

// gpt-5 默认不包含 apply_patch
assert_model_tools(
    "gpt-5",
    &Features::with_defaults(),
    &["shell", "view_image", ...],  // 无 apply_patch
);
```

#### 1.1.6 配置示例

要为不支持的模型启用 apply_patch，需要在配置中启用特性：

```toml
# 在 config.toml 中
[features]
apply_patch_freeform = true

# 或使用旧版配置
experimental_use_freeform_apply_patch = true
include_apply_patch_tool = true
```

### 1.2 两种工具格式

**文件引用**: `d:\codex\codex-rs\core\src\tools\handlers\apply_patch.rs`

#### Freeform 格式 (适用于 GPT-5 等模型)

```rust
pub(crate) fn create_apply_patch_freeform_tool() -> ToolSpec {
    ToolSpec::Freeform(FreeformTool {
        name: "apply_patch".to_string(),
        description: "Use the `apply_patch` tool to edit files. This is a FREEFORM tool, so do not wrap the patch in JSON.".to_string(),
        format: FreeformToolFormat {
            r#type: "grammar".to_string(),
            syntax: "lark".to_string(),
            definition: APPLY_PATCH_LARK_GRAMMAR.to_string(),
        },
    })
}
```

#### Function 格式 (适用于 GPT-OSS 模型)

```rust
pub(crate) fn create_apply_patch_json_tool() -> ToolSpec {
    let mut properties = BTreeMap::new();
    properties.insert(
        "input".to_string(),
        JsonSchema::String {
            description: Some(r#"The entire contents of the apply_patch command"#.to_string()),
        },
    );

    ToolSpec::Function(ResponsesApiTool {
        name: "apply_patch".to_string(),
        description: r#"Use the `apply_patch` tool to edit files."#.to_string(),
        strict: false,
        parameters: JsonSchema::Object {
            properties,
            required: Some(vec!["input".to_string()]),
            additional_properties: Some(false.into()),
        },
    })
}
```

### 1.2 Lark 语法定义

**文件引用**: `d:\codex\codex-rs\core\src\tools\handlers\apply_patch.rs` (APPLY_PATCH_LARK_GRAMMAR)

```lark
Patch := Begin { FileOp } End
Begin := "*** Begin Patch" NEWLINE
End := "*** End Patch" NEWLINE
FileOp := AddFile | DeleteFile | UpdateFile
AddFile := "*** Add File: " path NEWLINE { "+" line NEWLINE }
DeleteFile := "*** Delete File: " path NEWLINE
UpdateFile := "*** Update File: " path NEWLINE [ MoveTo ] { Hunk }
MoveTo := "*** Move to: " newPath NEWLINE
Hunk := "@@" [ header ] NEWLINE { HunkLine } [ "*** End of File" NEWLINE ]
HunkLine := (" " | "-" | "+") text NEWLINE
```

## 2. 工具注册 (Tool Registration)

### 2.1 配置驱动的注册

**文件引用**: `d:\codex\codex-rs\core\src\tools\spec.rs`

```rust
// 在 build_specs 函数中
if let Some(apply_patch_tool_type) = &config.apply_patch_tool_type {
    match apply_patch_tool_type {
        ApplyPatchToolType::Freeform => {
            builder.push_spec(create_apply_patch_freeform_tool());
        }
        ApplyPatchToolType::Function => {
            builder.push_spec(create_apply_patch_json_tool());
        }
    }
    builder.register_handler("apply_patch", apply_patch_handler);
}
```

### 2.2 处理器注册

**文件引用**: `d:\codex\codex-rs\core\src\tools\spec.rs`

```rust
let apply_patch_handler = Arc::new(ApplyPatchHandler);
```

## 3. 处理器实现 (Handler Implementation)

### 3.1 ApplyPatchHandler trait 实现

**文件引用**: `d:\codex\codex-rs\core\src\tools\handlers\apply_patch.rs`

```rust
#[async_trait]
impl ToolHandler for ApplyPatchHandler {
    fn kind(&self) -> ToolKind {
        ToolKind::Function
    }

    fn matches_kind(&self, payload: &ToolPayload) -> bool {
        matches!(
            payload,
            ToolPayload::Function { .. } | ToolPayload::Custom { .. }
        )
    }

    async fn is_mutating(&self, _invocation: &ToolInvocation) -> bool {
        true  // apply_patch 总是修改性操作
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        // 核心处理逻辑
    }
}
```

### 3.2 核心处理流程

```rust
async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
    let ToolInvocation {
        session,
        turn,
        tracker,
        call_id,
        tool_name,
        payload,
    } = invocation;

    // 1. 提取补丁输入
    let patch_input = match payload {
        ToolPayload::Function { arguments } => {
            let args: ApplyPatchToolArgs = serde_json::from_str(&arguments)?;
            args.input
        }
        ToolPayload::Custom { input } => input,
        _ => return Err(FunctionCallError::RespondToModel(...)),
    };

    // 2. 解析和验证补丁
    let cwd = turn.cwd.clone();
    let command = vec!["apply_patch".to_string(), patch_input.clone()];
    match codex_apply_patch::maybe_parse_apply_patch_verified(&command, &cwd) {
        codex_apply_patch::MaybeApplyPatchVerified::Body(changes) => {
            // 3. 应用补丁
            match apply_patch::apply_patch(session.as_ref(), turn.as_ref(), &call_id, changes).await {
                InternalApplyPatchInvocation::Output(item) => {
                    // 直接输出结果
                    let content = item?;
                    Ok(ToolOutput::Function { content, ... })
                }
                InternalApplyPatchInvocation::DelegateToExec(apply) => {
                    // 4. 委托给执行层
                    let emitter = ToolEmitter::apply_patch(...);
                    emitter.begin(event_ctx).await;
                    
                    let req = ApplyPatchRequest { ... };
                    let mut orchestrator = ToolOrchestrator::new();
                    let mut runtime = ApplyPatchRuntime::new();
                    let out = orchestrator.run(&mut runtime, &req, &tool_ctx, &turn, ...).await;
                    
                    let content = emitter.finish(event_ctx, out).await?;
                    Ok(ToolOutput::Function { content, ... })
                }
            }
        }
        // 错误处理...
    }
}
```

## 4. 解析与验证 (Parsing & Verification)

### 4.1 解析流程

**文件引用**: `d:\codex\codex-rs\apply-patch\src\invocation.rs`

```rust
pub fn maybe_parse_apply_patch_verified(argv: &[String], cwd: &Path) -> MaybeApplyPatchVerified {
    // 1. 检查是否为原始补丁体（拒绝）
    if let [body] = argv && parse_patch(body).is_ok() {
        return MaybeApplyPatchVerified::CorrectnessError(ApplyPatchError::ImplicitInvocation);
    }

    // 2. 解析补丁
    match maybe_parse_apply_patch(argv) {
        MaybeApplyPatch::Body(ApplyPatchArgs { patch, hunks, workdir }) => {
            // 3. 计算有效工作目录
            let effective_cwd = workdir
                .as_ref()
                .map(|dir| {
                    let path = Path::new(dir);
                    if path.is_absolute() {
                        path.to_path_buf()
                    } else {
                        cwd.join(path)
                    }
                })
                .unwrap_or_else(|| cwd.to_path_buf());

            // 4. 转换为文件变更
            let mut changes = HashMap::new();
            for hunk in hunks {
                let path = hunk.resolve_path(&effective_cwd);
                match hunk {
                    Hunk::AddFile { contents, .. } => {
                        changes.insert(path, ApplyPatchFileChange::Add { content: contents });
                    }
                    Hunk::DeleteFile { .. } => {
                        let content = std::fs::read_to_string(&path)?;
                        changes.insert(path, ApplyPatchFileChange::Delete { content });
                    }
                    Hunk::UpdateFile { move_path, chunks, .. } => {
                        let diff = unified_diff_from_chunks(&path, &chunks)?;
                        changes.insert(path, ApplyPatchFileChange::Update {
                            unified_diff: diff,
                            move_path: move_path.map(|p| effective_cwd.join(p)),
                            new_content: contents,
                        });
                    }
                }
            }
            
            MaybeApplyPatchVerified::Body(ApplyPatchAction {
                changes,
                patch,
                cwd: effective_cwd,
            })
        }
        // 其他情况...
    }
}
```

### 4.2 补丁格式解析

**文件引用**: `d:\codex\codex-rs\apply-patch\src\parser.rs`

支持三种操作类型：
- **Add File**: 创建新文件
- **Delete File**: 删除现有文件  
- **Update File**: 更新现有文件（可选移动）

每个操作包含：
- 文件路径（相对路径解析为绝对路径）
- 变更内容（对于 Add 和 Update）
- 统一差异格式（对于 Update）

## 5. 安全评估 (Safety Assessment)

### 5.1 安全检查流程

**文件引用**: `d:\codex\codex-rs\core\src\apply_patch.rs`

```rust
pub(crate) async fn apply_patch(
    sess: &Session,
    turn_context: &TurnContext,
    call_id: &str,
    action: ApplyPatchAction,
) -> InternalApplyPatchInvocation {
    match assess_patch_safety(
        &action,
        turn_context.approval_policy,
        &turn_context.sandbox_policy,
        &turn_context.cwd,
    ) {
        SafetyCheck::AutoApprove { user_explicitly_approved, .. } => {
            InternalApplyPatchInvocation::DelegateToExec(ApplyPatchExec {
                action,
                user_explicitly_approved_this_action: user_explicitly_approved,
            })
        }
        SafetyCheck::AskUser => {
            // 请求用户批准
            let rx_approve = sess.request_patch_approval(...).await;
            match rx_approve.await.unwrap_or_default() {
                ReviewDecision::Approved | ... => {
                    InternalApplyPatchInvocation::DelegateToExec(ApplyPatchExec {
                        action,
                        user_explicitly_approved_this_action: true,
                    })
                }
                ReviewDecision::Denied | ReviewDecision::Abort => {
                    InternalApplyPatchInvocation::Output(Err(
                        FunctionCallError::RespondToModel("patch rejected by user".to_string())
                    ))
                }
            }
        }
        SafetyCheck::Reject { reason } => {
            InternalApplyPatchInvocation::Output(Err(
                FunctionCallError::RespondToModel(format!("patch rejected: {reason}"))
            ))
        }
    }
}
```

### 5.2 安全评估逻辑

**文件引用**: `d:\codex\codex-rs\core\src\safety.rs`

```rust
pub fn assess_patch_safety(
    action: &ApplyPatchAction,
    policy: AskForApproval,
    sandbox_policy: &SandboxPolicy,
    cwd: &Path,
) -> SafetyCheck {
    if action.is_empty() {
        return SafetyCheck::Reject { reason: "empty patch".to_string() };
    }

    // 检查补丁是否限制在可写路径内
    if is_write_patch_constrained_to_writable_paths(action, sandbox_policy, cwd)
        || policy == AskForApproval::OnFailure
    {
        if matches!(sandbox_policy, SandboxPolicy::DangerFullAccess) {
            SafetyCheck::AutoApprove {
                sandbox_type: SandboxType::None,
                user_explicitly_approved: false,
            }
        } else {
            match get_platform_sandbox() {
                Some(sandbox_type) => SafetyCheck::AutoApprove {
                    sandbox_type,
                    user_explicitly_approved: false,
                },
                None => SafetyCheck::AskUser,
            }
        }
    } else if policy == AskForApproval::Never {
        SafetyCheck::Reject {
            reason: "writing outside of the project; rejected by user approval settings".to_string(),
        }
    } else {
        SafetyCheck::AskUser
    }
}
```

## 6. 执行层 (Execution Layer)

### 6.1 ApplyPatchRuntime

**文件引用**: `d:\codex\codex-rs\core\src\tools\runtimes\apply_patch.rs`

```rust
impl ToolRuntime<ApplyPatchRequest, ExecToolCallOutput> for ApplyPatchRuntime {
    async fn run(
        &mut self,
        req: &ApplyPatchRequest,
        attempt: &SandboxAttempt<'_>,
        ctx: &ToolCtx<'_>,
    ) -> Result<ExecToolCallOutput, ToolError> {
        let spec = Self::build_command_spec(req)?;
        let env = attempt.env_for(spec).map_err(|err| ToolError::Codex(err.into()))?;
        let out = execute_env(env, attempt.policy, Self::stdout_stream(ctx)).await?;
        Ok(out)
    }
}
```

### 6.2 命令构建

```rust
fn build_command_spec(req: &ApplyPatchRequest) -> Result<CommandSpec, ToolError> {
    let exe = if let Some(path) = &req.codex_exe {
        path.clone()
    } else {
        env::current_exe().map_err(|e| ToolError::Rejected(...))?
    };
    
    Ok(CommandSpec {
        program: exe.to_string_lossy().to_string(),
        args: vec![CODEX_APPLY_PATCH_ARG1.to_string(), req.patch.clone()],
        cwd: req.cwd.clone(),
        expiration: req.timeout_ms.into(),
        env: HashMap::new(),
        sandbox_permissions: SandboxPermissions::UseDefault,
        justification: None,
    })
}
```

### 6.3 ToolOrchestrator 协调

**文件引用**: `d:\codex\codex-rs\core\src\tools\orchestrator.rs`

执行流程：
1. **审批检查**: 通过 `exec_approval_requirement` 决定是否需要批准
2. **沙箱选择**: 根据策略选择合适的沙箱类型
3. **执行尝试**: 在沙箱中执行命令
4. **重试逻辑**: 如果失败，尝试无沙箱执行（如果策略允许）

## 7. 独立可执行文件 (Standalone Executable)

### 7.1 主函数

**文件引用**: `d:\codex\codex-rs\apply-patch\src\standalone_executable.rs`

```rust
pub fn main() -> ! {
    let exit_code = run_main();
    std::process::exit(exit_code);
}

pub fn run_main() -> i32 {
    let mut args = std::env::args_os();
    let _argv0 = args.next();

    // 读取补丁参数
    let patch_arg = match args.next() {
        Some(arg) => arg.into_string().unwrap_or_else(|_| {
            eprintln!("Error: apply_patch requires a UTF-8 PATCH argument.");
            1
        }),
        None => {
            // 从 stdin 读取
            let mut buf = String::new();
            std::io::stdin().read_to_string(&mut buf).unwrap();
            buf
        }
    };

    // 应用补丁
    match crate::apply_patch(&patch_arg, &mut stdout, &mut stderr) {
        Ok(()) => 0,
        Err(_) => 1,
    }
}
```

### 7.2 应用补丁函数

**文件引用**: `d:\codex\codex-rs\apply-patch\src\lib.rs`

```rust
pub fn apply_patch(
    patch: &str,
    stdout: &mut impl std::io::Write,
    stderr: &mut impl std::io::Write,
) -> Result<(), ApplyPatchError> {
    // 1. 解析补丁
    let hunks = match parse_patch(patch) {
        Ok(source) => source.hunks,
        Err(e) => {
            writeln!(stderr, "Invalid patch: {message}")?;
            return Err(ApplyPatchError::ParseError(e));
        }
    };

    // 2. 应用变更
    apply_hunks(&hunks, stdout, stderr)?;

    Ok(())
}

pub fn apply_hunks(
    hunks: &[Hunk],
    stdout: &mut impl std::io::Write,
    stderr: &mut impl std::io::Write,
) -> Result<(), ApplyPatchError> {
    // 3. 执行文件操作
    match apply_hunks_to_files(hunks) {
        Ok(affected) => {
            print_summary(&affected, stdout)?;
            Ok(())
        }
        Err(err) => {
            writeln!(stderr, "{err}")?;
            Err(ApplyPatchError::IoError(...))
        }
    }
}
```

## 8. 事件与跟踪 (Events & Tracking)

### 8.1 工具事件发射器

**文件引用**: `d:\codex\codex-rs\core\src\tools\events.rs`

```rust
pub struct ToolEmitter {
    // 不同类型的事件发射器
}

impl ToolEmitter {
    pub fn apply_patch(changes: HashMap<PathBuf, FileChange>, auto_approved: bool) -> Self {
        Self::ApplyPatch { changes, auto_approved }
    }

    pub async fn begin(&self, ctx: ToolEventCtx<'_>) {
        // 发出开始事件
        let event = EventMsg::PatchApplyBegin(PatchApplyBeginEvent {
            call_id: ctx.call_id.to_string(),
            changes: self.changes.clone(),
            auto_approved: self.auto_approved,
        });
        ctx.session.emit_event(event).await;
    }

    pub async fn finish(
        &self,
        ctx: ToolEventCtx<'_>,
        result: Result<ExecToolCallOutput, ToolError>,
    ) -> Result<String, FunctionCallError> {
        // 发出结束事件并返回结果
        match result {
            Ok(output) => {
                let event = EventMsg::PatchApplyEnd(PatchApplyEndEvent {
                    call_id: ctx.call_id.to_string(),
                    success: true,
                    output: output.stdout.clone(),
                });
                ctx.session.emit_event(event).await;
                Ok(output.stdout)
            }
            Err(err) => {
                // 发出失败事件
                Err(FunctionCallError::RespondToModel(err.to_string()))
            }
        }
    }
}
```

### 8.2 文件变更跟踪

**文件引用**: `d:\codex\codex-rs\core\src\tools\context.rs`

```rust
pub struct TurnDiffTracker {
    pub changes: Arc<Mutex<HashMap<PathBuf, FileChange>>>,
}

impl TurnDiffTracker {
    pub fn record_apply_patch(&self, changes: HashMap<PathBuf, FileChange>) {
        let mut guard = self.changes.lock().unwrap();
        for (path, change) in changes {
            guard.insert(path, change);
        }
    }
}
```

## 9. 完整调用流程示例

### 9.1 从模型请求到文件修改

```
1. 模型响应
   ↓
2. 解析工具调用 (ToolRouter::build_tool_call)
   - 检测到 apply_patch 工具
   - 创建 ToolCall 结构
   ↓
3. 路由分发 (ToolRouter::dispatch_tool_call)
   - 查找 ApplyPatchHandler
   - 创建 ToolInvocation
   ↓
4. 处理器执行 (ApplyPatchHandler::handle)
   - 提取补丁输入
   - 调用 maybe_parse_apply_patch_verified
   ↓
5. 解析验证 (codex_apply_patch)
   - 语法解析
   - 路径验证
   - 生成 ApplyPatchAction
   ↓
6. 安全评估 (apply_patch::apply_patch)
   - assess_patch_safety
   - 决定：AutoApprove / AskUser / Reject
   ↓
7. 执行准备
   - 如果需要批准：请求用户
   - 构建 ApplyPatchRequest
   ↓
8. 工具编排 (ToolOrchestrator::run)
   - 审批检查
   - 沙箱选择
   - 执行命令
   ↓
9. 独立进程执行
   - codex --codex-run-as-apply-patch "patch"
   - apply_patch 二进制解析并应用变更
   ↓
10. 事件发射
    - PatchApplyBeginEvent
    - PatchApplyEndEvent
    ↓
11. 返回结果
    - ToolOutput::Function
    - 包含执行结果和变更摘要
```

### 9.2 代码示例

```rust
// 模型调用
let tool_call = ResponseItem::FunctionCall {
    name: "apply_patch".to_string(),
    arguments: r#"{"input":"*** Begin Patch\n*** Add File: hello.txt\n+Hello, world!\n*** End Patch\n"}"#.to_string(),
    call_id: "call_123".to_string(),
    ..Default::default()
};

// 路由分发
let tool_call = ToolRouter::build_tool_call(session, tool_call).await?;
let response = router.dispatch_tool_call(session, turn, tracker, tool_call).await?;

// 处理器处理
let handler = registry.handler("apply_patch").unwrap();
let output = handler.handle(invocation).await?;

// 执行结果
assert_eq!(output.success, Some(true));
assert!(output.content.contains("hello.txt"));
```

## 10. 关键文件总结

| 文件 | 作用 |
|------|------|
| `core/src/tools/handlers/apply_patch.rs` | 工具处理器实现 |
| `core/src/tools/runtimes/apply_patch.rs` | 执行运行时 |
| `core/src/apply_patch.rs` | 核心应用逻辑和安全评估 |
| `core/src/safety.rs` | 安全检查和沙箱策略 |
| `apply-patch/src/lib.rs` | 独立库（解析、应用、执行） |
| `apply-patch/src/invocation.rs` | 命令解析和验证 |
| `apply-patch/src/parser.rs` | 补丁语法解析 |
| `apply-patch/src/standalone_executable.rs` | 独立可执行文件入口 |
| `core/src/tools/spec.rs` | 工具定义和注册 |
| `core/src/tools/registry.rs` | 工具注册表 |
| `core/src/tools/router.rs` | 工具路由分发 |
| `core/src/tools/orchestrator.rs` | 执行协调器 |
| `core/src/tools/events.rs` | 事件发射器 |

## 11. 安全特性

### 11.1 多层防护

1. **语法验证**: 通过 Lark 语法确保补丁格式正确
2. **路径验证**: 确保所有路径在可写范围内
3. **沙箱执行**: 在隔离环境中执行文件操作
4. **用户批准**: 敏感操作需要用户明确批准
5. **审计日志**: 所有变更都有详细事件记录

### 11.2 沙箱策略

- **macOS**: Seatbelt (`/usr/bin/sandbox-exec`)
- **Linux**: Landlock + Seccomp
- **Windows**: Restricted Token

### 11.3 审批策略

```rust
pub enum AskForApproval {
    Never,        // 从不请求（但仍有安全检查）
    OnRequest,    // 按需请求
    OnFailure,    // 仅在失败时
    UnlessTrusted, // 除非受信任
}
```

## 12. 性能优化

### 12.1 输出截断

```rust
pub struct TruncationPolicy {
    pub max_output_tokens: Option<usize>,
    pub max_lines: Option<usize>,
}
```

### 12.2 门控机制

通过 `ReadinessFlag` 避免并发工具调用的竞争条件：

```rust
if handler.is_mutating(&invocation).await {
    invocation.turn.tool_call_gate.wait_ready().await;
}
```

### 12.3 缓存批准

```rust
with_cached_approval(&session.services, key, move || async move {
    // 仅在需要时请求批准
})
```

## 13. 测试策略

### 13.1 单元测试

```rust
#[test]
fn test_apply_patch_add_file() {
    let patch = "*** Begin Patch\n*** Add File: test.txt\n+content\n*** End Patch";
    let result = apply_patch(patch, &mut stdout, &mut stderr);
    assert!(result.is_ok());
}
```

### 13.2 集成测试

**文件引用**: `d:\codex\codex-rs\core\tests\suite\apply_patch_cli.rs`

测试场景：
- 自由格式工具调用
- JSON 格式工具调用
- 用户批准流程
- 沙箱执行
- 错误处理

### 13.3 快照测试

验证工具规范和输出格式的稳定性。

## 14. 模型可用性总结

### 14.1 核心要点

`apply_patch` 工具的可用性遵循以下原则：

1. **模型特定**: 只有特定模型系列支持该工具
2. **配置驱动**: 通过模型家族配置和特性标志共同决定
3. **向后兼容**: 支持旧版配置选项
4. **实验性特性**: 默认关闭，需要显式启用

### 14.2 支持矩阵

**默认支持的模型**（无需额外配置）：
- ✅ `gpt-5-codex`, `gpt-5.1-codex`, `codex-*`
- ✅ `gpt-5.1`, `gpt-5.2`, `gpt-5.2-codex`
- ✅ `exp-*`, `bengalfox`, `boomslang`
- ✅ `gpt-5.1-codex-max`
- ✅ `gpt-oss`, `openai/gpt-oss`
- ✅ `deepseek-chat`, `deepseek/deepseek-chat`
- ✅ `mimo-v2-flash`, `mi-mo-v2-flash`, `MiMo-V2-Flash`

**默认不支持的模型**（需要配置）：
- ❌ `gpt-5`
- ❌ `o3`, `o4-mini`
- ❌ `gpt-4.1`, `gpt-4o`, `gpt-3.5`
- ❌ `codex-mini-latest`

### 14.3 启用方法

对于不支持的模型，可以通过以下方式启用：

1. **特性标志配置**：
   ```toml
   [features]
   apply_patch_freeform = true
   ```

2. **旧版配置兼容**：
   ```toml
   experimental_use_freeform_apply_patch = true
   include_apply_patch_tool = true
   ```

### 14.4 设计启示

该工具的可用性设计体现了 Codex 的**渐进式特性发布**策略：

- **实验阶段**: 默认关闭，仅对特定模型开放
- **配置驱动**: 通过模型元数据和用户配置灵活控制
- **安全优先**: 即使支持，也有严格的安全评估流程
- **向后兼容**: 旧配置选项仍然有效

这种设计确保了工具的稳定性和可控性，同时为未来扩展预留了空间。

## 15. 总结

`apply_patch` 工具的实现体现了 Codex 项目的核心设计原则：

1. **分层架构**: 清晰的职责分离，从定义到执行各层独立
2. **安全优先**: 多层防护机制确保文件操作安全
3. **配置驱动**: 通过配置决定工具格式和可用性
4. **异步处理**: 基于 Rust async/await 的高效执行
5. **事件驱动**: 完整的事件流支持调试和审计
6. **可扩展性**: 易于添加新的安全策略和执行模式

该工具是 Codex 项目文件操作能力的核心，为所有代码编辑任务提供了安全、可靠且高效的解决方案。