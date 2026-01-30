# Dictionary Management Actions

## Why You Need This

Your requirements specify: "Add word to dictionary" and "(Optional) Ignore word for this session." These are the core actions your spell checker needs.

## What to Learn

### Action Types for Spell Checking

**1. Add to Dictionary (Permanent)**
- Word saved to dictionary file
- Persists across sessions
- Best for: proper nouns, technical terms, correctly spelled words

**2. Ignore for Session (Temporary)**
- Word remembered only until server restarts
- Good for: one-off cases user doesn't want permanently added

**3. Replace with Suggestion (If You Have Suggestions)**
- Replace misspelled word with correct spelling
- Requires suggestion data from engine

### Implementing "Add to Dictionary"

```c
JsonValue *create_add_word_action(const char *word, const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    char title[256];
    snprintf(title, sizeof(title), "Add '%s' to dictionary", word);
    json_set_string(action, "title", title);
    json_set_string(action, "kind", "quickfix");
    
    // Link to diagnostic
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    // Command
    JsonValue *command = json_object();
    json_set_string(command, "title", "Add to dictionary");
    json_set_string(command, "command", "wordlib.addWord");
    
    JsonValue *args = json_array();
    json_array_push_string(args, word);
    json_set(command, "arguments", args);
    json_set(action, "command", command);
    
    return action;
}
```

### Implementing "Ignore for Session"

```c
JsonValue *create_ignore_action(const char *word, const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    char title[256];
    snprintf(title, sizeof(title), "Ignore '%s' for this session", word);
    json_set_string(action, "title", title);
    json_set_string(action, "kind", "quickfix");
    
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    JsonValue *command = json_object();
    json_set_string(command, "title", "Ignore word");
    json_set_string(command, "command", "wordlib.ignoreWord");
    
    JsonValue *args = json_array();
    json_array_push_string(args, word);
    json_set(command, "arguments", args);
    json_set(action, "command", command);
    
    return action;
}
```

### Handling Execute Command

When user selects an action, editor sends:

```json
{
  "jsonrpc": "2.0",
  "id": 25,
  "method": "workspace/executeCommand",
  "params": {
    "command": "wordlib.addWord",
    "arguments": ["quikc"]
  }
}
```

Your handler:

```c
JsonValue *handle_execute_command(Server *server, const JsonValue *params) {
    const char *command = json_get_string(params, "command");
    const JsonValue *args = json_get(params, "arguments");
    
    if (strcmp(command, "wordlib.addWord") == 0) {
        return execute_add_word(server, args);
    }
    
    if (strcmp(command, "wordlib.ignoreWord") == 0) {
        return execute_ignore_word(server, args);
    }
    
    // Unknown command
    return json_null();
}

JsonValue *execute_add_word(Server *server, const JsonValue *args) {
    const char *word = json_get_string(json_array_get(args, 0), NULL);
    if (!word) return json_null();
    
    // Add to engine
    wordlib_add_word(server->engine, word);
    
    // Save dictionary
    if (server->workspace_dict_path) {
        wordlib_save(server->engine, server->workspace_dict_path);
    }
    
    // Re-check all open documents
    revalidate_all_documents(server);
    
    return json_null();  // Success
}

JsonValue *execute_ignore_word(Server *server, const JsonValue *args) {
    const char *word = json_get_string(json_array_get(args, 0), NULL);
    if (!word) return json_null();
    
    // Add to session-only ignore list
    ignore_list_add(server->session_ignore, word);
    
    // Re-check all open documents
    revalidate_all_documents(server);
    
    return json_null();
}
```

### Session Ignore List

```c
typedef struct {
    char **words;
    size_t count;
    size_t capacity;
} IgnoreList;

IgnoreList *ignore_list_create(void) {
    IgnoreList *list = calloc(1, sizeof(IgnoreList));
    list->capacity = 64;
    list->words = malloc(list->capacity * sizeof(char *));
    return list;
}

void ignore_list_add(IgnoreList *list, const char *word) {
    // Check if already ignored
    for (size_t i = 0; i < list->count; i++) {
        if (strcmp(list->words[i], word) == 0) {
            return;
        }
    }
    
    // Add word
    if (list->count >= list->capacity) {
        list->capacity *= 2;
        list->words = realloc(list->words, list->capacity * sizeof(char *));
    }
    list->words[list->count++] = strdup(word);
}

bool ignore_list_contains(IgnoreList *list, const char *word) {
    for (size_t i = 0; i < list->count; i++) {
        if (strcmp(list->words[i], word) == 0) {
            return true;
        }
    }
    return false;
}
```

### Updated Diagnostic Computation

Check both dictionary and ignore list:

```c
JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    // ... tokenization loop ...
    
    // Check if word is known
    if (!wordlib_contains(server->engine, word) &&
        !ignore_list_contains(server->session_ignore, word)) {
        // Create diagnostic
    }
    
    // ...
}
```

### Extracting Word from Diagnostic Message

```c
// Message format: "Unknown word: 'quikc'"
char *extract_word_from_message(const char *message) {
    // Find opening quote
    const char *start = strchr(message, '\'');
    if (!start) return NULL;
    start++;  // Skip quote
    
    // Find closing quote
    const char *end = strchr(start, '\'');
    if (!end) return NULL;
    
    size_t len = end - start;
    char *word = malloc(len + 1);
    memcpy(word, start, len);
    word[len] = '\0';
    
    return word;
}
```

### Alternative: Store Word in Diagnostic Data

```c
// When creating diagnostic, include word in data field
JsonValue *create_diagnostic_with_data(Range range, const char *word) {
    JsonValue *diag = create_diagnostic(range, word);
    
    // Add data field with word
    JsonValue *data = json_object();
    json_set_string(data, "word", word);
    json_set(diag, "data", data);
    
    return diag;
}

// When handling code action, extract from data
char *extract_word_from_diagnostic(const JsonValue *diagnostic) {
    const JsonValue *data = json_get(diagnostic, "data");
    if (data) {
        const char *word = json_get_string(data, "word");
        if (word) return strdup(word);
    }
    
    // Fallback to message parsing
    const char *message = json_get_string(diagnostic, "message");
    return extract_word_from_message(message);
}
```

## Where to Learn

1. **LSP Specification - executeCommand:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_executeCommand

2. **Similar implementations:**
   - How other spell checkers handle dictionary actions

## Practice Exercise

Test the complete action flow:

```c
void test_add_word_flow(void) {
    Server *server = test_server_create();
    
    // Initially, "quikc" is unknown
    assert(!wordlib_contains(server->engine, "quikc"));
    
    // Simulate executeCommand for addWord
    JsonValue *params = json_parse(
        "{\"command\":\"wordlib.addWord\","
        "\"arguments\":[\"quikc\"]}"
    );
    
    handle_execute_command(server, params);
    
    // Now "quikc" should be known
    assert(wordlib_contains(server->engine, "quikc"));
    
    json_free(params);
    test_server_destroy(server);
    printf("test_add_word_flow: PASS\n");
}
```

## Connection to Project

Dictionary management actions make your spell checker useful:

| Without Actions | With Actions |
|-----------------|--------------|
| See error, manually edit dictionary file | Click action, done |
| Restart server to reload | Instant update |
| Temporary words pollute dictionary | Session ignore keeps it clean |

This is the difference between a tool and a useful tool.
