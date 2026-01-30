# Section 8 Project: Production-Ready Server

## Objective

Harden your LSP server for production use with comprehensive error handling, logging, resource management, and testing.

## Requirements

1. **Graceful error handling** - No crashes on bad input
2. **Proper logging** - stderr/file without protocol interference
3. **Resource cleanup** - No memory leaks
4. **Comprehensive tests** - Unit, integration, and protocol tests
5. **Documentation** - Usage instructions and configuration reference

## Specification

### Logging Module (`include/log.h`)

```c
#ifndef LOG_H
#define LOG_H

typedef enum {
    LOG_ERROR = 0,
    LOG_WARN = 1,
    LOG_INFO = 2,
    LOG_DEBUG = 3
} LogLevel;

// Initialize logging
void log_init(const char *file_path);  // NULL for stderr only
void log_shutdown(void);

// Set log level
void log_set_level(LogLevel level);
LogLevel log_get_level(void);

// Log functions
void log_error(const char *fmt, ...);
void log_warn(const char *fmt, ...);
void log_info(const char *fmt, ...);
void log_debug(const char *fmt, ...);

#endif
```

### Error Handling (`include/errors.h`)

```c
#ifndef ERRORS_H
#define ERRORS_H

#include "json.h"

// JSON-RPC error codes
#define ERR_PARSE_ERROR       -32700
#define ERR_INVALID_REQUEST   -32600
#define ERR_METHOD_NOT_FOUND  -32601
#define ERR_INVALID_PARAMS    -32602
#define ERR_INTERNAL_ERROR    -32603
#define ERR_SERVER_NOT_INIT   -32002

// Error response creation
JsonValue *error_response(int64_t id, int code, const char *message);

// Validation helpers
bool validate_params(const JsonValue *params, const char *required_fields[]);

#endif
```

### Final Directory Structure

```
wordlib-lsp/
├── include/
│   ├── transport.h
│   ├── json.h
│   ├── document_store.h
│   ├── uri.h
│   ├── diagnostics.h
│   ├── completion.h
│   ├── code_actions.h
│   ├── config.h
│   ├── capabilities.h
│   ├── log.h
│   ├── errors.h
│   └── server.h
├── src/
│   ├── main.c
│   ├── server.c
│   ├── transport.c
│   ├── json.c
│   ├── cJSON.c
│   ├── document_store.c
│   ├── uri.c
│   ├── diagnostics.c
│   ├── completion.c
│   ├── code_actions.c
│   ├── config.c
│   ├── capabilities.c
│   ├── log.c
│   └── errors.c
├── test/
│   ├── test_main.c
│   ├── test_unit.c
│   ├── test_integration.c
│   ├── test_protocol.c
│   └── fixtures.h
├── docs/
│   └── CONFIGURATION.md
├── Makefile
└── README.md
```

## Implementation Guide

### Step 1: Logging Implementation

```c
// src/log.c
#include "log.h"
#include <stdio.h>
#include <stdarg.h>
#include <time.h>
#include <string.h>

static FILE *log_file = NULL;
static LogLevel current_level = LOG_INFO;

void log_init(const char *file_path) {
    if (file_path) {
        log_file = fopen(file_path, "a");
        if (!log_file) {
            fprintf(stderr, "Warning: Could not open log file: %s\n", file_path);
        }
    }
}

void log_shutdown(void) {
    if (log_file) {
        fclose(log_file);
        log_file = NULL;
    }
}

void log_set_level(LogLevel level) {
    current_level = level;
}

LogLevel log_get_level(void) {
    return current_level;
}

static void log_write(const char *level, const char *fmt, va_list args) {
    FILE *out = log_file ? log_file : stderr;
    
    // Timestamp
    time_t now = time(NULL);
    struct tm *tm = localtime(&now);
    char timestamp[32];
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", tm);
    
    fprintf(out, "[%s] [%s] ", timestamp, level);
    vfprintf(out, fmt, args);
    fprintf(out, "\n");
    fflush(out);
}

void log_error(const char *fmt, ...) {
    if (current_level >= LOG_ERROR) {
        va_list args;
        va_start(args, fmt);
        log_write("ERROR", fmt, args);
        va_end(args);
    }
}

void log_warn(const char *fmt, ...) {
    if (current_level >= LOG_WARN) {
        va_list args;
        va_start(args, fmt);
        log_write("WARN", fmt, args);
        va_end(args);
    }
}

void log_info(const char *fmt, ...) {
    if (current_level >= LOG_INFO) {
        va_list args;
        va_start(args, fmt);
        log_write("INFO", fmt, args);
        va_end(args);
    }
}

void log_debug(const char *fmt, ...) {
    if (current_level >= LOG_DEBUG) {
        va_list args;
        va_start(args, fmt);
        log_write("DEBUG", fmt, args);
        va_end(args);
    }
}
```

### Step 2: Error Handling

```c
// src/errors.c
#include "errors.h"
#include <string.h>

JsonValue *error_response(int64_t id, int code, const char *message) {
    JsonValue *response = json_object();
    json_set_string(response, "jsonrpc", "2.0");
    json_set_int(response, "id", id);
    
    JsonValue *error = json_object();
    json_set_int(error, "code", code);
    json_set_string(error, "message", message);
    json_set(response, "error", error);
    
    return response;
}

bool validate_params(const JsonValue *params, const char *required_fields[]) {
    if (!params) return false;
    
    for (int i = 0; required_fields[i]; i++) {
        if (!json_has_key(params, required_fields[i])) {
            return false;
        }
    }
    
    return true;
}
```

### Step 3: Robust Main Loop

```c
// src/main.c
#include "server.h"
#include "transport.h"
#include "log.h"
#include <signal.h>
#include <stdlib.h>

static volatile sig_atomic_t should_exit = 0;

static void signal_handler(int sig) {
    (void)sig;
    should_exit = 1;
}

int main(int argc, char **argv) {
    // Initialize logging
    const char *log_file = getenv("WORDLIB_LOG_FILE");
    log_init(log_file);
    
    const char *log_level = getenv("WORDLIB_LOG_LEVEL");
    if (log_level) {
        if (strcmp(log_level, "debug") == 0) log_set_level(LOG_DEBUG);
        else if (strcmp(log_level, "warn") == 0) log_set_level(LOG_WARN);
        else if (strcmp(log_level, "error") == 0) log_set_level(LOG_ERROR);
    }
    
    log_info("wordlib-lsp starting");
    
    // Setup signal handlers
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);
    signal(SIGPIPE, SIG_IGN);
    
    // Create transport and server
    Transport *transport = transport_create(stdin, stdout);
    if (!transport) {
        log_error("Failed to create transport");
        return 1;
    }
    
    Server *server = server_create(transport);
    if (!server) {
        log_error("Failed to create server");
        transport_destroy(transport);
        return 1;
    }
    
    // Main loop
    int exit_code = 0;
    while (!should_exit && server_is_running(server)) {
        char *message = transport_read(transport);
        
        if (!message) {
            if (transport_eof(transport)) {
                log_info("Client disconnected");
                break;
            }
            log_warn("Transport read error, continuing");
            continue;
        }
        
        // Process message with error handling
        if (!server_handle_message(server, message)) {
            log_warn("Message handling failed");
        }
        
        free(message);
    }
    
    // Cleanup
    log_info("Server shutting down");
    server_destroy(server);
    transport_destroy(transport);
    log_shutdown();
    
    return exit_code;
}
```

### Step 4: Comprehensive Tests

```c
// test/test_main.c
#include <stdio.h>

// Import all test functions
extern void test_json_parsing(void);
extern void test_transport(void);
extern void test_document_store(void);
extern void test_uri(void);
extern void test_diagnostics(void);
extern void test_completion(void);
extern void test_code_actions(void);
extern void test_config(void);
extern void test_protocol_flow(void);
extern void test_error_handling(void);

int main(void) {
    printf("Running wordlib-lsp tests\n");
    printf("=========================\n\n");
    
    printf("Unit tests:\n");
    test_json_parsing();
    test_transport();
    test_document_store();
    test_uri();
    test_diagnostics();
    test_completion();
    test_code_actions();
    test_config();
    
    printf("\nIntegration tests:\n");
    test_protocol_flow();
    test_error_handling();
    
    printf("\n=========================\n");
    printf("All tests passed!\n");
    return 0;
}
```

```c
// test/test_protocol.c
#include "server.h"
#include "log.h"
#include <assert.h>
#include <string.h>

void test_protocol_flow(void) {
    Server *server = test_server_create();
    
    // Initialize
    JsonValue *result = server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\","
        "\"params\":{\"rootUri\":null,\"capabilities\":{}}}");
    assert(json_has_key(result, "result"));
    json_free(result);
    
    // Initialized notification
    server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"method\":\"initialized\",\"params\":{}}");
    
    // Open document
    server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"method\":\"textDocument/didOpen\","
        "\"params\":{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\",\"languageId\":\"plaintext\","
        "\"version\":1,\"text\":\"Hello wrold\"}}}");
    
    // Should have diagnostic published (check output)
    
    // Completion
    result = server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"textDocument/completion\","
        "\"params\":{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":5}}}");
    assert(json_has_key(result, "result"));
    json_free(result);
    
    // Shutdown
    result = server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"shutdown\"}");
    assert(json_has_key(result, "result"));
    json_free(result);
    
    test_server_destroy(server);
    printf("test_protocol_flow: PASS\n");
}

void test_error_handling(void) {
    Server *server = test_server_create();
    
    // Bad JSON - should not crash
    JsonValue *result = server_handle_json(server, "not json");
    // Might be NULL or error, but shouldn't crash
    json_free(result);
    
    // Unknown method
    result = server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"unknown\"}");
    // Should return error response
    assert(result == NULL || json_has_key(result, "error"));
    json_free(result);
    
    // Missing required field
    result = server_handle_json(server,
        "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"textDocument/completion\"}");
    // Should handle gracefully
    json_free(result);
    
    test_server_destroy(server);
    printf("test_error_handling: PASS\n");
}
```

### Step 5: Documentation

```markdown
<!-- docs/CONFIGURATION.md -->
# wordlib-lsp Configuration

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `WORDLIB_LOG_FILE` | Path to log file | stderr |
| `WORDLIB_LOG_LEVEL` | Log level (error/warn/info/debug) | info |

## LSP Configuration

In your editor's LSP settings:

```json
{
  "wordlib": {
    "diagnosticSeverity": "warning",
    "caseSensitive": false,
    "dictionaryPath": "/custom/path/dictionary.txt"
  }
}
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `diagnosticSeverity` | string | "information" | error/warning/information/hint |
| `caseSensitive` | boolean | false | Case-sensitive spell checking |
| `dictionaryPath` | string | ~/.wordlib/dictionary.txt | Custom dictionary path |

## Dictionary Files

- Global: `~/.wordlib/dictionary.txt`
- Workspace: `<workspace>/.wordlib/dictionary.txt`

One word per line. Lines starting with `#` are comments.
```

## Acceptance Criteria

### No Crashes
```bash
# Malformed input doesn't crash
echo "garbage" | ./wordlib-lsp
echo '{"bad":' | ./wordlib-lsp
# Server exits cleanly
```

### Memory Clean
```bash
$ valgrind --leak-check=full ./test_main
All heap blocks were freed -- no leaks are possible
```

### All Tests Pass
```bash
$ make test
Running wordlib-lsp tests
=========================

Unit tests:
test_json_parsing: PASS
test_transport: PASS
... all pass ...

Integration tests:
test_protocol_flow: PASS
test_error_handling: PASS

=========================
All tests passed!
```

### Works with Editor
```lua
-- Neovim integration works
-- Unknown words highlighted
-- Completions appear
-- Code actions work
-- No errors in :LspLog
```

## Final Checklist

Before calling your server "production-ready":

- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] Valgrind shows no leaks
- [ ] ASAN shows no errors
- [ ] Logging works without protocol interference
- [ ] Configuration documented
- [ ] Works in Neovim
- [ ] Handles malformed input gracefully
- [ ] Clean shutdown works
- [ ] Dictionary persists correctly

## What You've Built

Congratulations! You now have:

- A fully functional LSP server
- Unknown word highlighting
- Word completion
- Add-to-dictionary code actions
- Workspace and global dictionaries
- Configurable settings
- Production-quality error handling
- Comprehensive test suite

## Next Steps

Potential enhancements:
- Spelling suggestions (using BK-tree from Course 1)
- Grammar checking
- Multi-language support
- VS Code extension packaging
- Performance optimization

*Your spell checker is ready for real use!* 🎉
