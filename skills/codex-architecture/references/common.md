# Common Module - Shared Utilities

## Overview
`codex-common` provides shared utilities and helper functions across crates.

## Key Functions
- Shared utilities across components
- Feature-gated functionality
- Avoids code duplication

## Design Patterns
- **Feature gating**: Defines different features via `[features]`
- **Conditional compilation**: Uses `#[cfg]` to control feature enablement
- **Lightweight**: Contains only common utilities, no business logic

## Common Features
- CLI-related utilities
- Time measurement tools
- Sandbox summary generation
- Other general helper functions

## Related Components
- `codex-common` (crate)
- `common/` (directory)

## Architecture Relationships
As infrastructure layer, depended on by almost all other crates:
- `codex-core`
- `codex-tui`
- `codex-exec`
- `codex-cli`
- `codex-mcp-server`

## Usage
In other crates' `Cargo.toml`:
```toml
[dependencies]
codex-common = { workspace = true, features = ["cli", "elapsed"] }
```

## Advantages
- Reduces code duplication
- Unified utility functions
- Flexible feature combination
- Cross-component consistency
 
 ## Test Addition
 This line was added to test file editing capabilities.
