# Server Capabilities

## Why You Need This

Your requirements say: "Respond to `initialize` with supported capabilities." Capabilities tell the editor what features you support. Advertise accurately—too little and features won't activate, too much and you'll get requests you can't handle.

## What to Learn

### What Are Server Capabilities?

During `initialize`, you tell the client what your server can do:

```json
{
  "capabilities": {
    "textDocumentSync": { "openClose": true, "change": 1 },
    "completionProvider": { "triggerCharacters": [] },
    "codeActionProvider": { "codeActionKinds": ["quickfix"] }
  }
}
```

The client only sends requests for features you advertise.

### Your Server's Capabilities

Based on your requirements:

```c
JsonValue *build_server_capabilities(void) {
    JsonValue *caps = json_object();
    
    // Text synchronization
    JsonValue *text_sync = json_object();
    json_set_bool(text_sync, "openClose", true);
    json_set_int(text_sync, "change", 1);  // Full sync
    json_set(caps, "textDocumentSync", text_sync);
    
    // Completion
    JsonValue *completion = json_object();
    json_set(completion, "triggerCharacters", json_array());
    json_set(caps, "completionProvider", completion);
    
    // Code actions
    JsonValue *code_action = json_object();
    JsonValue *kinds = json_array();
    json_array_push(kinds, json_string("quickfix"));
    json_set(code_action, "codeActionKinds", kinds);
    json_set(caps, "codeActionProvider", code_action);
    
    // Execute command (for code action commands)
    JsonValue *exec_cmd = json_object();
    JsonValue *commands = json_array();
    json_array_push(commands, json_string("wordlib.addWord"));
    json_array_push(commands, json_string("wordlib.ignoreWord"));
    json_set(exec_cmd, "commands", commands);
    json_set(caps, "executeCommandProvider", exec_cmd);
    
    return caps;
}
```

### TextDocumentSync Options

```c
// Sync kind
typedef enum {
    SYNC_NONE = 0,        // No sync
    SYNC_FULL = 1,        // Full document on every change
    SYNC_INCREMENTAL = 2  // Only changes
} TextDocumentSyncKind;

// Full options object
JsonValue *text_sync_options(void) {
    JsonValue *opts = json_object();
    
    // Receive didOpen/didClose
    json_set_bool(opts, "openClose", true);
    
    // Sync kind
    json_set_int(opts, "change", SYNC_FULL);
    
    // Receive willSave? (usually not needed)
    json_set_bool(opts, "willSave", false);
    
    // Receive didSave? (with text?)
    JsonValue *save = json_object();
    json_set_bool(save, "includeText", false);
    json_set(opts, "save", save);
    
    return opts;
}
```

### Completion Provider Options

```c
JsonValue *completion_options(void) {
    JsonValue *opts = json_object();
    
    // Characters that trigger completion automatically
    JsonValue *triggers = json_array();
    // For word completion, usually empty (trigger on word boundary)
    json_set(opts, "triggerCharacters", triggers);
    
    // Do we support resolve requests?
    json_set_bool(opts, "resolveProvider", false);
    
    return opts;
}
```

### Code Action Provider Options

```c
JsonValue *code_action_options(void) {
    // Simple: just true
    // return json_bool(true);
    
    // Or specify kinds
    JsonValue *opts = json_object();
    JsonValue *kinds = json_array();
    json_array_push(kinds, json_string("quickfix"));
    // json_array_push(kinds, json_string("refactor"));  // If supported
    json_set(opts, "codeActionKinds", kinds);
    
    return opts;
}
```

### Full Initialize Response

```c
JsonValue *handle_initialize(Server *server, const JsonValue *params) {
    // Store client info if needed
    const JsonValue *client_caps = json_get(params, "capabilities");
    parse_client_capabilities(server, client_caps);
    
    // Store workspace root
    const char *root_uri = json_get_string(params, "rootUri");
    if (root_uri) {
        server->workspace_root = strdup(root_uri);
    }
    
    // Build response
    JsonValue *result = json_object();
    
    // Capabilities
    json_set(result, "capabilities", build_server_capabilities());
    
    // Server info
    JsonValue *server_info = json_object();
    json_set_string(server_info, "name", "wordlib-lsp");
    json_set_string(server_info, "version", "1.0.0");
    json_set(result, "serverInfo", server_info);
    
    server->state = SERVER_INITIALIZING;
    return result;
}
```

### Capability Contract

If you declare a capability, you MUST handle the corresponding requests:

| Capability | Must Handle |
|------------|-------------|
| `textDocumentSync` | didOpen, didChange, didClose |
| `completionProvider` | textDocument/completion |
| `codeActionProvider` | textDocument/codeAction |
| `executeCommandProvider` | workspace/executeCommand |

### Testing Capabilities

```c
void test_capabilities(void) {
    JsonValue *caps = build_server_capabilities();
    
    // Verify text sync
    JsonValue *sync = json_get(caps, "textDocumentSync");
    assert(json_get_bool(sync, "openClose") == true);
    
    // Verify completion
    assert(json_has_key(caps, "completionProvider"));
    
    // Verify code action
    assert(json_has_key(caps, "codeActionProvider"));
    
    json_free(caps);
    printf("test_capabilities: PASS\n");
}
```

## Where to Learn

1. **LSP Specification - ServerCapabilities:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#serverCapabilities

2. **Test with your editor:**
   - Check what capabilities trigger which features

## Practice Exercise

Verify your server's capabilities match its handlers:

```c
// Capability audit
void audit_capabilities(void) {
    // For each capability declared...
    // Verify corresponding handler exists
    
    // textDocumentSync
    assert(can_handle("textDocument/didOpen"));
    assert(can_handle("textDocument/didChange"));
    assert(can_handle("textDocument/didClose"));
    
    // completionProvider
    assert(can_handle("textDocument/completion"));
    
    // codeActionProvider
    assert(can_handle("textDocument/codeAction"));
    
    // executeCommandProvider
    assert(can_handle("workspace/executeCommand"));
    
    printf("Capability audit: PASS\n");
}
```

## Connection to Project

Capabilities are your server's contract with editors:

```
Initialize:
  Server: "I support completion, code actions, and text sync"
  
Runtime:
  Client: "textDocument/completion" → Server handles
  Client: "textDocument/hover" → Server rejects (not declared)
```

Accurate capabilities = correct behavior.
