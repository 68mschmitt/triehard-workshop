# Server Lifecycle

## Why You Need This

Your requirements specify: "Respond to `initialize` with supported capabilities" and "Respond to `shutdown` / cleanly release engine resources / persist dictionary state before exit."

The lifecycle is the handshake that establishes the session. Get it wrong and editors won't talk to you.

## What to Learn

### Lifecycle State Machine

```
                   ┌─────────────────────┐
                   │    Not Started      │
                   └──────────┬──────────┘
                              │
                     initialize request
                              │
                              ▼
                   ┌─────────────────────┐
                   │   Initializing      │
                   └──────────┬──────────┘
                              │
                   initialize response + 
                   initialized notification
                              │
                              ▼
                   ┌─────────────────────┐
        ┌─────────│     Running         │◀────────┐
        │         └──────────┬──────────┘         │
        │                    │                    │
   document ops         shutdown request     normal ops
   completion               │
   code actions             ▼
        │         ┌─────────────────────┐
        └────────▶│    Shutting Down    │
                  └──────────┬──────────┘
                             │
                        exit notification
                             │
                             ▼
                  ┌─────────────────────┐
                  │      Stopped        │
                  └─────────────────────┘
```

### Initialize Request

Client sends first:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "processId": 12345,
    "rootUri": "file:///home/user/project",
    "capabilities": {
      "textDocument": {
        "completion": { "completionItem": { "snippetSupport": true } },
        "publishDiagnostics": { "relatedInformation": true }
      },
      "workspace": {
        "configuration": true
      }
    }
  }
}
```

**Important fields:**
- `processId`: Editor's process ID (for zombie detection)
- `rootUri`: Workspace root (for workspace dictionaries)
- `capabilities`: What the *client* supports

### Initialize Response

Your server responds with its capabilities:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "capabilities": {
      "textDocumentSync": 1,
      "completionProvider": { "triggerCharacters": [] },
      "codeActionProvider": true,
      "diagnosticProvider": {
        "interFileDependencies": false,
        "workspaceDiagnostics": false
      }
    },
    "serverInfo": {
      "name": "wordlib-lsp",
      "version": "1.0.0"
    }
  }
}
```

### Initialize Handler

```c
typedef enum {
    SERVER_UNINITIALIZED,
    SERVER_INITIALIZING,
    SERVER_RUNNING,
    SERVER_SHUTTING_DOWN,
    SERVER_STOPPED
} ServerState;

typedef struct {
    ServerState state;
    char *workspace_root;
    WordLib *engine;
    // ... other fields
} Server;

JsonValue *handle_initialize(Server *server, const JsonValue *params) {
    if (server->state != SERVER_UNINITIALIZED) {
        // Already initialized - protocol error
        return NULL;  // Will trigger error response
    }
    
    server->state = SERVER_INITIALIZING;
    
    // Extract workspace root
    const char *root_uri = json_get_string(
        json_get(params, "rootUri"), NULL);
    if (root_uri) {
        server->workspace_root = uri_to_path(root_uri);
    }
    
    // Initialize engine with workspace context
    server->engine = wordlib_create();
    if (server->workspace_root) {
        char dict_path[PATH_MAX];
        snprintf(dict_path, sizeof(dict_path), 
                 "%s/.wordlib/dictionary.txt", server->workspace_root);
        wordlib_load(server->engine, dict_path);
    }
    
    // Build response
    JsonValue *result = json_object();
    JsonValue *capabilities = json_object();
    
    // Text sync: 1 = Full, 2 = Incremental
    json_set_int(capabilities, "textDocumentSync", 1);
    
    // Completion support
    JsonValue *completion = json_object();
    json_set(completion, "triggerCharacters", json_array());
    json_set(capabilities, "completionProvider", completion);
    
    // Code action support
    json_set_bool(capabilities, "codeActionProvider", true);
    
    json_set(result, "capabilities", capabilities);
    
    // Optional server info
    JsonValue *server_info = json_object();
    json_set_string(server_info, "name", "wordlib-lsp");
    json_set_string(server_info, "version", "1.0.0");
    json_set(result, "serverInfo", server_info);
    
    return result;
}
```

### Initialized Notification

After receiving initialize response, client sends:

```json
{
  "jsonrpc": "2.0",
  "method": "initialized",
  "params": {}
}
```

Your handler:

```c
void handle_initialized(Server *server, const JsonValue *params) {
    (void)params;  // No params needed
    
    if (server->state == SERVER_INITIALIZING) {
        server->state = SERVER_RUNNING;
        log_info("Server initialized and ready");
    }
}
```

### Shutdown Request

Client requests graceful shutdown:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "shutdown"
}
```

Your handler:

```c
JsonValue *handle_shutdown(Server *server, const JsonValue *params) {
    (void)params;
    
    server->state = SERVER_SHUTTING_DOWN;
    
    // Persist dictionary before shutdown
    if (server->workspace_root) {
        char dict_path[PATH_MAX];
        snprintf(dict_path, sizeof(dict_path),
                 "%s/.wordlib/dictionary.txt", server->workspace_root);
        wordlib_save(server->engine, dict_path);
    }
    
    // Response is null on success
    return json_null();
}
```

### Exit Notification

Client tells server to terminate:

```json
{
  "jsonrpc": "2.0",
  "method": "exit"
}
```

Your handler:

```c
void handle_exit(Server *server, const JsonValue *params) {
    (void)params;
    
    // Clean up resources
    wordlib_destroy(server->engine);
    free(server->workspace_root);
    
    // Exit with appropriate code
    // 0 if shutdown was received, 1 otherwise
    int exit_code = (server->state == SERVER_SHUTTING_DOWN) ? 0 : 1;
    exit(exit_code);
}
```

### Guarding Against Invalid States

```c
JsonValue *dispatch_request(Server *server, int64_t id, 
                            const char *method, const JsonValue *params) {
    // Only initialize allowed before initialized
    if (server->state == SERVER_UNINITIALIZED) {
        if (strcmp(method, "initialize") != 0) {
            return make_error(LSP_SERVER_NOT_INITIALIZED,
                            "Server not initialized");
        }
    }
    
    // Only shutdown/exit allowed after shutdown
    if (server->state == SERVER_SHUTTING_DOWN) {
        if (strcmp(method, "exit") != 0) {
            return make_error(JSONRPC_INVALID_REQUEST,
                            "Server is shutting down");
        }
    }
    
    // Route to handler
    if (strcmp(method, "initialize") == 0) {
        return handle_initialize(server, params);
    } else if (strcmp(method, "shutdown") == 0) {
        return handle_shutdown(server, params);
    }
    // ... other handlers
    
    return make_error(JSONRPC_METHOD_NOT_FOUND, "Unknown method");
}
```

### Zombie Detection

If the editor crashes, your server becomes a zombie. Detect this:

```c
void check_parent_process(Server *server) {
    if (server->parent_pid > 0) {
        // Check if parent still exists
        if (kill(server->parent_pid, 0) == -1 && errno == ESRCH) {
            log_warn("Parent process %d died, exiting", server->parent_pid);
            exit(1);
        }
    }
}
```

Call this periodically or when idle.

## Where to Learn

1. **LSP Specification - Lifecycle:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#lifeCycleMessages

2. **Reference:** See how `clangd` or `rust-analyzer` handle lifecycle

3. **Testing:** Use a simple client script to test lifecycle:
   ```bash
   echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | ./your-server
   ```

## Practice Exercise

Write a minimal lifecycle handler:

1. Accept `initialize`, respond with empty capabilities
2. Track state transitions
3. Handle `shutdown` and `exit`
4. Test with manual JSON input:

```bash
# test_lifecycle.sh
(
  echo 'Content-Length: 52'
  echo ''
  echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
  sleep 1
  echo 'Content-Length: 38'
  echo ''
  echo '{"jsonrpc":"2.0","method":"initialized"}'
  sleep 1
  echo 'Content-Length: 44'
  echo ''
  echo '{"jsonrpc":"2.0","id":2,"method":"shutdown"}'
  sleep 1
  echo 'Content-Length: 33'
  echo ''
  echo '{"jsonrpc":"2.0","method":"exit"}'
) | ./your-server 2>&1
```

Expected: Initialize response, shutdown response (null result), clean exit.

## Connection to Project

The lifecycle establishes your server's context:

1. **initialize**: Get workspace root → load workspace dictionary
2. **Running**: Handle all document/feature requests
3. **shutdown**: Persist dictionary changes
4. **exit**: Clean up memory

Without proper lifecycle handling, your server can't integrate with any editor. This is table stakes.
