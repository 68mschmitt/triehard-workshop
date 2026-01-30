# LSP Architecture Overview

## Why You Need This

Your requirements say: "Implement a fully functional Language Server Protocol adapter for the Word Library Engine." Before writing code, you need to understand what LSP actually is and why it exists. This mental model will guide every design decision.

## What to Learn

### The Problem LSP Solves

Before LSP, editor support for a language required N×M implementations:

```
Editors:     Vim, Emacs, VS Code, Sublime, Atom, ...
Languages:   C, Python, Rust, TypeScript, Go, ...

Without LSP: Each (editor, language) pair needs custom code
             10 editors × 20 languages = 200 integrations
```

LSP introduces a standard protocol:

```
With LSP:    Each editor implements LSP client (10 implementations)
             Each language implements LSP server (20 implementations)
             Total: 30 implementations, all interoperable
```

### Client-Server Model

```
┌─────────────────────────────────────────────────────┐
│                      Editor                          │
│  (Neovim, VS Code, Emacs, etc.)                     │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │               LSP Client                        │ │
│  │  - Sends requests/notifications                │ │
│  │  - Receives responses/notifications            │ │
│  │  - Renders diagnostics, completions, etc.      │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
                          │
                    (stdio / TCP)
                   JSON-RPC messages
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                   LSP Server                         │
│  (Your wordlib-lsp)                                 │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │            Protocol Handler                     │ │
│  │  - Parses JSON-RPC messages                    │ │
│  │  - Routes to appropriate handlers              │ │
│  │  - Formats responses                           │ │
│  └────────────────────────────────────────────────┘ │
│                          │                          │
│                          ▼                          │
│  ┌────────────────────────────────────────────────┐ │
│  │         Language-Specific Logic                 │ │
│  │  (Your Word Library Engine)                    │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### The Adapter Pattern

Your LSP server is an **adapter**—it translates between two interfaces:

```c
// What the editor speaks (LSP)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/completion",
  "params": {
    "textDocument": { "uri": "file:///path/to/doc.txt" },
    "position": { "line": 5, "character": 10 }
  }
}

// What your engine speaks (C API)
size_t wordlib_complete(const WordLib *wl, const char *prefix,
                        const char **results, size_t max);
```

The adapter's job:
1. Parse the LSP request
2. Extract the prefix from document at position
3. Call `wordlib_complete()`
4. Format results as LSP completion items
5. Send the response

### What the Adapter Owns

**The LSP server SHALL own:**
- Protocol parsing and message framing
- Document text state and version tracking
- Coordinate conversion (byte offsets ↔ LSP positions)
- Response formatting

**The LSP server SHALL NOT own:**
- Spell checking logic
- Completion algorithms
- Suggestion generation
- Dictionary data structures

This is your requirements document's "Adapter-Only Responsibility" constraint.

### Key LSP Concepts

**Documents:** Files open in the editor, identified by URIs (`file:///path/to/file.txt`)

**Positions:** Line/character coordinates (0-indexed, characters in UTF-16 code units)

**Ranges:** Start position + end position (like a text selection)

**Diagnostics:** Problems reported to the editor (errors, warnings, info)

**Capabilities:** Features the server supports (declared during initialization)

### Communication Flow

```
Editor                               Server
  │                                    │
  │────── initialize request ─────────▶│
  │◀───── initialize response ─────────│
  │                                    │
  │────── initialized notification ───▶│
  │                                    │
  │────── didOpen notification ───────▶│
  │◀───── publishDiagnostics ──────────│
  │                                    │
  │────── completion request ─────────▶│
  │◀───── completion response ─────────│
  │                                    │
  │────── didChange notification ─────▶│
  │◀───── publishDiagnostics ──────────│
  │                                    │
  │────── shutdown request ───────────▶│
  │◀───── shutdown response ───────────│
  │                                    │
  │────── exit notification ──────────▶│
  │                                    │
```

### Synchronous vs Asynchronous

**Requests** expect responses (have an `id`):
```json
{"jsonrpc": "2.0", "id": 1, "method": "textDocument/completion", ...}
// Server MUST respond with matching id
{"jsonrpc": "2.0", "id": 1, "result": [...]}
```

**Notifications** are fire-and-forget (no `id`):
```json
{"jsonrpc": "2.0", "method": "textDocument/didOpen", ...}
// Server processes but doesn't respond
```

### Transport Independence

LSP is transport-agnostic, but your requirements specify **stdio**:

```
Server reads from stdin  ─▶  Incoming messages
Server writes to stdout  ─▶  Outgoing messages
Server can write to stderr ─▶ Logging (doesn't interfere)
```

This is simple and works everywhere. TCP is used when server runs separately.

## Where to Learn

1. **Primary:** LSP Specification
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
   - Start with "Overview" and "Base Protocol"

2. **Reference implementations:**
   - `rust-analyzer` - Well-documented Rust implementation
   - `clangd` - C++ server, see their architecture docs
   - `gopls` - Go's official server

3. **Quick overview:**
   - Microsoft's "Language Server Protocol" page
   - "LSP Tutorial" in VS Code docs

## Practice Exercise

Trace the LSP communication for a simple session:

1. Open Neovim with an LSP server configured
2. Use `:lua vim.lsp.set_log_level("debug")` to enable logging
3. Open a file, type some text, trigger completion
4. Examine the log: `:lua vim.cmd('edit ' .. vim.lsp.get_log_path())`

Identify:
- Initialize handshake
- Document open notification
- At least one request/response pair
- Diagnostic publications

Write down the sequence of messages you observe.

## Connection to Project

Every feature you build follows this pattern:

1. Editor sends LSP message
2. Your adapter parses it
3. Your adapter calls engine API
4. Your adapter formats LSP response
5. Editor receives and displays result

Understanding this flow makes implementation straightforward. You're not inventing anything—just translating between two well-defined interfaces.
