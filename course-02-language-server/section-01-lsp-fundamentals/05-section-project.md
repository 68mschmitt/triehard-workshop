# Section 1 Project: Manual Protocol Walkthrough

## Objective

Before writing any code, understand LSP by manually playing the role of client and server. This builds intuition for the protocol that will guide your implementation.

## Requirements

1. **Trace a real LSP session** between Neovim and an existing server
2. **Manually construct LSP messages** and understand each field
3. **Document the message flow** for your target features
4. **Create a protocol test script** for future testing

## Part 1: Trace an Existing Server

### Setup

Install a simple LSP server to trace. Good options:
- `lua-language-server` (if you have it)
- `typescript-language-server` (npm install -g typescript-language-server)
- Any server you already have

### Enable LSP Logging in Neovim

```lua
-- Add to init.lua or run in command mode
vim.lsp.set_log_level("debug")

-- Find the log file
print(vim.lsp.get_log_path())
-- Usually: ~/.local/state/nvim/lsp.log
```

### Capture a Session

1. Open Neovim
2. Open a file that triggers the LSP server
3. Type some text
4. Trigger completion (Ctrl+Space or start typing)
5. Close the file and Neovim

### Analyze the Log

Open the log file and identify:

```
[ ] initialize request from client
[ ] initialize response from server (note capabilities)
[ ] initialized notification
[ ] textDocument/didOpen
[ ] textDocument/publishDiagnostics (server → client)
[ ] textDocument/completion request
[ ] textDocument/completion response
[ ] textDocument/didChange
[ ] shutdown request
[ ] exit notification
```

**Deliverable:** Write down the sequence of messages for one editing session.

## Part 2: Construct Messages by Hand

### Initialize Request

Write out the initialize request your server will receive:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "processId": ___,
    "rootUri": "file:///home/user/myproject",
    "capabilities": {
      "textDocument": {
        "completion": {},
        "publishDiagnostics": {}
      }
    }
  }
}
```

### Initialize Response

Write out what your server should respond:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "capabilities": {
      "textDocumentSync": ___,
      "completionProvider": {
        "triggerCharacters": []
      },
      "codeActionProvider": ___
    }
  }
}
```

### Document Open

Write the didOpen notification for opening a file with content "Hello wrold":

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didOpen",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/test.txt",
      "languageId": "plaintext",
      "version": 1,
      "text": "Hello wrold"
    }
  }
}
```

### Publish Diagnostics

Write the diagnostic notification your server should send for the misspelled "wrold":

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///home/user/test.txt",
    "diagnostics": [
      {
        "range": {
          "start": { "line": 0, "character": ___ },
          "end": { "line": 0, "character": ___ }
        },
        "severity": ___,
        "code": "wordlib.unknown",
        "source": "wordlib",
        "message": "Unknown word: wrold"
      }
    ]
  }
}
```

### Completion Request

Write the completion request when cursor is after "Hel" in "Hello":

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "textDocument/completion",
  "params": {
    "textDocument": { "uri": "file:///home/user/test.txt" },
    "position": { "line": 0, "character": 3 }
  }
}
```

### Completion Response

Write what your server should return:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": [
    {
      "label": "Hello",
      "kind": 1,
      "detail": "from dictionary"
    },
    {
      "label": "Help",
      "kind": 1
    }
  ]
}
```

**Deliverable:** Complete JSON for each message type.

## Part 3: Document Your Protocol Flow

Create a diagram of the message flow for your server's main features:

### Diagnostic Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     DIAGNOSTIC FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client                              Server                  │
│    │                                    │                    │
│    │─── didOpen ──────────────────────▶│                    │
│    │                                    │ tokenize text      │
│    │                                    │ check each word    │
│    │                                    │ build diagnostics  │
│    │◀── publishDiagnostics ────────────│                    │
│    │                                    │                    │
│    │─── didChange ────────────────────▶│                    │
│    │                                    │ re-tokenize        │
│    │                                    │ re-check           │
│    │◀── publishDiagnostics ────────────│                    │
│    │                                    │                    │
└─────────────────────────────────────────────────────────────┘
```

### Completion Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     COMPLETION FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client                              Server                  │
│    │                                    │                    │
│    │─── completion ───────────────────▶│                    │
│    │    (uri, position)                 │ get document text  │
│    │                                    │ extract prefix     │
│    │                                    │ call engine API    │
│    │                                    │ format results     │
│    │◀── completion response ───────────│                    │
│    │    (items[])                       │                    │
│    │                                    │                    │
└─────────────────────────────────────────────────────────────┘
```

### Code Action Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    CODE ACTION FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client                              Server                  │
│    │                                    │                    │
│    │─── codeAction ───────────────────▶│                    │
│    │    (uri, range, diagnostics)       │ find diagnostic    │
│    │                                    │ extract word       │
│    │                                    │ create action      │
│    │◀── codeAction response ───────────│                    │
│    │    (actions[])                     │                    │
│    │                                    │                    │
│    │─── executeCommand ───────────────▶│ (if using commands)│
│    │    (command, args)                 │ add to dictionary  │
│    │                                    │ re-check document  │
│    │◀── publishDiagnostics ────────────│                    │
│    │                                    │                    │
└─────────────────────────────────────────────────────────────┘
```

**Deliverable:** Flow diagrams for diagnostics, completion, and code actions.

## Part 4: Create Test Script

Write a shell script that sends a minimal LSP session to stdin:

```bash
#!/bin/bash
# test_protocol.sh - Test basic LSP protocol handling

send_message() {
    local msg="$1"
    local len=${#msg}
    printf "Content-Length: %d\r\n\r\n%s" "$len" "$msg"
}

# Initialize
send_message '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"processId":1234,"rootUri":null,"capabilities":{}}}'

sleep 0.5

# Initialized notification
send_message '{"jsonrpc":"2.0","method":"initialized","params":{}}'

sleep 0.5

# Open document
send_message '{"jsonrpc":"2.0","method":"textDocument/didOpen","params":{"textDocument":{"uri":"file:///test.txt","languageId":"plaintext","version":1,"text":"Hello wrold"}}}'

sleep 1

# Shutdown
send_message '{"jsonrpc":"2.0","id":2,"method":"shutdown","params":null}'

sleep 0.5

# Exit
send_message '{"jsonrpc":"2.0","method":"exit"}'
```

Run it:
```bash
chmod +x test_protocol.sh
./test_protocol.sh | ./your-server 2>&1 | tee output.log
```

**Deliverable:** Working test script that can drive your server.

## Acceptance Criteria

### Protocol Understanding
- [ ] Can explain difference between request and notification
- [ ] Can identify all fields in initialize request/response
- [ ] Understand what capabilities you need to declare

### Message Construction
- [ ] Correctly formatted initialize response
- [ ] Correctly formatted diagnostic notification
- [ ] Correctly formatted completion response
- [ ] Correctly formatted code action response

### Flow Documentation
- [ ] Diagnostic flow diagram complete
- [ ] Completion flow diagram complete  
- [ ] Code action flow diagram complete

### Test Infrastructure
- [ ] Test script sends valid LSP messages
- [ ] Script handles Content-Length framing
- [ ] Script waits appropriately between messages

## Hints

### Calculate Content-Length

The Content-Length header counts bytes, not characters:
```bash
msg='{"jsonrpc":"2.0","method":"exit"}'
echo ${#msg}  # Bash string length
```

For UTF-8 JSON, this works. Be careful with special characters.

### JSON Validation

Validate your hand-written JSON:
```bash
echo '{"jsonrpc":"2.0","id":1}' | jq .
```

### Reading Server Output

Your server writes to stdout. Capture it:
```bash
./test_protocol.sh | ./your-server > response.log 2> error.log
```

## What You'll Learn

By completing this project, you'll have:
- Deep understanding of LSP message structure
- Clear mental model of client-server interaction
- Test infrastructure for development
- Documentation to reference during implementation

## Next Section Preview

Section 2 builds the transport layer—reading and writing these messages over stdio with proper framing. You'll implement the Content-Length header parsing and JSON serialization that makes your test script actually work.
