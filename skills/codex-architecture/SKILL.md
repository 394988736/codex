---
name: codex-architecture
description: Comprehensive guide to Codex project architecture and component relationships. Use when you need to understand Codex's Rust codebase structure, module dependencies, or how components like core, CLI, TUI, exec, MCP server, backend client, protocol, and common utilities fit together.
---

# Codex Architecture Skill

This skill provides comprehensive knowledge about the Codex project's Rust architecture, helping developers quickly understand the codebase structure and component relationships.


## Reference Files Overview

This section provides a brief summary of the content of each reference file mentioned in this skill.

### Core Module (`references/core.md`)
- **Content**: Detailed documentation of `codex-core`, the core business logic layer
- **Key Topics**: Architecture design, platform dependencies, module structure, agent execution flow, tool integration
- **Use Case**: Understanding the central orchestration layer and business logic implementation

### CLI Module (`references/cli.md`)
- **Content**: Comprehensive guide to `codex-cli`, the main command-line interface
- **Key Topics**: Supported subcommands, configuration system, security features, command structure, integration patterns
- **Use Case**: Learning about CLI commands, options, and how commands are processed

### TUI Module (`references/tui.md`)
- **Content**: Documentation for `codex-tui`, the terminal user interface
- **Key Topics**: Technology stack, component structure, state management, rendering features, input handling
- **Use Case**: Understanding the interactive terminal UI architecture and features

### Exec Module (`references/exec.md`)
- **Content**: Guide to `codex-exec`, the non-interactive execution engine
- **Key Topics**: Usage patterns, core features, typical scenarios, automation integration
- **Use Case**: Learning about automated/scripted Codex usage

### MCP Server Module (`references/mcp-server.md`)
- **Content**: Documentation for `codex-mcp-server`, the Model Context Protocol server
- **Key Topics**: Startup methods, management commands, configuration, use cases
- **Use Case**: Understanding how to use Codex as an MCP tool service

### Backend Client Module (`references/backend-client.md`)
- **Content**: Detailed documentation of `codex-backend-client`, the HTTP client for backend API
- **Key Topics**: API communication, technology stack, supported endpoints, data models, features
- **Use Case**: Understanding network communication layer and API integration

### Protocol Module (`references/protocol.md`)
- **Content**: Guide to `codex-protocol`, the protocol type definitions
- **Key Topics**: Type categories, design principles, architecture relationships, advantages
- **Use Case**: Learning about type definitions used across components

### Common Module (`references/common.md`)
- **Content**: Documentation for `codex-common`, shared utilities
- **Key Topics**: Feature gating, design patterns, common features, usage examples
- **Use Case**: Understanding shared utilities and how to use them across crates

### Network Request Implementation (`references/项目网络请求实现分析.md`)
- **Content**: Analysis of network request implementation in Codex project
- **Key Topics**: HTTP clients, API endpoints, authentication, error handling, performance optimization
- **Use Case**: Understanding how Codex handles network communication across different components

### SubAgent Implementation (`references/SUBAGENT_IMPLEMENTATION.md`)
- **Content**: Detailed documentation of the sub-agent system implementation
- **Key Topics**: Protocol definitions, delegate mechanism, task-specific implementations (review/compact), event forwarding, HTTP header integration
- **Use Case**: Understanding how sub-agents work, how to extend them, and how they integrate with the main agent system

### Tool Orchestration Module (`references/tool-orchestration.md`)
- **Content**: Comprehensive guide to the tool orchestration system in Codex
- **Key Topics**: Architecture design, tool specification, registry pattern, routing, execution handlers, security mechanisms, MCP integration
- **Use Case**: Understanding how tools are defined, registered, routed, and executed in the Codex system

## Related Skills

- **Skill Creator**: Use `skill-creator` skill for creating new skills
- **Rust Development**: General Rust patterns and best practices
- **Testing**: Unit and integration testing strategies
- **CLI Design**: Command-line interface design patterns
- **TUI Development**: Terminal UI development with Ratatui
- **Async Rust**: Tokio patterns and best practices