# Backend Client Module - Backend Client

## Overview
`codex-backend-client` 是一个用于与 Codex 后端服务通信的客户端实现模块。它封装了 HTTP 请求和响应处理，提供了类型安全的 API 调用接口，并支持多种后端服务路径风格。

## Key Functions
- **API 通信**: 与 Codex 后端 API 进行通信，支持 `/api/codex/...` 和 `/wham/...` 两种路径风格
- **HTTP 请求处理**: 使用 reqwest 库处理 HTTP 请求和响应，支持 GET、POST 等方法
- **TLS 加密**: 通过 Rustls 实现安全的 TLS 加密通信
- **认证管理**: 支持 Bearer Token 认证和 ChatGPT 账户 ID 头信息
- **速率限制查询**: 提供获取 API 使用情况和速率限制的接口
- **任务管理**: 支持创建任务、查询任务详情、列出任务列表等操作
- **数据转换**: 将后端 API 返回的数据转换为类型安全的 Rust 结构体

## Technology Stack
- **Reqwest** - HTTP 客户端库，支持异步请求和 JSON 处理
- **Rustls** - TLS 实现，提供安全的加密通信
- **Serde** - 序列化/反序列化库，用于 JSON 数据处理
- **OpenAPI 模型** - 基于 OpenAPI 规范生成的 API 类型定义
- **Tokio** - 异步运行时（通过 reqwest 间接依赖）

## Core Dependencies
- `codex-backend-openapi-models` - API 数据模型定义
- `codex-protocol` - 协议类型定义，包括账户计划类型、速率限制快照等
- `codex-core` - 核心逻辑，包括认证管理和默认客户端配置

## Related Components
- `codex-backend-client` (crate) - Rust 库包
- `backend-client/` (directory) - 源代码目录
- `codex-backend-openapi-models/` - API 模型定义
- `codex-core` - 核心依赖，提供认证和默认配置

## Architecture Relationships
作为网络通信层，连接：
- 本地业务逻辑 (`codex-core`) - 提供认证和核心功能
- 远程后端服务 (OpenAI Codex API) - 实际的 API 服务端点
- 上层应用 - 通过此客户端与后端进行交互

## Features
- **轻量级设计**: 默认不包含额外功能，仅提供必要的 HTTP 通信能力
- **路径风格支持**: 自动识别并支持两种 API 路径风格（Codex API 和 ChatGPT API）
- **类型安全**: 使用 Rust 类型系统确保 API 调用的正确性
- **错误处理**: 提供详细的错误信息，包括 HTTP 状态码、内容类型和响应体
- **扩展性**: 提供扩展 trait (`CodeTaskDetailsResponseExt`) 用于处理任务详情响应
- **测试覆盖**: 包含单元测试和测试 fixtures 确保代码质量

## API 端点支持
- `GET /api/codex/usage` 或 `/wham/usage` - 获取速率限制和使用情况
- `GET /api/codex/tasks/list` 或 `/wham/tasks/list` - 列出任务列表
- `GET /api/codex/tasks/{task_id}` 或 `/wham/tasks/{task_id}` - 获取任务详情
- `GET /api/codex/tasks/{task_id}/turns/{turn_id}/sibling_turns` 或 `/wham/tasks/{task_id}/turns/{turn_id}/sibling_turns` - 获取同级轮次
- `POST /api/codex/tasks` 或 `/wham/tasks` - 创建新任务

## 数据模型
模块定义了以下关键数据结构：
- `Client` - 主要的客户端结构体，包含配置和 HTTP 客户端
- `CodeTaskDetailsResponse` - 任务详情响应，包含用户轮次、助手轮次和差异轮次
- `Turn` - 轮次信息，包含输入输出项、工作日志和错误信息
- `TurnItem` - 轮次中的单项，包含类型、角色和内容
- `ContentFragment` - 内容片段，支持结构化和文本两种格式
- `RateLimitSnapshot` - 速率限制快照，包含主要/次要窗口信息和信用额度