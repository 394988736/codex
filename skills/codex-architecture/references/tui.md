# TUI Module - Terminal User Interface

## Overview
`codex-tui` is a full-screen terminal user interface built on Ratatui, providing an interactive, rich terminal experience for Codex.

## Key Functions
- **Real-time Rendering**: Continuous screen updates and animations
- **Event Handling**: Keyboard, mouse, and paste event processing
- **State Management**: Local UI state synchronized with core
- **Visual Feedback**: Progress indicators, loading states, error displays
- **Rich Text Display**: Syntax highlighting, formatted output, color support

## Technology Stack
- **Ratatui** - TUI framework for widgets and layout
- **Crossterm** - Terminal operations (raw mode, events, colors)
- **Tokio** - Async runtime for non-blocking operations
- **Tree-sitter** - Syntax highlighting for code blocks
- **Textwrap** - Text wrapping utilities
- **Clipboard** - Clipboard operations (optional feature)

## Architecture Design
### Component Structure
```
tui/
├── src/
│   ├── app.rs          - Application state and lifecycle
│   ├── event.rs        - Event handling and dispatch
│   ├── frame/          - UI frame rendering
│   │   ├── chat.rs     - Chat interface
│   │   ├── status.rs   - Status bar
│   │   └── input.rs    - Input area
│   ├── styles.rs       - Styling and themes
│   ├── wrapping.rs     - Text wrapping utilities
│   └── main.rs         - Entry point
├── styles.md           - Style guidelines
└── frames/             - UI framework components
```

### State Management
- **Local State**: UI-specific state (scroll position, focus, etc.)
- **Core State**: Mirrors relevant core application state
- **Synchronization**: Updates via events from core

### Event Flow
1. User input → Event handler
2. Event → Core action (if needed)
3. Core response → State update
4. State update → Re-render

## Core Features
### UI Components
- **Chat Interface**: Main conversation view with message history
- **Input Area**: Multi-line input with history
- **Status Bar**: Current status, model, tokens
- **Sidebar**: Optional panels (tools, context, etc.)

### Rendering Features
- **Real-time Updates**: Immediate feedback on user actions
- **Smooth Scrolling**: Efficient message navigation
- **Syntax Highlighting**: Code blocks with language-specific colors
- **Rich Formatting**: Bold, italic, colors, and styles
- **Progress Indicators**: Loading states and progress bars

### Input Handling
- **Keyboard Shortcuts**: Vim-style navigation, Emacs bindings
- **Multi-line Input**: Support for long prompts
- **Paste Handling**: Smart paste detection and formatting
- **Autocomplete**: Command and context suggestions

## Platform Support
- **Unix-like**: Full feature support (raw mode, colors, clipboard)
- **Windows**: Limited features (no raw mode in some terminals)
- **All**: Graceful degradation for unsupported features

## Dependencies
### Internal
- `codex-core`: Business logic and state
- `codex-protocol`: Type definitions
- `codex-common`: Utilities

### External
- `ratatui`: TUI framework
- `crossterm`: Terminal operations
- `tokio`: Async runtime
- `tree-sitter`: Syntax highlighting
- `textwrap`: Text wrapping

## Related Components
- `codex-tui` (crate)
- `tui/` (directory)
- `tui/styles.md` - Style guidelines
- `tui/frames/` - UI framework
- `tui/src/wrapping.rs` - Text wrapping utilities

## Architecture Relationships
- **Depends on**: `codex-core` for business logic and state
- **Provides**: Interactive user interface layer
- **Integrates with**: CLI (as subcommand), Core (state sync)

## Configuration
- **Theme**: Color scheme and styling
- **Keybindings**: Custom keyboard shortcuts
- **Behavior**: Scrollback, auto-update, etc.

## Best Practices
- Keep UI logic separate from business logic
- Use async operations for non-blocking UI
- Handle all error states gracefully
- Provide clear user feedback
- Follow style guidelines in `styles.md`