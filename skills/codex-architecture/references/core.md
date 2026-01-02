# Core 模块 - 核心业务逻辑

## 概述
`codex-core` 是 Codex 项目的核心业务逻辑实现，为所有 UI 组件提供基础功能，并协调各层之间的交互。该模块位于 UI 层和协议/通信层之间，不依赖 UI，作为复杂操作的外观（Facade）。

## 核心功能
- **编排**: 协调 UI、协议和后端层之间的交互
- **业务逻辑**: 实现 Codex 的核心功能
- **沙箱管理**: 提供跨平台的沙箱执行环境
- **状态管理**: 处理应用状态和配置
- **错误处理**: 集中化的错误处理和传播

## 架构设计
### 层级位置
- 位于 UI 层和协议/通信层之间
- 无直接 UI 依赖（纯业务逻辑）
- 作为复杂操作的外观（Facade）

### 关键模块
- `codex.rs`: 核心会话管理和任务编排
- `client.rs`: 模型客户端，处理与后端 API 的通信
- `config/mod.rs`: 配置管理，加载和合并配置层
- `tools/`: 工具实现和路由
  - `handlers/`: 具体工具处理器（apply_patch, shell, 等）
  - `runtimes/`: 工具运行时（处理沙箱和审批）
  - `sandboxing.rs`: 共享的沙箱和审批逻辑
  - `spec.rs`: 工具定义和 JSON Schema
- `exec.rs`: 命令执行和沙箱环境
- `sandboxing/`: 平台特定的沙箱实现（seatbelt, landlock, windows）
- `mcp/`: MCP（Model Context Protocol）客户端和连接管理
- `unified_exec/`: 交互式 PTY 执行管理
- `skills/`: 技能加载和管理
- `conversation_manager.rs`: 对话历史管理
- `rollout/`: 会话记录和回放

## 平台依赖
- **macOS**: 需要 `/usr/bin/sandbox-exec` 和 Seatbelt 配置文件
- **Linux**: 需要基于命名空间的沙箱（通过 `codex-linux-sandbox`）
- **Windows**: 需要 Windows Sandbox 集成
- **所有平台**: 需要模拟的 `apply_patch` CLI（通过 `--codex-run-as-apply-patch` 参数）

## 依赖关系
### 内部依赖
- `codex-protocol`: 内部通信的类型定义
- `codex-backend-client`: API 通信
- `codex-common`: 共享工具
- `codex-login`: 认证处理

### 外部依赖
- `tokio`: 异步运行时
- `anyhow`: 错误处理
- `serde`: 序列化
- `tracing`: 日志记录

## 相关组件
- `codex-core` (crate)
- `core/` (目录)

## 架构关系
作为业务逻辑层，它被以下组件依赖：
- `codex-tui` (终端 UI) - 为交互式界面提供业务逻辑
- `codex-exec` (非交互式执行) - 为自动化任务提供逻辑
- `codex-cli` (主 CLI 工具) - 为命令提供核心功能
- `codex-mcp-server` (MCP 服务器) - 为 MCP 协议提供业务逻辑

## 常见使用模式
### 代理执行流程
1. UI 接收用户输入
2. Core 处理输入并管理代理状态
3. Core 调用后端客户端进行 API 通信
4. Core 处理响应并更新状态
5. UI 从核心状态渲染结果

### 工具集成
- Core 实现工具接口
- 工具由 Core 注册和管理
- UI 层通过 Core 外观调用工具

## 详细模块说明

### `codex.rs` - 核心会话管理
- `Codex`: 高级接口，提供 `submit()` 和 `next_event()` 方法
- `Session`: 会话状态管理，包含：
  - `SessionConfiguration`: 会话配置（模型、提供者、策略等）
  - `TurnContext`: 单次任务的上下文（工作目录、工具配置等）
  - `SessionServices`: 共享服务（MCP 连接、审批存储、遥测等）
- `run_task()`: 主要任务执行循环，处理模型响应和工具调用
- `run_turn()`: 单次模型调用，处理流式响应和并行工具调用

### `client.rs` - 模型客户端
- `ModelClient`: 封装与后端 API 的通信
  - `stream()`: 根据提供者使用 Chat 或 Responses API 流式处理响应
  - `compact_conversation_history()`: 使用压缩端点压缩对话历史
- 支持 `WireApi::Chat` 和 `WireApi::Responses` 两种协议
- 处理认证刷新和错误映射

### `config/mod.rs` - 配置管理
- `Config`: 应用配置结构，包含：
  - 模型和提供者配置
  - 审批和沙箱策略
  - 功能标志（`Features`）
  - MCP 服务器配置
  - TUI 设置
- `ConfigBuilder`: 用于构建配置的构建器模式
- 支持多层配置合并（全局、项目、CLI 覆盖）

### `tools/` - 工具系统
- **工具定义** (`spec.rs`):
  - `exec_command`: 执行命令（统一 exec 或 shell）
  - `shell`: 执行 shell 命令（数组参数）
  - `shell_command`: 执行 shell 命令（字符串参数）
  - `apply_patch`: 文件编辑工具（自由格式或 JSON）
  - `view_image`: 查看图像
  - `grep_files`: 文件搜索
  - `read_file`: 文件读取
  - `plan`: 规划工具
- **工具路由** (`router.rs`):
  - 将工具调用分派到相应的处理器
  - 处理 MCP 工具名称解析（`mcp__server__tool`）
- **工具注册** (`registry.rs`):
  - 注册工具处理器
  - 处理工具调用的生命周期（日志、遥测、错误处理）
- **工具处理器** (`handlers/`):
  - `ShellHandler`/`ShellCommandHandler`: 执行 shell 命令
  - `ApplyPatchHandler`: 处理文件编辑
  - `McpHandler`: 调用 MCP 工具
  - `UnifiedExecHandler`: 管理交互式 PTY 会话
- **工具运行时** (`runtimes/`):
  - `ShellRuntime`: 执行 shell 命令的运行时（处理沙箱和审批）
  - `ApplyPatchRuntime`: 执行 apply_patch 的运行时
  - `UnifiedExecRuntime`: 管理交互式 exec 会话的运行时
- **沙箱和审批** (`sandboxing.rs`):
  - `ToolOrchestrator`: 协调审批、沙箱选择和重试
  - `SandboxAttempt`: 代表一次沙箱执行尝试
  - `ExecApprovalRequirement`: 定义审批需求

### `exec.rs` - 命令执行
- `process_exec_tool_call()`: 处理 exec 工具调用的入口点
- `ExecParams`: 执行参数（命令、工作目录、超时、环境、沙箱权限）
- `ExecToolCallOutput`: 执行输出（退出码、stdout、stderr、聚合输出、持续时间）
- 支持平台特定沙箱（Seatbelt、Linux Seccomp、Windows Restricted Token）

### `sandboxing/` - 平台沙箱
- `SandboxManager`: 转换 `CommandSpec` 为 `ExecEnv`
- `seatbelt.rs`: macOS Seatbelt 配置文件生成
- `landlock.rs`: Linux landlock/seccomp 配置
- `windows-sandbox-rs`: Windows 沙箱集成

### `mcp/` - MCP 客户端
- `McpConnectionManager`: 管理多个 MCP 服务器连接
- `collect_mcp_snapshot()`: 收集所有 MCP 工具和资源
- 支持 OAuth、Bearer Token 和 Stdio/HTTP 传输
- `SandboxState` 通知：向 MCP 服务器推送沙箱策略更新

### `unified_exec/` - 交互式执行
- `UnifiedExecSessionManager`: 管理持久化 PTY 会话
- `UnifiedExecSession`: 封装 PTY 进程和输出缓冲
- `exec_command()`: 打开或重用会话
- `write_stdin()`: 向会话写入输入并获取输出
- 支持会话复用、输出截断和自动清理

### `skills/` - 技能系统
- `SkillsManager`: 加载和缓存技能
- `load_skills()`: 从 `codex_home/skills/` 和项目目录加载技能
- `SkillMetadata`: 技能元数据（名称、描述、路径、范围）
- 技能作为初始上下文注入到对话中

### `conversation_manager.rs` - 对话管理
- `ContextManager`: 管理对话历史（`ResponseItem` 列表）
- `record_items()`: 将项目添加到历史记录
- `estimate_token_count()`: 估算令牌使用量
- `replace_history()`: 替换整个历史记录

### `rollout/` - 会话记录
- `RolloutRecorder`: 将事件和响应项记录到 `~/.codex/sessions/`
- `SessionMeta`: 会话元数据
- `list_conversations()`: 列出和分页历史会话
- 支持会话恢复和分叉

## 关键执行流程示例

### 用户输入到模型响应
1. `Codex::submit(Op::UserTurn { ... })`
2. `submission_loop()` 处理提交
3. `Session::new_turn_with_sub_id()` 创建 `TurnContext`
4. `run_task()` 启动任务
5. `run_turn()` 调用 `ModelClient::stream()`
6. 流式处理响应，处理 `ResponseEvent`
7. 如果是工具调用，通过 `ToolRouter` 分派
8. `ToolOrchestrator` 处理审批和沙箱
9. 工具运行时执行命令
10. 输出返回给模型进行下一轮

### 沙箱执行流程
1. `ToolOrchestrator::run()`
2. 检查 `ExecApprovalRequirement`
3. 如果需要，请求用户审批
4. `SandboxManager::select_initial()` 选择沙箱类型
5. `SandboxAttempt::env_for()` 转换 `CommandSpec` 为 `ExecEnv`
6. 执行命令（`execute_env()`）
7. 如果沙箱被拒绝且策略允许，重试无沙箱
8. 返回结果或错误

### MCP 工具调用
1. `ModelClient` 流式处理响应，检测到 `mcp__server__tool` 名称
2. `ToolRouter::build_tool_call()` 解析为 `ToolPayload::Mcp`
3. `McpHandler::handle()` 调用 `McpConnectionManager::call_tool()`
4. `RmcpClient` 发送请求到 MCP 服务器
5. 结果返回给模型

### 交互式执行 (Unified Exec)
1. `UnifiedExecHandler::handle()` 接收 `exec_command` 调用
2. `UnifiedExecSessionManager::exec_command()`
3. `open_session_with_sandbox()` 创建或重用 PTY 会话
4. `UnifiedExecRuntime` 处理审批和沙箱
5. `codex_utils_pty::spawn_pty_process()` 启动进程
6. `UnifiedExecSession` 缓冲输出
7. `write_stdin()` 可用于后续交互
8. 会话在空闲时保留，直到退出或被清理

## 错误处理
- `CodexErr`: 集中式错误类型，涵盖：
  - 内部错误（代理死亡）
  - 沙箱错误（超时、拒绝、信号）
  - API 错误（流断开、配额超限）
  - 环境错误（缺少变量）
- `ToolError`: 工具运行时错误（被拒绝或 Codex 错误）
- 错误通过 `EventMsg::Error` 事件传播到 UI

## 遥测和日志
- `OtelManager`: 处理 OpenTelemetry 遥测
- `tracing`: 结构化日志记录，带字段和跨度
- 工具执行、API 调用和错误都记录遥测数据

## 性能考虑
- `compact`: 当令牌使用量超过限制时自动压缩对话历史
- `TruncationPolicy`: 根据模型上下文窗口截断输出
- `ReadinessFlag`: 门控工具调用以避免竞争条件
- `FuturesOrdered`: 并行处理工具调用，保持顺序

## 安全考虑
- `is_safe_command()`: 检查已知安全命令
- `ExecPolicy`: 基于前缀的命令执行策略
- `SandboxPermissions`: 控制沙箱权限（`UseDefault`, `RequireEscalated`）
- 审批策略：`Never`, `OnRequest`, `OnFailure`, `UnlessTrusted`
- `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`: 控制网络访问

## 测试
- `make_session_and_context()`: 测试辅助函数
- `snapshot tests`: 验证渲染输出
- `pretty_assertions`: 清晰的断言差异
- 跳过沙箱依赖的测试（`skip_if_sandbox!`）

## 关键文件路径
- `codex-rs/core/src/codex.rs`: 核心会话和任务逻辑
- `codex-rs/core/src/client.rs`: 模型客户端和 API 通信
- `codex-rs/core/src/config/mod.rs`: 配置加载和管理
- `codex-rs/core/src/tools/`: 工具系统（路由、处理器、运行时）
- `codex-rs/core/src/exec.rs`: 命令执行和沙箱环境
- `codex-rs/core/src/sandboxing/`: 平台特定沙箱实现
- `codex-rs/core/src/mcp/`: MCP 客户端和连接管理
- `codex-rs/core/src/unified_exec/`: 交互式 PTY 执行
- `codex-rs/core/src/skills/`: 技能加载和管理
- `codex-rs/core/src/rollout/`: 会话记录和回放