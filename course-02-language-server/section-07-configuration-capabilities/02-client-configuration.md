# Client Configuration

## Why You Need This

Your requirements say: "Support configuration via `workspace/didChangeConfiguration`" and list configurable options like severity, dictionary paths, and enabled file types.

Configuration lets users customize your server without modifying code.

## What to Learn

### How Configuration Works

1. Client sends initial config (optionally) during/after initialize
2. User changes settings in editor
3. Editor sends `workspace/didChangeConfiguration` notification
4. Server updates its behavior

### Configuration Notification

```json
{
  "jsonrpc": "2.0",
  "method": "workspace/didChangeConfiguration",
  "params": {
    "settings": {
      "wordlib": {
        "diagnosticSeverity": "warning",
        "caseSensitive": false,
        "maxSuggestionDistance": 2,
        "dictionaryPath": "/custom/path/dictionary.txt"
      }
    }
  }
}
```

### Your Configuration Options

Based on requirements:

```c
typedef struct {
    // Diagnostic severity: error(1), warning(2), info(3), hint(4)
    int diagnostic_severity;
    
    // Case sensitivity for word checking
    bool case_sensitive;
    
    // Maximum edit distance for suggestions
    int max_suggestion_distance;
    
    // Dictionary paths
    char *global_dictionary_path;
    char *workspace_dictionary_path;
    
    // Enabled file types
    char **enabled_languages;
    size_t enabled_languages_count;
} ServerConfig;
```

### Default Configuration

```c
ServerConfig default_config(void) {
    return (ServerConfig){
        .diagnostic_severity = 3,  // Information
        .case_sensitive = false,
        .max_suggestion_distance = 2,
        .global_dictionary_path = NULL,  // Use default
        .workspace_dictionary_path = NULL,  // Derived from workspace root
        .enabled_languages = NULL,  // All languages
        .enabled_languages_count = 0
    };
}
```

### Handling Configuration Change

```c
void handle_did_change_configuration(Server *server, const JsonValue *params) {
    const JsonValue *settings = json_get(params, "settings");
    const JsonValue *wordlib = json_get(settings, "wordlib");
    
    if (!wordlib) return;
    
    // Update severity
    const char *severity = json_get_string(wordlib, "diagnosticSeverity");
    if (severity) {
        server->config.diagnostic_severity = parse_severity(severity);
    }
    
    // Update case sensitivity
    if (json_has_key(wordlib, "caseSensitive")) {
        server->config.case_sensitive = json_get_bool(wordlib, "caseSensitive");
    }
    
    // Update max distance
    if (json_has_key(wordlib, "maxSuggestionDistance")) {
        server->config.max_suggestion_distance = 
            json_get_int(wordlib, "maxSuggestionDistance");
    }
    
    // Update dictionary path
    const char *dict_path = json_get_string(wordlib, "dictionaryPath");
    if (dict_path) {
        free(server->config.global_dictionary_path);
        server->config.global_dictionary_path = strdup(dict_path);
        reload_dictionary(server);
    }
    
    // Revalidate with new settings
    revalidate_all_documents(server);
}

int parse_severity(const char *severity) {
    if (strcmp(severity, "error") == 0) return 1;
    if (strcmp(severity, "warning") == 0) return 2;
    if (strcmp(severity, "information") == 0) return 3;
    if (strcmp(severity, "hint") == 0) return 4;
    return 3;  // Default to information
}
```

### Pull Configuration (Alternative)

Some editors prefer server requesting config:

```c
void request_configuration(Server *server) {
    // Send workspace/configuration request
    JsonValue *request = json_object();
    json_set_string(request, "jsonrpc", "2.0");
    json_set_int(request, "id", server->next_request_id++);
    json_set_string(request, "method", "workspace/configuration");
    
    JsonValue *params = json_object();
    JsonValue *items = json_array();
    
    JsonValue *item = json_object();
    json_set_string(item, "section", "wordlib");
    json_array_push(items, item);
    
    json_set(params, "items", items);
    json_set(request, "params", params);
    
    // Send and handle response
    send_request(server, request, handle_config_response);
}
```

But `didChangeConfiguration` is simpler and sufficient.

### Applying Configuration

Use config values where relevant:

```c
// In diagnostic creation
JsonValue *create_diagnostic(Server *server, Range range, const char *word) {
    JsonValue *diag = json_object();
    // ... range, message, etc ...
    
    // Use configured severity
    json_set_int(diag, "severity", server->config.diagnostic_severity);
    
    return diag;
}

// In word checking
bool is_word_known(Server *server, const char *word) {
    if (server->config.case_sensitive) {
        return wordlib_contains(server->engine, word);
    } else {
        // Check with lowercase
        char *lower = str_tolower(word);
        bool known = wordlib_contains(server->engine, lower);
        free(lower);
        return known;
    }
}
```

### File Type Filtering

```c
bool is_language_enabled(Server *server, const char *language_id) {
    // If no filter configured, all languages enabled
    if (server->config.enabled_languages_count == 0) {
        return true;
    }
    
    for (size_t i = 0; i < server->config.enabled_languages_count; i++) {
        if (strcmp(server->config.enabled_languages[i], language_id) == 0) {
            return true;
        }
    }
    
    return false;
}

void handle_did_open(Server *server, const JsonValue *params) {
    // ...
    const char *language = json_get_string(td, "languageId");
    
    if (!is_language_enabled(server, language)) {
        return;  // Don't process this document
    }
    
    // ... continue
}
```

## Where to Learn

1. **LSP Specification - Configuration:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeConfiguration

2. **Editor settings:**
   - How VS Code, Neovim expose LSP settings

## Practice Exercise

Test configuration handling:

```c
void test_configuration(void) {
    Server *server = test_server_create();
    
    // Default severity
    assert(server->config.diagnostic_severity == 3);
    
    // Change via notification
    JsonValue *params = json_parse(
        "{\"settings\":{\"wordlib\":{"
        "\"diagnosticSeverity\":\"warning\","
        "\"caseSensitive\":true}}}");
    
    handle_did_change_configuration(server, params);
    
    assert(server->config.diagnostic_severity == 2);
    assert(server->config.case_sensitive == true);
    
    json_free(params);
    test_server_destroy(server);
    printf("test_configuration: PASS\n");
}
```

## Connection to Project

Configuration makes your server adaptable:

```
User A: "I want unknown words as errors"
  → diagnosticSeverity: "error"

User B: "I want case-sensitive checking"
  → caseSensitive: true

User C: "Only check markdown files"
  → enabledLanguages: ["markdown"]
```

Without configuration, one size fits all. With configuration, everyone's happy.
