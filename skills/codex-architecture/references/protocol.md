# Protocol Module - Protocol Type Definitions

## Overview
`codex-protocol` defines the protocol types used by Codex CLI, including internal communication types and external API types.

## Key Functions
- Defines internal communication types (codex-core ↔ codex-tui)
- Defines external API types (codex app-server)
- Provides type-safe protocol definitions
- Minimal dependencies

## Design Principles
- **Minimal dependencies**: Avoids introducing too many external dependencies
- **No business logic**: Only defines types, no business logic
- **Extensibility**: Adds functionality through Ext-style traits

## Related Components
- `codex-protocol` (crate)
- `protocol/` (directory)

## Architecture Relationships
As a type layer, used by:
- `codex-core` - Core business logic
- `codex-tui` - Terminal UI
- `codex-app-server` - Application server
- `codex-cli` - CLI tool

## Type Categories
1. **Internal types**: Component communication
2. **External types**: API interface definitions
3. **Shared types**: Cross-component usage

## Advantages
- Type safety
- Compile-time checks
- Clear interface definitions
- Easy maintenance and extension