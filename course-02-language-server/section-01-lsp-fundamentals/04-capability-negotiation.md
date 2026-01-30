# Capability Negotiation

## Why You Need This

Your requirements say: "Respond to `initialize` with supported capabilities" and "Configuration and capability negotiation."

Capabilities are how client and server agree on what features to use. Advertise too little and features won't work. Advertise too much and you'll get requests you can't handle.

## What to Learn

### The Negotiation Dance

```
Client: "Here's what I support" (client capabilities)
Server: "Here's what I support" (server capabilities)
Result: Both use the intersection
```

This happens once during `initialize`. After that, both sides know what to expect.

### Client Capabilities (What You Receive)

The client tells you what it can handle:

```json
{
  "capabilities": {
    "textDocument": {
      "synchronization": {
        "dynamicRegistration": true,
        "willSave": true,
        "didSave": true
      },
      "completion": {
        "completionItem": {
          "snippetSupport": true,
          "documentationFormat": ["markdown", "plaintext"]
        }
      },
      "publishDiagnostics": {
        "relatedInformation": true,
        "tagSupport": { "valueSet": [1, 2] }
      },
      "codeAction": {
        "codeActionLiteralSupport": {
          "codeActionKind": {
            "valueSet": ["quickfix", "refactor"]
          }
        }
      }
    },
    "workspace": {
      "configuration": true,
      "didChangeConfiguration": {
        "dynamicRegistration": true
      }
    }
  }
}
```

**What to check:**
- Does client support incremental sync? (affects how you send document changes)
- Does client support specific diagnostic features? (tags, related info)
- Does client support code action kinds you want to use?

### Server Capabilities (What You Advertise)

You tell the client what you provide:

```json
{
  "capabilities": {
    "textDocumentSync": {
      "openClose": true,
      "change": 1
    },
    "completionProvider": {
      "triggerCharacters": []
    },
    "codeActionProvider": {
      "codeActionKinds": ["quickfix"]
    }
  }
}
```

### Capabilities Your Server Should Declare

Based on your requirements:

```c
JsonValue *build_server_capabilities(Server *server) {
    JsonValue *caps = json_object();
    
    // Document synchronization
    // 1 = Full sync (receive entire document on change)
    // 2 = Incremental sync (receive only changes)
    JsonValue *text_sync = json_object();
    json_set_bool(text_sync, "openClose", true);  // didOpen/didClose
    json_set_int(text_sync, "change", 1);         // Full sync (simpler)
    json_set(caps, "textDocumentSync", text_sync);
    
    // Completion support
    JsonValue *completion = json_object();
    // Empty trigger chars = trigger on word boundaries
    json_set(completion, "triggerCharacters", json_array());
    json_set(caps, "completionProvider", completion);
    
    // Code actions (add word to dictionary)
    JsonValue *code_action = json_object();
    JsonValue *kinds = json_array();
    json_array_push_string(kinds, "quickfix");
    json_set(code_action, "codeActionKinds", kinds);
    json_set(caps, "codeActionProvider", code_action);
    
    // Note: Diagnostics are push-only (publishDiagnostics)
    // No capability needed - we just send them
    
    return caps;
}
```

### Text Document Sync Options

```c
typedef enum {
    SYNC_NONE = 0,        // No sync (useless for us)
    SYNC_FULL = 1,        // Send full text on every change
    SYNC_INCREMENTAL = 2  // Send only changes (more complex)
} TextDocumentSyncKind;
```

Your requirements say: "At minimum, the server shall support full document synchronization. Incremental sync support is recommended but optional."

Start with `SYNC_FULL`. It's simpler:

```c
// With full sync, didChange gives you complete text:
void handle_did_change(Server *server, const JsonValue *params) {
    const char *uri = get_document_uri(params);
    
    // Full sync: contentChanges[0].text is the entire document
    const JsonValue *changes = json_get(params, "contentChanges");
    const JsonValue *first_change = json_array_get(changes, 0);
    const char *new_text = json_get_string(first_change, "text");
    
    update_document(server, uri, new_text);
}
```

### Completion Provider Options

```c
JsonValue *completion_provider = json_object();

// Characters that trigger completion automatically
// Empty = only manual trigger (Ctrl+Space) or word boundary
JsonValue *triggers = json_array();
// json_array_push_string(triggers, ".");  // Trigger on dot
json_set(completion, "triggerCharacters", triggers);

// Whether server handles resolve (lazy item details)
// false = send complete items in completion response
// true = send minimal items, resolve on demand
json_set_bool(completion, "resolveProvider", false);
```

### Code Action Provider Options

```c
// Simple: just say you support code actions
json_set_bool(caps, "codeActionProvider", true);

// Or specify which kinds:
JsonValue *code_action = json_object();
JsonValue *kinds = json_array();
json_array_push_string(kinds, "quickfix");        // Our "add word" action
// json_array_push_string(kinds, "refactor");     // Not applicable
// json_array_push_string(kinds, "source");       // Not applicable
json_set(code_action, "codeActionKinds", kinds);
json_set(caps, "codeActionProvider", code_action);
```

### Storing Client Capabilities

Some client capabilities affect your behavior:

```c
typedef struct {
    bool supports_related_diagnostics;
    bool supports_diagnostic_tags;
    bool supports_markdown_completion;
    // ... etc
} ClientCapabilities;

void parse_client_capabilities(Server *server, const JsonValue *caps) {
    server->client_caps = (ClientCapabilities){0};
    
    // Check diagnostic support
    const JsonValue *diag = json_get(caps, "textDocument.publishDiagnostics");
    if (diag) {
        server->client_caps.supports_related_diagnostics = 
            json_get_bool(diag, "relatedInformation", false);
        
        const JsonValue *tags = json_get(diag, "tagSupport");
        server->client_caps.supports_diagnostic_tags = (tags != NULL);
    }
    
    // Check completion support
    const JsonValue *comp = json_get(caps, "textDocument.completion.completionItem");
    if (comp) {
        const JsonValue *doc_format = json_get(comp, "documentationFormat");
        if (doc_format) {
            // Check if markdown is in the array
            for (size_t i = 0; i < json_array_length(doc_format); i++) {
                if (strcmp(json_array_get_string(doc_format, i), "markdown") == 0) {
                    server->client_caps.supports_markdown_completion = true;
                    break;
                }
            }
        }
    }
}
```

### Dynamic Registration (Advanced)

Some capabilities can be registered after initialization:

```json
// Server → Client: Register completion for specific file types
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "client/registerCapability",
  "params": {
    "registrations": [{
      "id": "completion-md",
      "method": "textDocument/completion",
      "registerOptions": {
        "documentSelector": [{ "language": "markdown" }]
      }
    }]
  }
}
```

**Skip this for now.** Static registration in initialize is sufficient for your requirements.

## Where to Learn

1. **LSP Specification - Capabilities:**
   - Server capabilities: https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#serverCapabilities
   - Client capabilities: Search for "ClientCapabilities"

2. **Examples:**
   - Look at `clangd`'s capabilities response
   - Check what Neovim's built-in LSP client sends

## Practice Exercise

1. Write a function that builds your server capabilities
2. Log the client capabilities you receive during initialize
3. Test with Neovim:
   ```lua
   vim.lsp.start({
     name = 'test',
     cmd = { './your-server' },
     on_init = function(client)
       print(vim.inspect(client.server_capabilities))
     end
   })
   ```
4. Verify the editor shows expected features (completion, code actions)

## Connection to Project

Your capability declaration is a contract:

| If you declare... | You must handle... |
|-------------------|-------------------|
| `textDocumentSync` | didOpen, didChange, didClose |
| `completionProvider` | textDocument/completion |
| `codeActionProvider` | textDocument/codeAction |

Don't advertise what you can't deliver. Start minimal, add features as you implement them.
