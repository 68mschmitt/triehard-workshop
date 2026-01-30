# JSON-RPC 2.0 Basics

## Why You Need This

Your requirements specify: "JSON-RPC 2.0 shall be used for all requests and notifications." LSP builds on JSON-RPC, so understanding its rules ensures your messages are well-formed and your error handling is correct.

## What to Learn

### JSON-RPC in One Sentence

JSON-RPC is a stateless, transport-agnostic protocol for calling remote procedures using JSON.

### Message Structure

Every JSON-RPC message has:
- `jsonrpc`: Always `"2.0"`
- Plus either request fields, response fields, or notification fields

### Request Object

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/completion",
  "params": { ... }
}
```

**Rules:**
- `id` MUST be string, number, or null (don't use null)
- `method` MUST be a string
- `params` MAY be omitted, but if present must be object or array

### Response Object (Success)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { ... }
}
```

**Rules:**
- `id` MUST match the request's id
- `result` is REQUIRED on success (can be null)
- MUST NOT have `error` field

### Response Object (Error)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid Request",
    "data": { ... }
  }
}
```

**Rules:**
- `id` MUST match request, or be null if request id couldn't be determined
- `code` MUST be integer
- `message` MUST be string
- `data` is optional

### Notification Object

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didOpen",
  "params": { ... }
}
```

**Rules:**
- NO `id` field (that's what makes it a notification)
- Server MUST NOT respond to notifications

### Standard Error Codes

```c
// JSON-RPC defined errors
#define JSONRPC_PARSE_ERROR      -32700  // Invalid JSON received
#define JSONRPC_INVALID_REQUEST  -32600  // JSON not a valid Request
#define JSONRPC_METHOD_NOT_FOUND -32601  // Method does not exist
#define JSONRPC_INVALID_PARAMS   -32602  // Invalid method parameters
#define JSONRPC_INTERNAL_ERROR   -32603  // Internal JSON-RPC error

// Reserved range for implementation
// -32000 to -32099: Server errors (LSP uses some of these)
```

### LSP-Specific Error Codes

```c
// LSP extends the error codes
#define LSP_SERVER_NOT_INITIALIZED  -32002
#define LSP_UNKNOWN_ERROR_CODE      -32001
#define LSP_REQUEST_FAILED          -32803
#define LSP_SERVER_CANCELLED        -32802
#define LSP_CONTENT_MODIFIED        -32801
#define LSP_REQUEST_CANCELLED       -32800
```

### Building Messages in C

```c
// Create a response
JsonValue *make_response(int64_t id, JsonValue *result) {
    JsonValue *resp = json_object();
    json_set_string(resp, "jsonrpc", "2.0");
    json_set_int(resp, "id", id);
    json_set(resp, "result", result);
    return resp;
}

// Create an error response
JsonValue *make_error_response(int64_t id, int code, const char *message) {
    JsonValue *resp = json_object();
    json_set_string(resp, "jsonrpc", "2.0");
    json_set_int(resp, "id", id);
    
    JsonValue *error = json_object();
    json_set_int(error, "code", code);
    json_set_string(error, "message", message);
    json_set(resp, "error", error);
    
    return resp;
}

// Create a notification (server → client)
JsonValue *make_notification(const char *method, JsonValue *params) {
    JsonValue *notif = json_object();
    json_set_string(notif, "jsonrpc", "2.0");
    json_set_string(notif, "method", method);
    if (params) {
        json_set(notif, "params", params);
    }
    return notif;
}
```

### Parsing Incoming Messages

```c
typedef struct {
    bool is_request;      // Has id and method
    bool is_notification; // Has method, no id
    bool is_response;     // Has id and (result or error)
    
    int64_t id;           // Valid if is_request or is_response
    const char *method;   // Valid if is_request or is_notification
    JsonValue *params;    // Request/notification params
    JsonValue *result;    // Response result
    JsonValue *error;     // Response error
} ParsedMessage;

ParsedMessage parse_message(const JsonValue *msg) {
    ParsedMessage pm = {0};
    
    bool has_id = json_has_key(msg, "id");
    bool has_method = json_has_key(msg, "method");
    bool has_result = json_has_key(msg, "result");
    bool has_error = json_has_key(msg, "error");
    
    if (has_method && has_id) {
        pm.is_request = true;
        pm.id = json_get_int(msg, "id");
        pm.method = json_get_string(msg, "method");
        pm.params = json_get(msg, "params");
    } else if (has_method && !has_id) {
        pm.is_notification = true;
        pm.method = json_get_string(msg, "method");
        pm.params = json_get(msg, "params");
    } else if (has_id && (has_result || has_error)) {
        pm.is_response = true;
        pm.id = json_get_int(msg, "id");
        pm.result = json_get(msg, "result");
        pm.error = json_get(msg, "error");
    }
    
    return pm;
}
```

### Batch Requests (Don't Worry About It)

JSON-RPC supports batch requests (array of messages). LSP doesn't use them—you can ignore this feature.

### ID Management

For requests you send to the client (rare for LSP servers):

```c
typedef struct {
    int64_t next_id;
} IdGenerator;

int64_t next_request_id(IdGenerator *gen) {
    return gen->next_id++;
}
```

For responses, just echo back the request's id.

## Where to Learn

1. **JSON-RPC 2.0 Specification:** https://www.jsonrpc.org/specification
   - It's short—read the whole thing

2. **Wikipedia:** "JSON-RPC" has a good summary

3. **LSP Specification - Base Protocol:**
   - Shows how LSP uses JSON-RPC

## Practice Exercise

Write unit tests for your message parsing:

```c
void test_parse_request(void) {
    const char *json = "{\"jsonrpc\":\"2.0\",\"id\":42,\"method\":\"test\",\"params\":{}}";
    JsonValue *msg = json_parse(json);
    
    ParsedMessage pm = parse_message(msg);
    
    assert(pm.is_request);
    assert(!pm.is_notification);
    assert(!pm.is_response);
    assert(pm.id == 42);
    assert(strcmp(pm.method, "test") == 0);
    
    json_free(msg);
    printf("test_parse_request: PASS\n");
}

void test_parse_notification(void) {
    const char *json = "{\"jsonrpc\":\"2.0\",\"method\":\"exit\"}";
    JsonValue *msg = json_parse(json);
    
    ParsedMessage pm = parse_message(msg);
    
    assert(!pm.is_request);
    assert(pm.is_notification);
    assert(!pm.is_response);
    assert(strcmp(pm.method, "exit") == 0);
    
    json_free(msg);
    printf("test_parse_notification: PASS\n");
}

void test_parse_response(void) {
    const char *json = "{\"jsonrpc\":\"2.0\",\"id\":1,\"result\":null}";
    JsonValue *msg = json_parse(json);
    
    ParsedMessage pm = parse_message(msg);
    
    assert(!pm.is_request);
    assert(!pm.is_notification);
    assert(pm.is_response);
    assert(pm.id == 1);
    
    json_free(msg);
    printf("test_parse_response: PASS\n");
}
```

## Connection to Project

JSON-RPC is the foundation. Every message your server sends or receives follows these rules. Getting this layer right means:

- Valid responses for every request
- Proper error codes when things go wrong
- Clean separation between protocol and feature logic

Your dispatcher will use the message classification to route to appropriate handlers.
