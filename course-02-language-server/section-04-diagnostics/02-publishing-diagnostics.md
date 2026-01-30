# Publishing Diagnostics

## Why You Need This

Your requirements say: "Publish diagnostics via `textDocument/publishDiagnostics`." This notification is how you send diagnostic information to the editor. Unlike requests, you initiate this—the editor just receives it.

## What to Learn

### The Notification Structure

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///home/user/doc.txt",
    "version": 5,
    "diagnostics": [
      { /* diagnostic 1 */ },
      { /* diagnostic 2 */ }
    ]
  }
}
```

**Fields:**
- `uri`: The document these diagnostics apply to
- `version` (optional): Document version when diagnostics were computed
- `diagnostics`: Array of diagnostic objects (can be empty)

### When to Publish

**Publish on:**
1. Document open (didOpen)
2. Document change (didChange)
3. Dictionary update (word added/removed)

```c
void handle_did_open(Server *server, const JsonValue *params) {
    // ... store document ...
    
    // Check and publish
    publish_diagnostics(server, uri);
}

void handle_did_change(Server *server, const JsonValue *params) {
    // ... update document ...
    
    // Re-check and publish
    publish_diagnostics(server, uri);
}

void handle_add_word(Server *server, const char *word) {
    wordlib_add_word(server->engine, word);
    
    // Re-check all open documents
    document_store_foreach(server->documents, 
                          recheck_document, server);
}
```

### Publishing Implementation

```c
void publish_diagnostics(Server *server, const char *uri) {
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    // Compute diagnostics
    JsonValue *diagnostics = compute_diagnostics(server, doc);
    
    // Build notification
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set_int(params, "version", doc->version);
    json_set(params, "diagnostics", diagnostics);
    
    JsonValue *notification = json_object();
    json_set_string(notification, "jsonrpc", "2.0");
    json_set_string(notification, "method", "textDocument/publishDiagnostics");
    json_set(notification, "params", params);
    
    // Send
    char *json = json_stringify(notification);
    transport_write(server->transport, json);
    free(json);
    
    json_free(notification);
}
```

### Computing Diagnostics

```c
JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    // Tokenize and check
    Tokenizer *tok = tokenizer_create(doc->text, doc->text_len);
    Token token;
    
    while (tokenizer_next(tok, &token)) {
        // Extract word
        char *word = strndup(doc->text + token.start, 
                            token.end - token.start);
        
        // Check if known
        if (!wordlib_contains(server->engine, word)) {
            // Convert byte span to LSP range
            Range range = byte_span_to_range(doc->text, 
                                            token.start, token.end);
            
            // Create and add diagnostic
            JsonValue *diag = create_diagnostic(range, word);
            json_array_push(diagnostics, diag);
        }
        
        free(word);
    }
    
    tokenizer_destroy(tok);
    return diagnostics;
}
```

### Clearing Diagnostics

To clear diagnostics (e.g., on document close), publish an empty array:

```c
void clear_diagnostics(Server *server, const char *uri) {
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set(params, "diagnostics", json_array());  // Empty array
    
    JsonValue *notification = json_object();
    json_set_string(notification, "jsonrpc", "2.0");
    json_set_string(notification, "method", "textDocument/publishDiagnostics");
    json_set(notification, "params", params);
    
    char *json = json_stringify(notification);
    transport_write(server->transport, json);
    free(json);
    
    json_free(notification);
}

void handle_did_close(Server *server, const JsonValue *params) {
    const char *uri = /* extract from params */;
    
    clear_diagnostics(server, uri);
    document_store_close(server->documents, uri);
}
```

### Version Tracking

Including the version helps editors discard stale diagnostics:

```c
// Good: Include version
json_set_int(params, "version", doc->version);

// If diagnostics arrive for version 5 but editor is at version 7,
// editor knows these are stale
```

### Batching Considerations

For performance, you might want to debounce diagnostic updates:

```c
typedef struct {
    char *uri;
    int64_t scheduled_version;
    struct timespec scheduled_time;
} PendingDiagnostic;

// Simple debounce: delay 50ms after last change
void schedule_diagnostics(Server *server, const char *uri, int64_t version) {
    // Cancel any pending for this URI
    // Schedule new check for 50ms from now
    // When timer fires, check if version still matches
}
```

For a simple implementation, immediate publishing is fine. Only optimize if you see performance issues.

### Full Flow Example

```
User opens file with "The quikc brown fox"
     │
     ▼
handle_did_open()
     │
     ├── document_store_open()
     │
     └── publish_diagnostics()
              │
              ├── tokenize: ["The", "quikc", "brown", "fox"]
              │
              ├── check each: quikc not in dictionary
              │
              └── send notification:
                  {
                    "method": "textDocument/publishDiagnostics",
                    "params": {
                      "uri": "file:///test.txt",
                      "diagnostics": [{
                        "range": {"start":{"line":0,"character":4},"end":{"line":0,"character":9}},
                        "message": "Unknown word: 'quikc'"
                      }]
                    }
                  }
     │
     ▼
Editor shows squiggle under "quikc"
```

## Where to Learn

1. **LSP Specification - publishDiagnostics:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_publishDiagnostics

2. **Test with editors:**
   - Watch what diagnostics your server sends
   - Verify they appear correctly

## Practice Exercise

Implement end-to-end diagnostic publishing:

```c
void test_publish_flow(void) {
    Server *server = create_test_server();
    
    // Simulate didOpen
    handle_did_open(server, json_parse(
        "{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\","
        "\"languageId\":\"plaintext\","
        "\"version\":1,"
        "\"text\":\"Hello wrold\"}}"
    ));
    
    // Capture what was sent
    char *sent = capture_last_message(server);
    
    // Verify it's a publishDiagnostics
    JsonValue *msg = json_parse(sent);
    assert(strcmp(json_get_string(msg, "method"), 
                  "textDocument/publishDiagnostics") == 0);
    
    // Verify diagnostic for "wrold"
    JsonValue *params = json_get(msg, "params");
    JsonValue *diags = json_get(params, "diagnostics");
    assert(json_array_length(diags) == 1);
    
    JsonValue *diag = json_array_get(diags, 0);
    assert(strstr(json_get_string(diag, "message"), "wrold") != NULL);
    
    free(sent);
    json_free(msg);
    destroy_test_server(server);
    printf("test_publish_flow: PASS\n");
}
```

## Connection to Project

Publishing diagnostics is the core of your spell checker's editor integration:

1. User opens/edits document
2. You tokenize and check words
3. You publish unknown words as diagnostics
4. Editor displays them
5. User sees what to fix

This is the visible output of all your work!
