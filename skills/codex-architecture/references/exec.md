# Exec模块 - 非交互式执行

## 概述
`codex-exec` 提供了Codex的非交互式执行能力，专为自动化和脚本化场景设计。该模块位于 `d:\codex\codex-rs\exec\` 目录，允许在不启动交互式TUI的情况下运行Codex，适合集成到CI/CD流水线、批处理任务和自动化脚本中。主要实现在 `d:\codex\codex-rs\exec\src\lib.rs` 和 `d:\codex\codex-rs\exec\src\main.rs`。

## 核心功能
- **非交互式任务执行**：无需用户输入，自动完成任务，实现在 `d:\codex\codex-rs\exec\src\lib.rs` 的 `run_main` 函数
- **程序化调用**：通过命令行接口以编程方式调用Codex，CLI定义在 `d:\codex\codex-rs\exec\src\cli.rs`
- **自动化工作流集成**：与现有自动化工具无缝集成，支持标准输入输出管道
- **直接终端输出**：结果直接输出到终端，便于管道操作，事件处理器在 `d:\codex\codex-rs\exec\src\event_processor\` 目录

## 使用方式

### 基本用法
```bash
# 直接提供提示词
codex exec "解释这个项目的架构"

# 通过stdin提供提示词
echo "分析代码中的潜在问题" | codex exec

# 使用文件重定向
codex exec < prompt.txt
```

### 高级用法
```bash
# 指定模型
codex exec --model gpt-5.1 "重构这段代码"

# 启用自动执行（允许文件修改）
codex exec --full-auto "修复所有lint错误"

# 完全自动模式（危险，绕过所有安全限制）
codex exec --sandbox danger-full-access "执行任意操作"

# JSON输出模式（适合程序解析）
codex exec --json "分析项目结构" | jq .

# 指定工作目录
codex exec --cd /path/to/project "检查代码质量"

# 附加图片（多模态支持）
codex exec --image diagram.png "解释这个架构图"
```

## 核心特性

### 1. 安全与权限控制
- **默认只读模式**：不修改文件，不执行网络命令
- **沙箱策略**：通过 `--sandbox` 参数控制执行权限
  - `read-only`：只读访问（默认）
  - `workspace-write`：允许工作区写入
  - `danger-full-access`：完全访问权限（危险）
- **审批策略**：非交互模式下默认不请求审批

### 2. 输出模式
- **默认模式**：活动信息输出到stderr，最终结果输出到stdout
- **JSONL模式**：`--json` 参数启用JSON Lines输出，包含所有事件流
- **结构化输出**：`--output-schema` 参数定义JSON Schema，强制结构化响应
- **输出文件**：`-o/--output-last-message` 参数指定结果保存文件

### 3. 会话管理
- **会话恢复**：`resume` 子命令恢复之前的会话
  - `codex exec resume --last`：恢复最近会话
  - `codex exec resume <SESSION_ID>`：按ID恢复
- **上下文保持**：恢复会话时保留对话上下文

### 4. 配置与覆盖
- **CLI参数覆盖**：通过 `-c` 参数覆盖配置文件设置
- **环境变量**：支持 `RUST_LOG` 控制日志级别
- **认证**：支持 `CODEX_API_KEY` 环境变量覆盖

## 支持的事件类型（JSON模式）
当使用 `--json` 参数时，会输出以下事件类型：
- `thread.started` - 线程启动或恢复
- `turn.started` - 回合开始
- `turn.completed` - 回合完成（包含用量信息）
- `turn.failed` - 回合失败（包含错误详情）
- `item.started/updated/completed` - 各类项目状态变更
- `error` - 不可恢复的错误

支持的项目类型：
- `agent_message` - 助手消息
- `reasoning` - 助手思考过程摘要
- `command_execution` - 命令执行
- `file_change` - 文件修改
- `mcp_tool_call` - MCP工具调用
- `web_search` - 网络搜索
- `todo_list` - 任务计划列表

## 架构关系

### 依赖关系
- **codex-core** (`d:\codex\codex-rs\core\`)：核心业务逻辑和协议处理
- **codex-common** (`d:\codex\codex-rs\common\`)：通用工具和配置
- **codex-protocol** (`d:\codex\codex-rs\protocol\`)：协议定义和类型
- **tokio**：异步运行时

### 组件结构
```
d:\codex\codex-rs\exec\
├── Cargo.toml                    # crate定义和依赖
├── src\
│   ├── main.rs                   # 入口点，支持arg0分发到codex-linux-sandbox
│   ├── lib.rs                    # 核心库逻辑，包含run_main函数
│   ├── cli.rs                    # CLI参数解析，定义Cli结构体和Command枚举
│   ├── event_processor\          # 事件处理器目录
│   │   ├── mod.rs
│   │   ├── event_processor_with_human_output.rs  # 人类可读输出处理
│   │   └── event_processor_with_jsonl_output.rs  # JSONL输出处理
│   └── exec_events.rs            # 事件定义
└── tests\                        # 集成测试
    ├── all.rs
    ├── event_processor_with_json_output.rs
    └── suite\                    # 测试套件
        ├── resume.rs             # 会话恢复测试
        ├── sandbox.rs            # 沙箱策略测试
        ├── apply_patch.rs        # 补丁应用测试
        └── ...                   # 其他测试
```

### 与其它模块的关系
- **codex-cli** (`d:\codex\codex-rs\cli\`)：共享核心逻辑，但提供交互式TUI
- **codex-core** (`d:\codex\codex-rs\core\`)：业务逻辑基础，提供会话管理、认证、配置
- **codex-linux-sandbox** (`d:\codex\codex-rs\linux-sandbox\`)：通过arg0分发支持独立沙箱功能
- **codex-tui** (`d:\codex\codex-rs\tui\`)：交互式终端用户界面

## 典型应用场景

### 1. CI/CD集成
```bash
# 代码审查
codex exec --json "审查本次提交的安全问题" | jq '.item.text'

# 自动修复
codex exec --full-auto "修复所有类型错误"

# 文档生成
codex exec "为这个模块生成API文档" > docs.md
```

### 2. 批处理任务
```bash
# 批量分析多个文件
for file in $(find . -name "*.rs"); do
  codex exec "分析 $file 的性能问题"
done
```

### 3. 脚本自动化
```bash
#!/bin/bash
# 自动重构脚本
codex exec --full-auto "将所有回调函数改为async/await"
codex exec resume --last "运行测试确保重构正确"
```

### 4. 结构化数据提取
```bash
# 定义JSON Schema
cat > schema.json <<EOF
{
  "type": "object",
  "properties": {
    "dependencies": {"type": "array", "items": {"type": "string"}},
    "main_language": {"type": "string"}
  },
  "required": ["dependencies", "main_language"]
}
EOF

# 提取项目信息
codex exec "分析项目依赖和主要语言" --output-schema schema.json
```

## 配置选项

### 命令行参数
CLI参数定义在 `d:\codex\codex-rs\exec\src\cli.rs` 的 `Cli` 结构体中：
- `--model, -m`：指定使用的模型
- `--oss`：使用开源提供商
- `--local-provider`：指定本地提供商（lmstudio/ollama）
- `--sandbox, -s`：沙箱策略，对应 `d:\codex\codex-rs\exec\src\lib.rs` 中的沙箱模式处理
- `--full-auto`：启用自动执行
- `--dangerously-bypass-approvals-and-sandbox`：绕过所有安全限制
- `--cd, -C`：指定工作目录
- `--skip-git-repo-check`：跳过Git仓库检查，在 `d:\codex\codex-rs\exec\src\lib.rs` 中验证
- `--add-dir`：额外可写目录
- `--output-schema`：输出模式文件路径，由 `d:\codex\codex-rs\exec\src\lib.rs` 中的 `load_output_schema` 函数处理
- `--json`：JSON输出模式，切换到 `d:\codex\codex-rs\exec\src\event_processor_with_jsonl_output.rs`
- `--output-last-message, -o`：最后消息输出文件
- `--color`：颜色设置（always/never/auto）

### 子命令
- `resume`：恢复会话，定义在 `d:\codex\codex-rs\exec\src\cli.rs` 的 `ResumeArgs` 结构体
  - `--last`：恢复最近会话，实现在 `d:\codex\codex-rs\exec\src\lib.rs` 的 `resolve_resume_path` 函数
  - `<SESSION_ID>`：按ID恢复
- `review`：代码审查，定义在 `d:\codex\codex-rs\exec\src\cli.rs` 的 `ReviewArgs` 结构体
  - `--uncommitted`：审查未提交更改
  - `--base <branch>`：与基分支比较
  - `--commit <sha>`：审查特定提交
  - `--title`：提交标题

## 错误处理与退出码
- **正常退出**：任务完成，退出码0
- **错误退出**：遇到错误，退出码1
- **中断处理**：Ctrl+C会优雅中断任务

## 性能考虑
- **异步执行**：使用Tokio异步运行时
- **事件流处理**：实时处理和输出事件
- **内存效率**：流式处理，避免大文件加载

## 限制与注意事项
1. **Git仓库要求**：默认需要在Git仓库中运行（可跳过）
2. **认证**：需要有效的API密钥或配置
3. **网络访问**：默认禁用，需明确启用
4. **文件修改**：默认只读，需明确启用写入权限
5. **会话持久化**：会话记录在 `~/.codex/sessions/` 目录

## 相关组件
- **codex-exec** (crate)：Rust库，定义在 `d:\codex\codex-rs\exec\Cargo.toml`
- **codex-exec** (二进制)：可执行文件，入口点为 `d:\codex\codex-rs\exec\src\main.rs`
- **exec/** (目录)：源代码目录 `d:\codex\codex-rs\exec\`
- **docs/exec.md**：用户文档 `d:\codex\docs\exec.md`
- **核心依赖**：
  - `d:\codex\codex-rs\core\Cargo.toml` - codex-core
  - `d:\codex\codex-rs\common\Cargo.toml` - codex-common
  - `d:\codex\codex-rs\protocol\Cargo.toml` - codex-protocol

## 架构集成
- **CLI集成**：作为子命令集成在 `d:\codex\codex-rs\cli\` 中
- **TUI集成**：与 `d:\codex\codex-rs\tui\` 共享核心逻辑但提供交互界面
- **核心依赖**：依赖 `d:\codex\codex-rs\core\` 处理业务逻辑、会话管理、认证和配置
- **MCP支持**：通过 `d:\codex\codex-rs\mcp-server\` 和 `d:\codex\codex-rs\mcp-types\` 支持Model Context Protocol
- **沙箱集成**：通过arg0分发支持 `d:\codex\codex-rs\linux-sandbox\` 独立功能
- **配置管理**：使用 `d:\codex\codex-rs\common\` 中的配置工具和 `d:\codex\codex-rs\keyring-store\` 进行认证存储