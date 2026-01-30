# Error Handling Strategy

## Why You Need This

Your requirements say: "Gracefully handle malformed LSP messages" and "Continue operating after recoverable failures."

A production server can't crash on bad input. Users depend on you to keep working even when things go wrong.

## What to Learn

### Error Categories

**1. Protocol Errors:** Malformed JSON, invalid messages
- Log and respond with JSON-RPC error
- Don't crash

**2. Internal Errors:** Null pointers, out of memory
- Log, return safe default
- Don't corrupt state

**3. Resource Errors:** File not found, permission denied
- Log, continue with degraded functionality
- Don't block other operations

### JSON-RPC Error Responses

When a request fails:

```c
void send_error_response(Server *server, int64_t id, int code, const char *message) {
    JsonValue *response = json_object();
    json_set_string(response, "jsonrpc", "2.0");
    json_set_int(response, "id", id);
    
    JsonValue *error = json_object();
    json_set_int(error, "code", code);
    json_set_string(error, "message", message);
    json_set(response, "error", error);
    
    char *json = json_stringify(response);
    transport_write(server->transport, json);
    free(json);
    json_free(response);
}
```

Standard error codes:
```c
#define ERR_PARSE_ERROR       -32700
#define ERR_INVALID_REQUEST   -32600
#define ERR_METHOD_NOT_FOUND  -32601
#define ERR_INVALID_PARAMS    -32602
#define ERR_INTERNAL_ERROR    -32603
#define ERR_SERVER_NOT_INIT   -32002
```

### Handling Bad Input

```c
void handle_message(Server *server, const char *raw_json) {
    // Parse JSON
    JsonValue *msg = json_parse(raw_json);
    if (!msg) {
        log_error("JSON parse failed");
        // Can't send error response without knowing the ID
        // Just log and continue
        return;
    }
    
    // Validate structure
    if (!json_has_key(msg, "jsonrpc")) {
        log_error("Missing jsonrpc field");
        if (json_has_key(msg, "id")) {
            send_error_response(server, json_get_int(msg, "id"),
                              ERR_INVALID_REQUEST, "Missing jsonrpc field");
        }
        json_free(msg);
        return;
    }
    
    // Route message
    // ... 
}
```

### Defensive Handler Pattern

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // Validate params
    if (!params) {
        log_warn("completion: missing params");
        return json_array();  // Empty result is safe
    }
    
    const JsonValue *td = json_get(params, "textDocument");
    if (!td) {
        log_warn("completion: missing textDocument");
        return json_array();
    }
    
    const char *uri = json_get_string(td, "uri");
    if (!uri) {
        log_warn("completion: missing uri");
        return json_array();
    }
    
    // Get document (might not exist)
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) {
        log_debug("completion: document not found: %s", uri);
        return json_array();
    }
    
    // Now we can safely proceed
    // ...
}
```

### Error Recovery in Main Loop

```c
void server_main_loop(Server *server) {
    while (server->state != SERVER_STOPPED) {
        char *msg = transport_read(server->transport);
        
        if (!msg) {
            if (transport_eof(server->transport)) {
                log_info("Client disconnected");
                break;
            }
            log_error("Transport read error, continuing...");
            continue;  // Don't crash
        }
        
        // Process with error boundary
        bool ok = process_message_safe(server, msg);
        free(msg);
        
        if (!ok) {
            log_error("Message processing failed, continuing...");
            // Server keeps running
        }
    }
}

bool process_message_safe(Server *server, const char *msg) {
    // Catch internal errors
    JsonValue *json = json_parse(msg);
    if (!json) {
        log_error("Parse failed");
        return false;
    }
    
    JsonValue *response = dispatch_message(server, json);
    
    if (response) {
        char *response_str = json_stringify(response);
        transport_write(server->transport, response_str);
        free(response_str);
        json_free(response);
    }
    
    json_free(json);
    return true;
}
```

### NULL Checks Pattern

```c
// Always check before use
void publish_diagnostics(Server *server, const char *uri) {
    if (!server) return;
    if (!uri) return;
    if (!server->documents) return;
    if (!server->transport) return;
    
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    // Safe to proceed
}
```

### Result Types (Optional)

For more explicit error handling:

```c
typedef struct {
    bool success;
    char *error_message;
    JsonValue *result;
} Result;

Result handle_completion_safe(Server *server, const JsonValue *params) {
    Result r = { .success = true };
    
    Document *doc = get_document_from_params(params);
    if (!doc) {
        r.success = false;
        r.error_message = "Document not found";
        r.result = json_array();
        return r;
    }
    
    r.result = compute_completions(server, doc, params);
    return r;
}
```

## Where to Learn

1. **Defensive programming:** "Code Complete" by Steve McConnell
2. **Error handling patterns:** How mature C projects handle errors
3. **LSP specification:** Error response format

## Practice Exercise

Test error handling:

```c
void test_malformed_input(void) {
    Server *server = test_server_create();
    
    // Invalid JSON
    bool ok = process_message_safe(server, "not json at all");
    assert(ok == false);
    // Server should still be running
    assert(server->state != SERVER_STOPPED);
    
    // Missing fields
    ok = process_message_safe(server, "{\"method\":\"test\"}");
    // Should handle gracefully
    
    // Unknown method
    ok = process_message_safe(server, 
        "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"unknown\"}");
    // Should return error response, not crash
    
    test_server_destroy(server);
    printf("test_malformed_input: PASS\n");
}
```

## Connection to Project

Robust error handling means:

| Situation | Bad Server | Good Server |
|-----------|------------|-------------|
| Malformed JSON | Crash | Log, continue |
| Unknown method | Crash | Error response |
| Missing document | Crash | Empty result |
| OOM | Crash | Graceful degradation |

Your users need a server that stays up.
