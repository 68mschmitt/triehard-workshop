# Message Types

## Why You Need This

Your server will handle dozens of different message types. Understanding the three categories—requests, responses, and notifications—and their structural patterns lets you build a clean dispatcher that routes messages correctly.

## What to Learn

### The Three Message Types

LSP uses JSON-RPC 2.0 with exactly three message types:

```
1. Request      Client → Server    Expects response
2. Response     Server → Client    Answers a request
3. Notification Either direction   No response expected
```

### Request Structure

Requests have an `id` and expect a response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/completion",
  "params": {
    "textDocument": { "uri": "file:///home/user/doc.txt" },
    "position": { "line": 10, "character": 5 }
  }
}
```

**Key fields:**
- `jsonrpc`: Always `"2.0"`
- `id`: Number or string, unique per session, used to match response
- `method`: The operation requested
- `params`: Method-specific parameters (can be omitted if empty)

### Response Structure

Responses match a request by `id`:

**Success response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    { "label": "hello", "kind": 1 },
    { "label": "help", "kind": 1 }
  ]
}
```

**Error response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32601,
    "message": "Method not found"
  }
}
```

**Key rules:**
- Response MUST have same `id` as request
- Response has EITHER `result` OR `error`, never both
- `result` can be `null` for methods that succeed but return nothing

### Notification Structure

Notifications have no `id`:

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didOpen",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt",
      "languageId": "markdown",
      "version": 1,
      "text": "# Hello World\n\nThis is my document."
    }
  }
}
```

**Server-to-client notification:**
```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///home/user/doc.txt",
    "diagnostics": [
      {
        "range": { "start": { "line": 2, "character": 0 }, "end": { "line": 2, "character": 5 } },
        "message": "Unknown word",
        "severity": 3
      }
    ]
  }
}
```

### Common Methods You'll Implement

**Lifecycle (requests):**
```
initialize          First message from client
shutdown            Client requests graceful stop
```

**Lifecycle (notifications):**
```
initialized         Client confirms init complete
exit                Client tells server to terminate
```

**Document sync (notifications):**
```
textDocument/didOpen     Document opened
textDocument/didChange   Document modified
textDocument/didClose    Document closed
```

**Features (requests):**
```
textDocument/completion  Get completions at position
textDocument/codeAction  Get available actions
```

**Features (server → client notifications):**
```
textDocument/publishDiagnostics   Report problems
```

### Identifying Message Type in Code

```c
typedef enum {
    MSG_REQUEST,
    MSG_RESPONSE,
    MSG_NOTIFICATION,
    MSG_INVALID
} MessageType;

MessageType classify_message(const JsonValue *msg) {
    bool has_id = json_has_key(msg, "id");
    bool has_method = json_has_key(msg, "method");
    bool has_result = json_has_key(msg, "result");
    bool has_error = json_has_key(msg, "error");
    
    if (has_method && has_id) {
        return MSG_REQUEST;
    }
    if (has_method && !has_id) {
        return MSG_NOTIFICATION;
    }
    if ((has_result || has_error) && has_id) {
        return MSG_RESPONSE;
    }
    return MSG_INVALID;
}
```

### Dispatch Pattern

```c
void handle_message(Server *server, const JsonValue *msg) {
    MessageType type = classify_message(msg);
    
    switch (type) {
        case MSG_REQUEST: {
            const char *method = json_get_string(msg, "method");
            int64_t id = json_get_int(msg, "id");
            const JsonValue *params = json_get(msg, "params");
            
            dispatch_request(server, id, method, params);
            break;
        }
        case MSG_NOTIFICATION: {
            const char *method = json_get_string(msg, "method");
            const JsonValue *params = json_get(msg, "params");
            
            dispatch_notification(server, method, params);
            break;
        }
        case MSG_RESPONSE:
            // Servers rarely receive responses
            // (unless using server-initiated requests)
            break;
        case MSG_INVALID:
            // Log and ignore, or send error response
            break;
    }
}
```

### Error Codes

Standard JSON-RPC error codes:
```c
#define JSONRPC_PARSE_ERROR      -32700  // Invalid JSON
#define JSONRPC_INVALID_REQUEST  -32600  // Not valid JSON-RPC
#define JSONRPC_METHOD_NOT_FOUND -32601  // Unknown method
#define JSONRPC_INVALID_PARAMS   -32602  // Invalid parameters
#define JSONRPC_INTERNAL_ERROR   -32603  // Server error
```

LSP-specific error codes:
```c
#define LSP_SERVER_NOT_INITIALIZED -32002
#define LSP_UNKNOWN_ERROR_CODE     -32001
#define LSP_REQUEST_FAILED         -32803
#define LSP_SERVER_CANCELLED       -32802
#define LSP_CONTENT_MODIFIED       -32801
#define LSP_REQUEST_CANCELLED      -32800
```

### Sending Responses

```c
void send_response(Server *server, int64_t id, const JsonValue *result) {
    JsonValue *response = json_object();
    json_set_string(response, "jsonrpc", "2.0");
    json_set_int(response, "id", id);
    json_set(response, "result", result);
    
    send_message(server, response);
    json_free(response);
}

void send_error(Server *server, int64_t id, int code, const char *message) {
    JsonValue *response = json_object();
    json_set_string(response, "jsonrpc", "2.0");
    json_set_int(response, "id", id);
    
    JsonValue *error = json_object();
    json_set_int(error, "code", code);
    json_set_string(error, "message", message);
    json_set(response, "error", error);
    
    send_message(server, response);
    json_free(response);
}
```

## Where to Learn

1. **JSON-RPC 2.0 Specification:** https://www.jsonrpc.org/specification
   - Short, precise, covers all edge cases

2. **LSP Specification - Base Protocol:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#baseProtocol
   
3. **Real examples:** Capture traffic from an existing LSP server
   - `clangd --log=verbose`
   - VS Code's "Output" panel → select language server

## Practice Exercise

Create a message classifier:

```c
// test_messages.c
#include <assert.h>
#include <string.h>

// Your classify_message function here

void test_classification(void) {
    // Request
    const char *request = "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{}}";
    assert(classify_message(parse_json(request)) == MSG_REQUEST);
    
    // Notification
    const char *notif = "{\"jsonrpc\":\"2.0\",\"method\":\"initialized\"}";
    assert(classify_message(parse_json(notif)) == MSG_NOTIFICATION);
    
    // Response
    const char *resp = "{\"jsonrpc\":\"2.0\",\"id\":1,\"result\":null}";
    assert(classify_message(parse_json(resp)) == MSG_RESPONSE);
    
    // Error response
    const char *err = "{\"jsonrpc\":\"2.0\",\"id\":1,\"error\":{\"code\":-32601,\"message\":\"x\"}}";
    assert(classify_message(parse_json(err)) == MSG_RESPONSE);
    
    printf("All classification tests passed!\n");
}
```

## Connection to Project

Your dispatcher will be the central routing hub:

```
Incoming message
      │
      ▼
  classify_message()
      │
      ├── Request ──────▶ dispatch_request() ──▶ handler ──▶ send_response()
      │
      ├── Notification ─▶ dispatch_notification() ──▶ handler (no response)
      │
      └── Invalid ──────▶ log error, ignore or respond with error
```

Getting this routing right early means each feature handler can focus on its specific logic without worrying about protocol details.
