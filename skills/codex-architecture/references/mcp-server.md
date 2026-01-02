# MCP Server Module - Model Context Protocol Server

## Overview
`codex-mcp-server` implements the MCP (Model Context Protocol) server, allowing other MCP clients to use Codex as a tool.

## Key Functions
- MCP protocol server implementation
- Serves as a tool service for other agents
- Supports MCP client connections
- Experimental functionality

## Startup Methods
```bash
# Direct startup
codex mcp-server

# Debug with MCP Inspector
npx @modelcontextprotocol/inspector codex mcp-server
```

## Management Commands
```bash
# Add/list/get/remove MCP server configurations
codex mcp add <server>
codex mcp list
codex mcp get <server>
codex mcp remove <server>
```

## Configuration
Configure `mcp_servers` in `~/.codex/config.toml`.

## Related Components
- `codex-mcp-server` (crate)
- `mcp-server/` (directory)
- `mcp-types/` (type definitions)

## Architecture Relationships
- Depends on `codex-core` for business logic
- Uses `mcp-types` for protocol type definitions
- Runs as a standalone binary
- Can be called by other MCP clients

## Use Cases
- Multi-agent collaboration
- Toolchain integration
- External system calls
- Experimental feature testing