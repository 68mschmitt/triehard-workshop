# Course 2: Word Library Language Server

**Build a production-quality LSP server for the Word Library Engine**

---

## Overview

This course takes your Word Library Engine from Course 1 and wraps it in a fully functional Language Server Protocol (LSP) implementation. You'll build a thin adapter that exposes word checking, completion, and suggestions through standard LSP mechanisms—making your engine usable in Neovim, VS Code, and any LSP-capable editor.

By the end, you'll have a working language server that can:
- Highlight unknown words as diagnostics in real-time
- Provide prefix-based word completions as you type
- Offer code actions to add words to your dictionary
- Handle document synchronization efficiently
- Negotiate capabilities with any LSP client
- Persist dictionary changes safely across sessions

---

## Target Audience

Software engineers who:
- Completed Course 1 (or have equivalent C experience)
- Want to understand LSP at the protocol level
- Are interested in editor tooling and integrations
- Prefer learning by building over abstract documentation

**Prerequisites:** Course 1 completion (working Word Library Engine) or equivalent.

---

## Course Structure

| Section | Focus | You'll Build |
|---------|-------|--------------|
| 1 | LSP Fundamentals | Protocol understanding |
| 2 | JSON-RPC & Transport | Message framing layer |
| 3 | Document Management | Document state tracker |
| 4 | Diagnostics | Unknown word highlighting |
| 5 | Completions | Word auto-completion |
| 6 | Code Actions | Dictionary management |
| 7 | Configuration & Capabilities | Settings negotiation |
| 8 | Robustness & Production | Error handling & logging |

---

## Section Details

### Section 1: LSP Fundamentals
*Understand the protocol before implementing it*

- **01** - LSP Architecture Overview (client-server model, adapter pattern)
- **02** - Message Types (requests, responses, notifications)
- **03** - Server Lifecycle (initialize, shutdown, exit)
- **04** - Capability Negotiation (declaring what you support)
- **Project** - Manual protocol walkthrough

### Section 2: JSON-RPC & Transport
*Build reliable message passing over stdio*

- **01** - JSON-RPC 2.0 Basics (structure, ids, error codes)
- **02** - LSP Framing Protocol (Content-Length headers)
- **03** - Stdio Communication (reading, writing, buffering)
- **04** - JSON Parsing in C (library selection, patterns)
- **Project** - Working transport layer

### Section 3: Document Management
*Track open documents and their contents*

- **01** - Document Lifecycle Events (open, change, close)
- **02** - Full vs Incremental Sync (trade-offs, implementation)
- **03** - Document State Storage (version tracking, text management)
- **04** - URI Handling (file URIs, path normalization)
- **Project** - Document store implementation

### Section 4: Diagnostics
*Report unknown words to the editor*

- **01** - Diagnostic Structure (range, severity, code, message)
- **02** - Publishing Diagnostics (notification flow)
- **03** - Byte Span to LSP Position (coordinate conversion)
- **04** - Diagnostic Stability (avoiding flicker)
- **Project** - Complete diagnostics provider

### Section 5: Completions
*Provide word suggestions as users type*

- **01** - Completion Request Flow (trigger, response format)
- **02** - Completion Items (labels, kinds, insert text)
- **03** - Cursor Context (word boundaries, prefix extraction)
- **04** - Result Limiting (performance, relevance)
- **Project** - Full completion provider

### Section 6: Code Actions
*Let users add words to the dictionary*

- **01** - Code Action Fundamentals (kinds, context, commands)
- **02** - Dictionary Management Actions (add word, ignore)
- **03** - Triggering Revalidation (refresh after changes)
- **04** - Workspace Edits (if supporting word replacement)
- **Project** - Code action provider

### Section 7: Configuration & Capabilities
*Make your server configurable*

- **01** - Server Capabilities (advertising features)
- **02** - Client Configuration (didChangeConfiguration)
- **03** - Workspace Dictionaries (global vs project-local)
- **04** - Sensible Defaults (zero-config operation)
- **Project** - Complete configuration system

### Section 8: Robustness & Production
*Build something you can rely on*

- **01** - Error Handling Strategy (graceful degradation)
- **02** - Logging Without Interference (stderr, log files)
- **03** - Resource Management (memory bounds, cleanup)
- **04** - Testing LSP Servers (replay, mocking)
- **Project** - Production-ready server

---

## Time Estimate

**Total: 6-10 weeks** (adjustable to your pace)

| Phase | Sections | Time |
|-------|----------|------|
| Protocol Basics | 1-2 | 2 weeks |
| Core Features | 3-5 | 3-4 weeks |
| Advanced Features | 6-7 | 2 weeks |
| Polish | 8 | 1-2 weeks |

Each section has ~4 learning topics plus a hands-on project. Plan for 2-4 hours per topic.

---

## Learning Approach

Each topic file follows this structure:

1. **Why You Need This** - Connects to your end goal
2. **What to Learn** - Core concepts with code examples
3. **Where to Learn** - LSP spec, reference implementations, articles
4. **Practice Exercise** - Hands-on task (30-60 minutes)
5. **Connection to Project** - How it fits the bigger picture

**Philosophy:**
- The engine stays untouched—you're building an adapter
- Learn the protocol by implementing it
- Every section produces testable functionality
- Ship something real, not just exercises

---

## What You'll Have Built

```
wordlib-lsp/
├── include/
│   ├── lsp_server.h       # Public server API
│   ├── transport.h        # Message I/O
│   ├── document_store.h   # Open document tracking
│   └── json.h             # JSON utilities
├── src/
│   ├── main.c             # Entry point
│   ├── transport.c        # Stdio framing
│   ├── dispatcher.c       # Request routing
│   ├── lifecycle.c        # Initialize/shutdown
│   ├── documents.c        # Document state
│   ├── diagnostics.c      # Word checking → diagnostics
│   ├── completion.c       # Prefix → completion items
│   ├── code_actions.c     # Add-to-dictionary actions
│   └── config.c           # Settings management
├── test/
│   └── *.c                # Protocol tests
└── Makefile
```

**Integration with Neovim:**
```lua
-- init.lua
vim.lsp.start({
  name = 'wordlib',
  cmd = { '/path/to/wordlib-lsp' },
  filetypes = { 'text', 'markdown' },
})
```

---

## Architectural Principle

This course enforces strict separation:

```
┌─────────────────────────────────────────────────────┐
│                   LSP Server (this course)          │
│  - Protocol parsing & framing                       │
│  - Document state management                        │
│  - Position coordinate conversion                   │
│  - LSP-compliant responses                          │
└─────────────────────────────────────────────────────┘
                          │
                   (public API calls)
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│               Word Library Engine (Course 1)        │
│  - Spell checking logic                             │
│  - Completion algorithms                            │
│  - Suggestion generation                            │
│  - Dictionary management                            │
└─────────────────────────────────────────────────────┘
```

**The engine has no idea it's being used by an LSP server.** That's the goal.

---

## Success Criteria

Your LSP implementation is complete when:

| Feature | Verification |
|---------|--------------|
| Unknown words highlighted | Open file, see squiggles |
| Completions appear | Type prefix, see suggestions |
| Code actions work | Click lightbulb, add word |
| No engine modifications | Engine code unchanged |
| Works in Neovim | Real editor integration |
| Handles edge cases | No crashes on weird input |

---

## Performance Targets

| Operation | Target | Notes |
|-----------|--------|-------|
| Diagnostic publish | < 100ms | After document change |
| Completion response | < 50ms | From request to response |
| Memory per document | ~2x text size | Document + metadata |
| Server startup | < 200ms | Including engine init |

---

## Next Steps After This Course

This LSP server is designed for extension:

- **Course 3** (potential): Editor Integration Deep Dive
  - Neovim plugin development
  - VS Code extension
  - Custom UI features

- **Extensions:**
  - Grammar checking integration
  - Multi-language support
  - Remote dictionary sync

---

## Getting Started

1. Ensure your Course 1 engine compiles and passes tests
2. Read `section-01-lsp-fundamentals/01-lsp-architecture-overview.md`
3. Work through each topic in order
4. Complete the section project before moving on
5. Test with Neovim after Section 4 (diagnostics)

**Remember:** You're building a thin adapter. When tempted to add features to the engine, stop—that's not this course's job.

---

## File Listing

```
course-02-language-server/
├── README.md (this file)
├── section-01-lsp-fundamentals/
│   ├── 01-lsp-architecture-overview.md
│   ├── 02-message-types.md
│   ├── 03-server-lifecycle.md
│   ├── 04-capability-negotiation.md
│   └── 05-section-project.md
├── section-02-json-rpc-transport/
│   ├── 01-json-rpc-basics.md
│   ├── 02-lsp-framing-protocol.md
│   ├── 03-stdio-communication.md
│   ├── 04-json-parsing-in-c.md
│   └── 05-section-project.md
├── section-03-document-management/
│   ├── 01-document-lifecycle-events.md
│   ├── 02-full-vs-incremental-sync.md
│   ├── 03-document-state-storage.md
│   ├── 04-uri-handling.md
│   └── 05-section-project.md
├── section-04-diagnostics/
│   ├── 01-diagnostic-structure.md
│   ├── 02-publishing-diagnostics.md
│   ├── 03-byte-span-to-lsp-position.md
│   ├── 04-diagnostic-stability.md
│   └── 05-section-project.md
├── section-05-completions/
│   ├── 01-completion-request-flow.md
│   ├── 02-completion-items.md
│   ├── 03-cursor-context.md
│   ├── 04-result-limiting.md
│   └── 05-section-project.md
├── section-06-code-actions/
│   ├── 01-code-action-fundamentals.md
│   ├── 02-dictionary-management-actions.md
│   ├── 03-triggering-revalidation.md
│   ├── 04-workspace-edits.md
│   └── 05-section-project.md
├── section-07-configuration-capabilities/
│   ├── 01-server-capabilities.md
│   ├── 02-client-configuration.md
│   ├── 03-workspace-dictionaries.md
│   ├── 04-sensible-defaults.md
│   └── 05-section-project.md
└── section-08-robustness-production/
    ├── 01-error-handling-strategy.md
    ├── 02-logging-without-interference.md
    ├── 03-resource-management.md
    ├── 04-testing-lsp-servers.md
    └── 05-section-project.md
```

---

*Happy protocol hacking!* 🔌
