# Section 7 Project: Complete Configuration System

## Objective

Build a configuration system that lets users customize your server while providing sensible defaults for zero-config operation.

## Requirements

1. **Declare accurate server capabilities**
2. **Handle workspace/didChangeConfiguration**
3. **Support global and workspace dictionaries**
4. **Provide working defaults** for all settings
5. **Validate and apply configuration safely**

## Specification

### Configuration Types (`include/config.h`)

```c
#ifndef CONFIG_H
#define CONFIG_H

#include "json.h"
#include <stdbool.h>

// Diagnostic severity levels
typedef enum {
    SEVERITY_ERROR = 1,
    SEVERITY_WARNING = 2,
    SEVERITY_INFORMATION = 3,
    SEVERITY_HINT = 4
} DiagnosticSeverity;

// Server configuration
typedef struct {
    DiagnosticSeverity diagnostic_severity;
    bool case_sensitive;
    int max_suggestion_distance;
    int max_completions;
    int min_prefix_length;
    char *global_dictionary_path;
    char *workspace_dictionary_path;
} ServerConfig;

// Dictionary paths
typedef struct {
    char *global_path;
    char *workspace_path;
} DictionaryPaths;

// Configuration functions
ServerConfig config_default(void);
void config_destroy(ServerConfig *config);
void config_apply(ServerConfig *config, const JsonValue *settings);

// Path resolution
DictionaryPaths resolve_paths(const char *workspace_uri);
void paths_destroy(DictionaryPaths *paths);

// Helpers
DiagnosticSeverity parse_severity(const char *str);
const char *severity_to_string(DiagnosticSeverity sev);

#endif
```

### Capabilities Module (`include/capabilities.h`)

```c
#ifndef CAPABILITIES_H
#define CAPABILITIES_H

#include "json.h"

// Build server capabilities for initialize response
JsonValue *build_server_capabilities(void);

// Parse client capabilities from initialize request
typedef struct {
    bool supports_related_diagnostics;
    bool supports_workspace_configuration;
} ClientCapabilities;

ClientCapabilities parse_client_capabilities(const JsonValue *caps);

#endif
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── config.h
│   ├── capabilities.h
│   └── ... (previous headers)
├── src/
│   ├── config.c
│   ├── capabilities.c
│   └── ... (previous sources)
├── test/
│   └── test_config.c
└── Makefile
```

## Implementation Guide

### Step 1: Default Configuration

```c
// src/config.c
#include "config.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

ServerConfig config_default(void) {
    ServerConfig config = {
        .diagnostic_severity = SEVERITY_INFORMATION,
        .case_sensitive = false,
        .max_suggestion_distance = 2,
        .max_completions = 50,
        .min_prefix_length = 1,
        .global_dictionary_path = NULL,
        .workspace_dictionary_path = NULL
    };
    
    // Set default global path
    const char *home = getenv("HOME");
    if (home) {
        size_t len = strlen(home) + 30;
        config.global_dictionary_path = malloc(len);
        snprintf(config.global_dictionary_path, len,
                 "%s/.wordlib/dictionary.txt", home);
    }
    
    return config;
}

void config_destroy(ServerConfig *config) {
    free(config->global_dictionary_path);
    free(config->workspace_dictionary_path);
    config->global_dictionary_path = NULL;
    config->workspace_dictionary_path = NULL;
}
```

### Step 2: Configuration Parsing

```c
DiagnosticSeverity parse_severity(const char *str) {
    if (!str) return SEVERITY_INFORMATION;
    
    if (strcmp(str, "error") == 0) return SEVERITY_ERROR;
    if (strcmp(str, "warning") == 0) return SEVERITY_WARNING;
    if (strcmp(str, "information") == 0) return SEVERITY_INFORMATION;
    if (strcmp(str, "info") == 0) return SEVERITY_INFORMATION;
    if (strcmp(str, "hint") == 0) return SEVERITY_HINT;
    
    return SEVERITY_INFORMATION;
}

const char *severity_to_string(DiagnosticSeverity sev) {
    switch (sev) {
        case SEVERITY_ERROR: return "error";
        case SEVERITY_WARNING: return "warning";
        case SEVERITY_INFORMATION: return "information";
        case SEVERITY_HINT: return "hint";
        default: return "information";
    }
}

void config_apply(ServerConfig *config, const JsonValue *settings) {
    if (!settings) return;
    
    const JsonValue *wordlib = json_get(settings, "wordlib");
    if (!wordlib) return;
    
    // Severity
    if (json_has_key(wordlib, "diagnosticSeverity")) {
        const char *sev = json_get_string(wordlib, "diagnosticSeverity");
        config->diagnostic_severity = parse_severity(sev);
    }
    
    // Case sensitivity
    if (json_has_key(wordlib, "caseSensitive")) {
        config->case_sensitive = json_get_bool(wordlib, "caseSensitive");
    }
    
    // Max suggestions
    if (json_has_key(wordlib, "maxSuggestionDistance")) {
        int val = (int)json_get_int(wordlib, "maxSuggestionDistance");
        if (val >= 1 && val <= 5) {
            config->max_suggestion_distance = val;
        }
    }
    
    // Dictionary path
    if (json_has_key(wordlib, "dictionaryPath")) {
        const char *path = json_get_string(wordlib, "dictionaryPath");
        if (path && strlen(path) > 0) {
            free(config->global_dictionary_path);
            config->global_dictionary_path = strdup(path);
        }
    }
}
```

### Step 3: Path Resolution

```c
DictionaryPaths resolve_paths(const char *workspace_uri) {
    DictionaryPaths paths = {0};
    
    // Global path
    const char *home = getenv("HOME");
    if (home) {
        size_t len = strlen(home) + 30;
        paths.global_path = malloc(len);
        snprintf(paths.global_path, len,
                 "%s/.wordlib/dictionary.txt", home);
    }
    
    // Workspace path
    if (workspace_uri) {
        char *workspace_path = uri_to_path(workspace_uri);
        if (workspace_path) {
            size_t len = strlen(workspace_path) + 30;
            paths.workspace_path = malloc(len);
            snprintf(paths.workspace_path, len,
                     "%s/.wordlib/dictionary.txt", workspace_path);
            free(workspace_path);
        }
    }
    
    return paths;
}

void paths_destroy(DictionaryPaths *paths) {
    free(paths->global_path);
    free(paths->workspace_path);
    paths->global_path = NULL;
    paths->workspace_path = NULL;
}
```

### Step 4: Server Capabilities

```c
// src/capabilities.c
#include "capabilities.h"

JsonValue *build_server_capabilities(void) {
    JsonValue *caps = json_object();
    
    // Text sync
    JsonValue *text_sync = json_object();
    json_set_bool(text_sync, "openClose", true);
    json_set_int(text_sync, "change", 1);  // Full sync
    json_set(caps, "textDocumentSync", text_sync);
    
    // Completion
    JsonValue *completion = json_object();
    json_set(completion, "triggerCharacters", json_array());
    json_set_bool(completion, "resolveProvider", false);
    json_set(caps, "completionProvider", completion);
    
    // Code actions
    JsonValue *code_action = json_object();
    JsonValue *kinds = json_array();
    json_array_push(kinds, json_string("quickfix"));
    json_set(code_action, "codeActionKinds", kinds);
    json_set(caps, "codeActionProvider", code_action);
    
    // Execute command
    JsonValue *exec = json_object();
    JsonValue *commands = json_array();
    json_array_push(commands, json_string("wordlib.addWord"));
    json_array_push(commands, json_string("wordlib.ignoreWord"));
    json_set(exec, "commands", commands);
    json_set(caps, "executeCommandProvider", exec);
    
    return caps;
}

ClientCapabilities parse_client_capabilities(const JsonValue *caps) {
    ClientCapabilities client = {0};
    
    if (!caps) return client;
    
    // Check for specific capabilities
    const JsonValue *td = json_get(caps, "textDocument");
    if (td) {
        const JsonValue *diag = json_get(td, "publishDiagnostics");
        if (diag) {
            client.supports_related_diagnostics = 
                json_get_bool(diag, "relatedInformation");
        }
    }
    
    const JsonValue *ws = json_get(caps, "workspace");
    if (ws) {
        client.supports_workspace_configuration = 
            json_get_bool(ws, "configuration");
    }
    
    return client;
}
```

### Step 5: Handler Updates

```c
// Update handle_initialize
JsonValue *handle_initialize(Server *server, const JsonValue *params) {
    server->state = SERVER_INITIALIZING;
    
    // Parse client capabilities
    server->client_caps = parse_client_capabilities(
        json_get(params, "capabilities"));
    
    // Get workspace root
    const char *root_uri = json_get_string(params, "rootUri");
    if (root_uri) {
        server->workspace_root = strdup(root_uri);
    }
    
    // Resolve dictionary paths
    server->paths = resolve_paths(root_uri);
    
    // Load dictionaries
    load_dictionaries(server);
    
    // Build response
    JsonValue *result = json_object();
    json_set(result, "capabilities", build_server_capabilities());
    
    JsonValue *info = json_object();
    json_set_string(info, "name", "wordlib-lsp");
    json_set_string(info, "version", "1.0.0");
    json_set(result, "serverInfo", info);
    
    return result;
}

// Update handle_did_change_configuration
void handle_did_change_configuration(Server *server, const JsonValue *params) {
    const JsonValue *settings = json_get(params, "settings");
    config_apply(&server->config, settings);
    
    // Reload dictionaries if path changed
    // (compare with previous path)
    
    // Revalidate all documents
    revalidate_all_documents(server);
}
```

## Test Cases

```c
// test/test_config.c
#include "config.h"
#include "capabilities.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>

void test_default_config(void) {
    ServerConfig config = config_default();
    
    assert(config.diagnostic_severity == SEVERITY_INFORMATION);
    assert(config.case_sensitive == false);
    assert(config.max_completions == 50);
    assert(config.global_dictionary_path != NULL);
    
    config_destroy(&config);
    printf("test_default_config: PASS\n");
}

void test_parse_severity(void) {
    assert(parse_severity("error") == SEVERITY_ERROR);
    assert(parse_severity("warning") == SEVERITY_WARNING);
    assert(parse_severity("information") == SEVERITY_INFORMATION);
    assert(parse_severity("info") == SEVERITY_INFORMATION);
    assert(parse_severity("hint") == SEVERITY_HINT);
    assert(parse_severity("invalid") == SEVERITY_INFORMATION);
    assert(parse_severity(NULL) == SEVERITY_INFORMATION);
    
    printf("test_parse_severity: PASS\n");
}

void test_config_apply(void) {
    ServerConfig config = config_default();
    
    JsonValue *settings = json_parse(
        "{\"wordlib\":{"
        "\"diagnosticSeverity\":\"warning\","
        "\"caseSensitive\":true,"
        "\"maxSuggestionDistance\":3}}");
    
    config_apply(&config, settings);
    
    assert(config.diagnostic_severity == SEVERITY_WARNING);
    assert(config.case_sensitive == true);
    assert(config.max_suggestion_distance == 3);
    
    json_free(settings);
    config_destroy(&config);
    printf("test_config_apply: PASS\n");
}

void test_path_resolution(void) {
    DictionaryPaths paths = resolve_paths("file:///home/user/project");
    
    assert(paths.global_path != NULL);
    assert(strstr(paths.global_path, ".wordlib") != NULL);
    assert(paths.workspace_path != NULL);
    assert(strstr(paths.workspace_path, "project/.wordlib") != NULL);
    
    paths_destroy(&paths);
    printf("test_path_resolution: PASS\n");
}

void test_capabilities(void) {
    JsonValue *caps = build_server_capabilities();
    
    // Text sync
    assert(json_has_key(caps, "textDocumentSync"));
    
    // Completion
    assert(json_has_key(caps, "completionProvider"));
    
    // Code actions
    assert(json_has_key(caps, "codeActionProvider"));
    
    // Execute command
    assert(json_has_key(caps, "executeCommandProvider"));
    
    json_free(caps);
    printf("test_capabilities: PASS\n");
}

int main(void) {
    test_default_config();
    test_parse_severity();
    test_config_apply();
    test_path_resolution();
    test_capabilities();
    printf("\nAll config tests passed!\n");
    return 0;
}
```

## Acceptance Criteria

### Zero-Config Works
```bash
# Server works without any configuration
./wordlib-lsp  # Just starts and works
```

### Configuration Applies
```lua
-- In Neovim/editor
vim.lsp.config.wordlib = {
  settings = {
    wordlib = {
      diagnosticSeverity = "warning",
      caseSensitive = true
    }
  }
}
-- Changes take effect immediately
```

### All Tests Pass
```bash
$ make test_config && ./test_config
test_default_config: PASS
test_parse_severity: PASS
test_config_apply: PASS
test_path_resolution: PASS
test_capabilities: PASS
All config tests passed!
```

## Stretch Goals

1. **Dynamic registration** - Register capabilities after init
2. **Config file support** - Read from `.wordlib/config.json`
3. **Per-document settings** - Different settings per language

## What You'll Learn

By completing this project:
- LSP capability negotiation
- Configuration handling patterns
- Path resolution for workspace features
- Default value management

## Next Section Preview

Section 8 focuses on robustness—error handling, logging, resource management, and testing strategies to make your server production-ready.
