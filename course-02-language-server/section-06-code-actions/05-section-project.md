# Section 6 Project: Code Action Provider

## Objective

Build a complete code action system that lets users add unknown words to the dictionary with a single click, and optionally ignore words for the current session.

## Requirements

1. **Handle textDocument/codeAction requests**
2. **Offer "Add to dictionary" action** for unknown words
3. **Offer "Ignore for session" action** (optional but recommended)
4. **Handle workspace/executeCommand** for action execution
5. **Trigger revalidation** after dictionary changes
6. **Persist dictionary changes**

## Specification

### Code Action API (`include/code_actions.h`)

```c
#ifndef CODE_ACTIONS_H
#define CODE_ACTIONS_H

#include "json.h"
#include "diagnostics.h"

// Forward declaration
typedef struct Server Server;

// Code action kinds
#define CODE_ACTION_QUICKFIX "quickfix"

// Commands
#define CMD_ADD_WORD    "wordlib.addWord"
#define CMD_IGNORE_WORD "wordlib.ignoreWord"

// Session ignore list
typedef struct IgnoreList IgnoreList;

IgnoreList *ignore_list_create(void);
void ignore_list_destroy(IgnoreList *list);
void ignore_list_add(IgnoreList *list, const char *word);
bool ignore_list_contains(const IgnoreList *list, const char *word);

// Code action handlers
JsonValue *handle_code_action(Server *server, const JsonValue *params);
JsonValue *handle_execute_command(Server *server, const JsonValue *params);

// Action creation
JsonValue *create_add_word_action(const char *word, const JsonValue *diagnostic);
JsonValue *create_ignore_action(const char *word, const JsonValue *diagnostic);

// Revalidation
void revalidate_all_documents(Server *server);

#endif
```

### Server Structure Update

```c
// Add to Server struct
typedef struct Server {
    // ... existing fields ...
    IgnoreList *session_ignore;
    char *workspace_dict_path;
} Server;
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── code_actions.h
│   └── ... (previous headers)
├── src/
│   ├── code_actions.c
│   ├── ignore_list.c
│   └── ... (previous sources)
├── test/
│   └── test_code_actions.c
└── Makefile
```

## Implementation Guide

### Step 1: Ignore List

```c
// src/ignore_list.c
#include "code_actions.h"
#include <stdlib.h>
#include <string.h>

struct IgnoreList {
    char **words;
    size_t count;
    size_t capacity;
};

IgnoreList *ignore_list_create(void) {
    IgnoreList *list = calloc(1, sizeof(IgnoreList));
    if (!list) return NULL;
    
    list->capacity = 32;
    list->words = malloc(list->capacity * sizeof(char *));
    if (!list->words) {
        free(list);
        return NULL;
    }
    
    return list;
}

void ignore_list_destroy(IgnoreList *list) {
    if (!list) return;
    
    for (size_t i = 0; i < list->count; i++) {
        free(list->words[i]);
    }
    free(list->words);
    free(list);
}

void ignore_list_add(IgnoreList *list, const char *word) {
    if (!list || !word) return;
    
    // Check if already present
    if (ignore_list_contains(list, word)) return;
    
    // Grow if needed
    if (list->count >= list->capacity) {
        list->capacity *= 2;
        list->words = realloc(list->words, list->capacity * sizeof(char *));
    }
    
    list->words[list->count++] = strdup(word);
}

bool ignore_list_contains(const IgnoreList *list, const char *word) {
    if (!list || !word) return false;
    
    for (size_t i = 0; i < list->count; i++) {
        if (strcmp(list->words[i], word) == 0) {
            return true;
        }
    }
    return false;
}
```

### Step 2: Action Creation

```c
// src/code_actions.c
#include "code_actions.h"
#include <stdio.h>
#include <string.h>

JsonValue *create_add_word_action(const char *word, const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    // Title
    char title[256];
    snprintf(title, sizeof(title), "Add '%s' to dictionary", word);
    json_set_string(action, "title", title);
    
    // Kind
    json_set_string(action, "kind", CODE_ACTION_QUICKFIX);
    
    // Diagnostics
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    // Command
    JsonValue *command = json_object();
    json_set_string(command, "title", "Add to dictionary");
    json_set_string(command, "command", CMD_ADD_WORD);
    
    JsonValue *args = json_array();
    json_array_push(args, json_string(word));
    json_set(command, "arguments", args);
    
    json_set(action, "command", command);
    
    return action;
}

JsonValue *create_ignore_action(const char *word, const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    char title[256];
    snprintf(title, sizeof(title), "Ignore '%s' for this session", word);
    json_set_string(action, "title", title);
    json_set_string(action, "kind", CODE_ACTION_QUICKFIX);
    
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    JsonValue *command = json_object();
    json_set_string(command, "title", "Ignore word");
    json_set_string(command, "command", CMD_IGNORE_WORD);
    
    JsonValue *args = json_array();
    json_array_push(args, json_string(word));
    json_set(command, "arguments", args);
    
    json_set(action, "command", command);
    
    return action;
}
```

### Step 3: Word Extraction

```c
// Extract word from diagnostic message "Unknown word: 'xyz'"
static char *extract_word_from_diagnostic(const JsonValue *diagnostic) {
    // First try data field
    const JsonValue *data = json_get(diagnostic, "data");
    if (data) {
        const char *word = json_get_string(data, "word");
        if (word) return strdup(word);
    }
    
    // Fall back to parsing message
    const char *message = json_get_string(diagnostic, "message");
    if (!message) return NULL;
    
    const char *start = strchr(message, '\'');
    if (!start) return NULL;
    start++;
    
    const char *end = strchr(start, '\'');
    if (!end) return NULL;
    
    size_t len = end - start;
    char *word = malloc(len + 1);
    memcpy(word, start, len);
    word[len] = '\0';
    
    return word;
}
```

### Step 4: Code Action Handler

```c
JsonValue *handle_code_action(Server *server, const JsonValue *params) {
    const JsonValue *context = json_get(params, "context");
    const JsonValue *diagnostics = json_get(context, "diagnostics");
    
    JsonValue *actions = json_array();
    
    size_t diag_count = json_array_length(diagnostics);
    for (size_t i = 0; i < diag_count; i++) {
        const JsonValue *diag = json_array_get(diagnostics, i);
        
        // Only handle our diagnostics
        const char *code = json_get_string(diag, "code");
        if (!code || strcmp(code, "wordlib.unknown") != 0) {
            continue;
        }
        
        // Extract word
        char *word = extract_word_from_diagnostic(diag);
        if (!word) continue;
        
        // Create actions
        json_array_push(actions, create_add_word_action(word, diag));
        json_array_push(actions, create_ignore_action(word, diag));
        
        free(word);
    }
    
    return actions;
}
```

### Step 5: Execute Command Handler

```c
JsonValue *handle_execute_command(Server *server, const JsonValue *params) {
    const char *command = json_get_string(params, "command");
    const JsonValue *args = json_get(params, "arguments");
    
    if (strcmp(command, CMD_ADD_WORD) == 0) {
        const char *word = json_get_string(json_array_get(args, 0), NULL);
        if (word) {
            // Add to dictionary
            wordlib_add_word(server->engine, word);
            
            // Persist
            if (server->workspace_dict_path) {
                wordlib_save(server->engine, server->workspace_dict_path);
            }
            
            // Revalidate
            revalidate_all_documents(server);
        }
    }
    else if (strcmp(command, CMD_IGNORE_WORD) == 0) {
        const char *word = json_get_string(json_array_get(args, 0), NULL);
        if (word) {
            ignore_list_add(server->session_ignore, word);
            revalidate_all_documents(server);
        }
    }
    
    return json_null();
}
```

### Step 6: Revalidation

```c
static void republish_diagnostics(Document *doc, void *userdata) {
    Server *server = (Server *)userdata;
    publish_diagnostics(server, doc->uri);
}

void revalidate_all_documents(Server *server) {
    document_store_foreach(server->documents, republish_diagnostics, server);
}
```

### Step 7: Update Diagnostic Computation

```c
// In diagnostics.c, update to check ignore list
JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    // ... tokenization loop ...
    
    // Check if word is known OR ignored
    if (!wordlib_contains(server->engine, word) &&
        !ignore_list_contains(server->session_ignore, word)) {
        // Create diagnostic with word in data field
        JsonValue *diag = create_diagnostic(range, word);
        
        // Add word to data for easy extraction
        JsonValue *data = json_object();
        json_set_string(data, "word", word);
        json_set(diag, "data", data);
        
        json_array_push(diagnostics, diag);
    }
    
    // ...
}
```

### Step 8: Register Handlers

```c
// In dispatcher
JsonValue *dispatch_request(Server *server, int64_t id,
                           const char *method, const JsonValue *params) {
    // ... existing handlers ...
    
    if (strcmp(method, "textDocument/codeAction") == 0) {
        return handle_code_action(server, params);
    }
    
    if (strcmp(method, "workspace/executeCommand") == 0) {
        return handle_execute_command(server, params);
    }
    
    // ...
}
```

## Test Cases

```c
// test/test_code_actions.c
#include "code_actions.h"
#include "server.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>

void test_ignore_list(void) {
    IgnoreList *list = ignore_list_create();
    
    assert(!ignore_list_contains(list, "test"));
    
    ignore_list_add(list, "test");
    assert(ignore_list_contains(list, "test"));
    
    // Duplicate add is safe
    ignore_list_add(list, "test");
    
    ignore_list_destroy(list);
    printf("test_ignore_list: PASS\n");
}

void test_action_creation(void) {
    JsonValue *diag = json_parse(
        "{\"message\":\"Unknown word: 'quikc'\","
        "\"code\":\"wordlib.unknown\"}");
    
    JsonValue *action = create_add_word_action("quikc", diag);
    
    assert(strstr(json_get_string(action, "title"), "quikc") != NULL);
    assert(strcmp(json_get_string(action, "kind"), "quickfix") == 0);
    
    JsonValue *cmd = json_get(action, "command");
    assert(strcmp(json_get_string(cmd, "command"), CMD_ADD_WORD) == 0);
    
    json_free(action);
    json_free(diag);
    printf("test_action_creation: PASS\n");
}

void test_code_action_handler(void) {
    Server *server = test_server_create();
    
    // Create code action request with diagnostic
    JsonValue *params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"range\":{\"start\":{\"line\":0,\"character\":0},"
        "         \"end\":{\"line\":0,\"character\":5}},"
        "\"context\":{\"diagnostics\":["
        "  {\"message\":\"Unknown word: 'quikc'\","
        "   \"code\":\"wordlib.unknown\","
        "   \"range\":{\"start\":{\"line\":0,\"character\":0},"
        "             \"end\":{\"line\":0,\"character\":5}}}"
        "]}}");
    
    JsonValue *result = handle_code_action(server, params);
    
    // Should have 2 actions: add + ignore
    assert(json_array_length(result) == 2);
    
    json_free(params);
    json_free(result);
    test_server_destroy(server);
    printf("test_code_action_handler: PASS\n");
}

void test_execute_add_word(void) {
    Server *server = test_server_create();
    
    // Initially unknown
    assert(!wordlib_contains(server->engine, "testword"));
    
    // Execute addWord
    JsonValue *params = json_parse(
        "{\"command\":\"wordlib.addWord\","
        "\"arguments\":[\"testword\"]}");
    
    handle_execute_command(server, params);
    
    // Now known
    assert(wordlib_contains(server->engine, "testword"));
    
    json_free(params);
    test_server_destroy(server);
    printf("test_execute_add_word: PASS\n");
}

void test_execute_ignore_word(void) {
    Server *server = test_server_create();
    
    // Initially not ignored
    assert(!ignore_list_contains(server->session_ignore, "ignored"));
    
    // Execute ignoreWord
    JsonValue *params = json_parse(
        "{\"command\":\"wordlib.ignoreWord\","
        "\"arguments\":[\"ignored\"]}");
    
    handle_execute_command(server, params);
    
    // Now ignored
    assert(ignore_list_contains(server->session_ignore, "ignored"));
    
    json_free(params);
    test_server_destroy(server);
    printf("test_execute_ignore_word: PASS\n");
}

void test_revalidation(void) {
    Server *server = test_server_with_transport();
    
    // Open document with unknown word
    document_store_open(server->documents,
        "file:///test.txt", "plaintext", 1, "Hello quikc");
    
    // Initial check: "quikc" should trigger diagnostic
    Document *doc = document_store_get(server->documents, "file:///test.txt");
    JsonValue *diags1 = compute_diagnostics(server, doc);
    assert(json_array_length(diags1) == 1);
    json_free(diags1);
    
    // Add word
    wordlib_add_word(server->engine, "quikc");
    
    // After revalidation: no diagnostics
    JsonValue *diags2 = compute_diagnostics(server, doc);
    assert(json_array_length(diags2) == 0);
    json_free(diags2);
    
    test_server_destroy(server);
    printf("test_revalidation: PASS\n");
}

int main(void) {
    test_ignore_list();
    test_action_creation();
    test_code_action_handler();
    test_execute_add_word();
    test_execute_ignore_word();
    test_revalidation();
    printf("\nAll code action tests passed!\n");
    return 0;
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_code_actions && ./test_code_actions
test_ignore_list: PASS
test_action_creation: PASS
test_code_action_handler: PASS
test_execute_add_word: PASS
test_execute_ignore_word: PASS
test_revalidation: PASS
All code action tests passed!
```

### Integration Test
```
1. Open file with unknown word
2. See squiggle under word
3. Click lightbulb / press shortcut
4. See "Add to dictionary" and "Ignore" options
5. Click "Add to dictionary"
6. Squiggle disappears
7. Word persists after restart
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_code_actions
All heap blocks were freed -- no leaks are possible
```

## Stretch Goals

1. **Replace with suggestions** - If engine has BK-tree, offer spelling corrections
2. **Add to workspace vs global** - Let user choose where to add word
3. **Bulk ignore** - "Ignore all occurrences of this type"
4. **Undo support** - Remember recently added words for potential removal

## What You'll Learn

By completing this project:
- Code action request/response handling
- Command execution flow
- State management (session ignore)
- Document revalidation patterns

## Next Section Preview

Section 7 implements configuration and capabilities, letting users customize behavior (severity levels, dictionary paths) and ensuring proper feature advertisement.
