# CLI 模块 - 命令行接口

## 概述
`codex-cli` 是 Codex 项目的主要命令行工具，提供多个子命令用于执行不同任务。它作为所有 CLI 交互的统一入口点和协调器。

## 核心功能
- **命令解析**: 使用 `clap` 进行参数解析和验证
- **子命令分发**: 将命令路由到相应的处理程序
- **统一入口**: 单一二进制文件 (`codex`) 执行所有操作
- **配置加载**: 读取并验证配置文件
- **错误报告**: 格式化并显示错误信息给用户

## 支持的子命令

### 交互式命令
- **`codex`** (默认行为) - 启动 TUI 界面
  - 支持 `--profile` 指定配置文件
  - 支持 `--cwd` 指定工作目录
  - 支持 `--search` 启用网络搜索
  - 支持 `--sandbox` 设置沙箱模式
  - 支持 `--ask-for-approval` 设置审批策略
  - 支持 `--full-auto` 启用全自动模式
  - 支持 `--dangerously-bypass-approvals-and-sandbox` 绕过安全检查
  - 支持 `-i/--images` 添加图片文件
  - 支持 `-a/--add-dir` 添加目录
  - 支持 `-p/--prompt` 添加提示词

### 非交互式命令
- **`codex exec`** - 非交互式执行
  - 支持 `--review` 代码审查模式
  - 支持 `--dry-run` 干运行
  - 支持 `--no-multitool` 禁用多工具模式
  - 支持 `--stream` 流式输出
  - 支持 `--json` JSON 输出
  - 支持 `--continue` 继续会话
  - 支持 `--resume` 恢复会话
  - 支持 `--images` 添加图片
  - 支持 `--add-dir` 添加目录
  - 支持 `--prompt` 添加提示词

- **`codex review`** - 代码审查 (本质上是 `exec --review` 的别名)

### 登录管理
- **`codex login`** - 管理登录
  - `--with-api-key` - 从 stdin 读取 API 密钥
  - `--device-auth` - 使用设备码 OAuth 流程
  - `status` - 显示登录状态
  - 支持 `--experimental_issuer` 和 `--experimental_client-id` 用于高级 OAuth 配置

- **`codex logout`** - 移除存储的认证凭据

### MCP (Model Context Protocol) 管理
- **`codex mcp`** - MCP 服务器管理
  - `list` - 列出配置的服务器
  - `get <name>` - 显示单个服务器配置
  - `add <name> -- <command>...` - 添加 stdio 服务器
  - `add <name> --url <URL>` - 添加流式 HTTP 服务器
  - `remove <name>` - 删除服务器配置
  - `login <name>` - OAuth 登录
  - `logout <name>` - OAuth 登出

- **`codex mcp-server`** - 运行 MCP 服务器 (stdio 传输)

### 应用服务器
- **`codex app-server`** - 运行应用服务器或相关工具
  - `generate-ts -o <DIR>` - 生成 TypeScript 绑定
  - `generate-json-schema -o <DIR>` - 生成 JSON Schema

### 沙箱执行
- **`codex sandbox`** - 在沙箱中运行命令
  - `macos/seatbelt` - macOS Seatbelt 沙箱
  - `linux/landlock` - Linux Landlock+seccomp 沙箱
  - `windows` - Windows 受限令牌沙箱
  - 支持 `--full-auto` 自动执行模式
  - 支持 `--log-denials` (macOS) 记录拒绝日志

### 其他工具
- **`codex apply`** - 应用 Codex 生成的 diff 到本地工作树
- **`codex resume`** - 恢复之前的交互式会话
  - `--last` - 继续最近的会话
  - `--all` - 显示所有会话
  - 支持所有 TUI 配置选项

- **`codex cloud`** - 浏览 Codex Cloud 任务并应用更改
- **`codex execpolicy`** - 执行策略工具
  - `check` - 检查命令是否符合策略

- **`codex completion`** - 生成 shell 补全脚本
- **`codex features`** - 检查功能标志
  - `list` - 列出所有已知功能及其状态

## 配置系统

### 配置加载优先级
1. 环境变量
2. CLI 参数 (`-c key=value`)
3. 配置文件 (`~/.codex/config.toml`)
4. 特定命令的配置覆盖

### 功能标志
- 支持 `--enable/--disable` 启用/禁用功能
- 支持 `--profile` 指定配置文件
- 支持 `--search` 启用网络搜索功能
- 支持 TUI v2 实验性功能

## 安全特性

### 沙箱支持
- **macOS Seatbelt**: 使用系统沙箱策略限制进程权限
- **Linux Landlock**: 使用 Landlock 和 seccomp 限制系统调用
- **Windows**: 使用受限令牌和作业对象限制进程

### 认证管理
- API 密钥存储在安全的凭据存储中
- OAuth 设备码流支持
- ChatGPT 登录支持
- 支持自定义 OAuth 配置

## 架构设计

### 命令结构
```rust
#[derive(Parser)]
#[command(name = "codex")]
struct MultitoolCli {
    #[clap(flatten)]
    pub config_overrides: CliConfigOverrides,
    #[clap(flatten)]
    pub feature_toggles: FeatureToggles,
    #[clap(flatten)]
    interactive: TuiCli,
    #[clap(subcommand)]
    subcommand: Option<Subcommand>,
}

#[derive(Debug, clap::Subcommand)]
enum Subcommand {
    Exec(ExecCli),
    Review(ReviewArgs),
    Login(LoginCommand),
    Logout(LogoutCommand),
    Mcp(McpCli),
    McpServer,
    AppServer(AppServerCommand),
    Completion(CompletionCommand),
    Sandbox(SandboxArgs),
    Execpolicy(ExecpolicyCommand),
    Apply(ApplyCommand),
    Resume(ResumeCommand),
    Cloud(CloudTasksCli),
    Features(FeaturesCli),
    // ... 内部命令
}
```

### 集成模式
1. **解析**: 使用 clap 解析参数
2. **路由**: 分发到适当的处理程序
3. **执行**: 处理程序调用核心功能
4. **输出**: 格式化并显示结果

### 核心依赖
- `codex-core` - 核心业务逻辑和协调
- `codex-tui` / `codex-tui2` - 终端用户界面
- `codex-exec` - 非交互式执行引擎
- `codex-mcp-server` - MCP 服务器实现
- `codex-login` - 认证处理
- `codex-protocol` - 协议类型定义
- `clap` - 命令行参数解析
- `anyhow` - 错误处理

## 架构特点
- **单一二进制**: 所有功能集成在一个可执行文件中
- **多 crate 集成**: 为每个子命令使用专门的 crate
- **统一配置**: 所有命令使用一致的配置加载机制
- **错误处理**: 带上下文的集中式错误报告
- **帮助系统**: 所有命令和子命令的全面帮助文本

## 实现细节

### 入口点
- `cli/src/main.rs`: 二进制入口点，设置 CLI 并分发命令
- `cli/src/lib.rs`: 用于程序化使用的库接口
- `cli/src/mcp_cmd.rs`: MCP 子命令实现
- `cli/src/login.rs`: 登录相关功能
- `cli/src/debug_sandbox.rs`: 沙箱执行逻辑
- `cli/src/exit_status.rs`: 退出状态处理

### 命令处理流程
1. `main()` 初始化 CLI 解析器
2. 解析参数并确定命令
3. 调用相应的处理函数
4. 处理程序使用核心功能
5. 返回结果或以错误代码退出

### 配置加载
- 从 `~/.codex/config.toml` 加载
- 支持环境变量覆盖
- 支持 CLI 参数覆盖 (`-c key=value`)
- 在命令执行前验证配置

## 相关文件
- `cli/Cargo.toml` - 依赖项和元数据
- `cli/src/main.rs` - 二进制入口点
- `cli/src/lib.rs` - 库接口
- `cli/src/mcp_cmd.rs` - MCP 命令实现
- `cli/src/login.rs` - 登录功能
- `cli/src/debug_sandbox.rs` - 沙箱功能
- `cli/tests/` - 测试文件

## 最佳实践
- CLI 保持轻量，将核心逻辑委托给专门的 crate
- 使用 clap 的 derive 特性实现类型安全的命令定义
- 提供清晰的错误信息和上下文
- 包含全面的帮助文本
- 在命令执行前尽早验证输入
- 支持所有平台的沙箱功能