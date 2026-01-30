# JSON Parsing in C

## Why You Need This

LSP messages are JSON. You need to parse incoming JSON into usable structures and serialize outgoing data back to JSON. Choosing the right approach saves hours of debugging.

## What to Learn

### Your Options

**1. Use an existing library (recommended):**
- cJSON (small, easy, widely used)
- Jansson (more features, larger)
- yyjson (fastest, modern)
- Write your own (educational, time-consuming)

**2. Write a minimal parser (for learning):**
- Good exercise, but not for production
- LSP messages are simple enough that a basic parser works

### Recommended: cJSON

Small, single-file, MIT licensed, well-tested.

**Installation:**
```bash
# Download to your project
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.c
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.h
```

**Add to Makefile:**
```makefile
SRCS += cJSON.c
```

### cJSON Basics

**Parsing:**
```c
#include "cJSON.h"

const char *json_text = "{\"id\":1,\"method\":\"test\"}";
cJSON *json = cJSON_Parse(json_text);

if (json == NULL) {
    const char *error = cJSON_GetErrorPtr();
    fprintf(stderr, "Parse error at: %s\n", error);
}
```

**Accessing values:**
```c
// Get object member
cJSON *id = cJSON_GetObjectItemCaseSensitive(json, "id");
cJSON *method = cJSON_GetObjectItem(json, "method");

// Check type and get value
if (cJSON_IsNumber(id)) {
    int64_t id_value = (int64_t)id->valuedouble;
}

if (cJSON_IsString(method)) {
    const char *method_name = method->valuestring;
}
```

**Creating JSON:**
```c
cJSON *response = cJSON_CreateObject();
cJSON_AddStringToObject(response, "jsonrpc", "2.0");
cJSON_AddNumberToObject(response, "id", 1);

cJSON *result = cJSON_CreateObject();
cJSON_AddStringToObject(result, "name", "wordlib-lsp");
cJSON_AddItemToObject(response, "result", result);

// Serialize to string
char *json_str = cJSON_PrintUnformatted(response);
// Use json_str...
free(json_str);

cJSON_Delete(response);  // Free everything
```

**Arrays:**
```c
// Create array
cJSON *items = cJSON_CreateArray();
cJSON *item1 = cJSON_CreateObject();
cJSON_AddStringToObject(item1, "label", "hello");
cJSON_AddItemToArray(items, item1);

// Iterate array
cJSON *item;
cJSON_ArrayForEach(item, items) {
    cJSON *label = cJSON_GetObjectItem(item, "label");
    printf("%s\n", label->valuestring);
}
```

### Wrapper Functions

Create a consistent API for your server:

```c
// json.h - Your wrapper API
typedef cJSON JsonValue;

// Parsing
JsonValue *json_parse(const char *text);
void json_free(JsonValue *json);

// Type checking
bool json_is_object(const JsonValue *v);
bool json_is_array(const JsonValue *v);
bool json_is_string(const JsonValue *v);
bool json_is_number(const JsonValue *v);
bool json_is_bool(const JsonValue *v);
bool json_is_null(const JsonValue *v);

// Accessors
const char *json_get_string(const JsonValue *obj, const char *key);
int64_t json_get_int(const JsonValue *obj, const char *key);
double json_get_number(const JsonValue *obj, const char *key);
bool json_get_bool(const JsonValue *obj, const char *key);
JsonValue *json_get(const JsonValue *obj, const char *key);
bool json_has_key(const JsonValue *obj, const char *key);

// Array access
size_t json_array_length(const JsonValue *arr);
JsonValue *json_array_get(const JsonValue *arr, size_t index);

// Creation
JsonValue *json_object(void);
JsonValue *json_array(void);
JsonValue *json_string(const char *s);
JsonValue *json_int(int64_t n);
JsonValue *json_bool(bool b);
JsonValue *json_null(void);

// Mutation
void json_set(JsonValue *obj, const char *key, JsonValue *value);
void json_set_string(JsonValue *obj, const char *key, const char *value);
void json_set_int(JsonValue *obj, const char *key, int64_t value);
void json_set_bool(JsonValue *obj, const char *key, bool value);
void json_array_push(JsonValue *arr, JsonValue *item);

// Serialization
char *json_stringify(const JsonValue *json);  // Caller must free
```

**Implementation:**
```c
// json.c
#include "json.h"
#include "cJSON.h"

JsonValue *json_parse(const char *text) {
    return cJSON_Parse(text);
}

void json_free(JsonValue *json) {
    cJSON_Delete(json);
}

bool json_is_object(const JsonValue *v) {
    return cJSON_IsObject(v);
}

const char *json_get_string(const JsonValue *obj, const char *key) {
    cJSON *item = cJSON_GetObjectItemCaseSensitive(obj, key);
    if (item && cJSON_IsString(item)) {
        return item->valuestring;
    }
    return NULL;
}

int64_t json_get_int(const JsonValue *obj, const char *key) {
    cJSON *item = cJSON_GetObjectItemCaseSensitive(obj, key);
    if (item && cJSON_IsNumber(item)) {
        return (int64_t)item->valuedouble;
    }
    return 0;
}

JsonValue *json_get(const JsonValue *obj, const char *key) {
    return cJSON_GetObjectItemCaseSensitive(obj, key);
}

bool json_has_key(const JsonValue *obj, const char *key) {
    return cJSON_GetObjectItemCaseSensitive(obj, key) != NULL;
}

JsonValue *json_object(void) {
    return cJSON_CreateObject();
}

void json_set_string(JsonValue *obj, const char *key, const char *value) {
    cJSON_AddStringToObject(obj, key, value);
}

void json_set_int(JsonValue *obj, const char *key, int64_t value) {
    cJSON_AddNumberToObject(obj, key, (double)value);
}

char *json_stringify(const JsonValue *json) {
    return cJSON_PrintUnformatted(json);
}
```

### Memory Management

**cJSON owns the memory:**
```c
// Strings returned by cJSON are owned by the JSON object
const char *method = json_get_string(msg, "method");
// method is valid until msg is freed

// If you need to keep it longer, copy it
char *method_copy = strdup(method);
```

**When building responses:**
```c
// Items added to objects are owned by that object
cJSON *response = cJSON_CreateObject();
cJSON *result = cJSON_CreateObject();  // Owned by...
cJSON_AddItemToObject(response, "result", result);  // ...response

cJSON_Delete(response);  // Frees response AND result
```

### Nested Path Access

LSP often has nested structures. Helper function:

```c
// Get nested value: json_path(obj, "textDocument.uri")
JsonValue *json_path(const JsonValue *obj, const char *path) {
    char *path_copy = strdup(path);
    char *token = strtok(path_copy, ".");
    
    const JsonValue *current = obj;
    while (token && current) {
        current = json_get(current, token);
        token = strtok(NULL, ".");
    }
    
    free(path_copy);
    return (JsonValue *)current;
}

// Usage
const char *uri = json_get_string(json_path(params, "textDocument"), "uri");
// Or with direct nesting
const JsonValue *td = json_get(params, "textDocument");
const char *uri = td ? json_get_string(td, "uri") : NULL;
```

### Error Handling

```c
JsonValue *parse_and_validate(const char *text) {
    JsonValue *json = json_parse(text);
    
    if (!json) {
        log_error("JSON parse failed");
        return NULL;
    }
    
    // Validate required fields
    if (!json_has_key(json, "jsonrpc")) {
        log_error("Missing jsonrpc field");
        json_free(json);
        return NULL;
    }
    
    const char *version = json_get_string(json, "jsonrpc");
    if (!version || strcmp(version, "2.0") != 0) {
        log_error("Invalid jsonrpc version");
        json_free(json);
        return NULL;
    }
    
    return json;
}
```

## Where to Learn

1. **cJSON GitHub:** https://github.com/DaveGamble/cJSON
   - README has extensive examples

2. **JSON specification:** https://www.json.org/
   - Know what you're parsing

3. **Alternatives:**
   - yyjson: https://github.com/ibireme/yyjson (fastest)
   - Jansson: https://github.com/akheron/jansson (more features)

## Practice Exercise

Parse this LSP initialize request and extract all fields:

```c
const char *json_text = "{"
    "\"jsonrpc\":\"2.0\","
    "\"id\":1,"
    "\"method\":\"initialize\","
    "\"params\":{"
        "\"processId\":12345,"
        "\"rootUri\":\"file:///home/user/project\","
        "\"capabilities\":{"
            "\"textDocument\":{"
                "\"completion\":{"
                    "\"completionItem\":{\"snippetSupport\":true}"
                "}"
            "}"
        "}"
    "}"
"}";

void test_parse_initialize(void) {
    JsonValue *json = json_parse(json_text);
    
    // Extract and print
    printf("jsonrpc: %s\n", json_get_string(json, "jsonrpc"));
    printf("id: %lld\n", json_get_int(json, "id"));
    printf("method: %s\n", json_get_string(json, "method"));
    
    JsonValue *params = json_get(json, "params");
    printf("processId: %lld\n", json_get_int(params, "processId"));
    printf("rootUri: %s\n", json_get_string(params, "rootUri"));
    
    // Check nested capability
    JsonValue *caps = json_get(params, "capabilities");
    JsonValue *td = json_get(caps, "textDocument");
    JsonValue *comp = json_get(td, "completion");
    JsonValue *ci = json_get(comp, "completionItem");
    bool snippets = json_get_bool_default(ci, "snippetSupport", false);
    printf("snippetSupport: %s\n", snippets ? "true" : "false");
    
    json_free(json);
}
```

## Connection to Project

JSON is the wire format. Your transport layer delivers raw JSON strings; your JSON layer turns them into structures you can work with:

```
Raw bytes  →  Transport  →  JSON string  →  JSON parser  →  Structured data
                                                                   │
                                                                   ▼
                                                            Your handlers
                                                                   │
                                                                   ▼
Raw bytes  ←  Transport  ←  JSON string  ←  JSON builder  ←  Response data
```

Pick cJSON, wrap it nicely, move on. JSON parsing shouldn't be where you spend your time.
