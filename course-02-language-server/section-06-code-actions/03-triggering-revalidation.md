# Triggering Revalidation

## Why You Need This

Your requirements say: "Code actions shall call the appropriate engine API, trigger a recheck of affected documents."

When a user adds a word to the dictionary, existing diagnostics for that word should disappear. Without revalidation, stale diagnostics confuse users.

## What to Learn

### The Revalidation Problem

```
Before "add word":
- Document shows: "Unknown word: 'quikc'"

User clicks "Add 'quikc' to dictionary"

Without revalidation:
- Diagnostic still visible (stale!)
- User confused

With revalidation:
- Diagnostic disappears immediately
- User sees the fix worked
```

### Revalidation Strategies

**1. Revalidate Single Document**
- After action, re-check the document where action occurred
- Fast, but might miss same word in other documents

**2. Revalidate All Open Documents**
- Re-check every open document
- Slower, but consistent
- Recommended for dictionary changes

**3. Selective Revalidation**
- Track which documents contain the affected word
- Only revalidate those
- More complex, best performance

### Simple Approach: Revalidate All

```c
void revalidate_all_documents(Server *server) {
    // Re-publish diagnostics for every open document
    document_store_foreach(server->documents, 
                          revalidate_document, server);
}

static void revalidate_document(Document *doc, void *userdata) {
    Server *server = (Server *)userdata;
    publish_diagnostics(server, doc->uri);
}
```

### When to Revalidate

```c
JsonValue *execute_add_word(Server *server, const JsonValue *args) {
    const char *word = json_get_string(json_array_get(args, 0), NULL);
    if (!word) return json_null();
    
    // 1. Add to dictionary
    wordlib_add_word(server->engine, word);
    
    // 2. Persist (optional, but good practice)
    persist_dictionary(server);
    
    // 3. Revalidate
    revalidate_all_documents(server);
    
    return json_null();
}

JsonValue *execute_ignore_word(Server *server, const JsonValue *args) {
    const char *word = json_get_string(json_array_get(args, 0), NULL);
    if (!word) return json_null();
    
    // 1. Add to ignore list
    ignore_list_add(server->session_ignore, word);
    
    // 2. Revalidate
    revalidate_all_documents(server);
    
    return json_null();
}
```

### Optimized Selective Revalidation

For larger numbers of open documents:

```c
typedef struct {
    char **uris;
    size_t count;
    size_t capacity;
} UriList;

// Track which documents might need revalidation
void revalidate_documents_with_word(Server *server, const char *word) {
    UriList *affected = uri_list_create();
    
    // Find documents containing this word
    document_store_foreach(server->documents, 
                          check_for_word, 
                          &(struct { const char *word; UriList *list; }){word, affected});
    
    // Revalidate only affected documents
    for (size_t i = 0; i < affected->count; i++) {
        publish_diagnostics(server, affected->uris[i]);
    }
    
    uri_list_destroy(affected);
}

static void check_for_word(Document *doc, void *userdata) {
    struct { const char *word; UriList *list; } *ctx = userdata;
    
    // Simple check: does document contain this word?
    if (strstr(doc->text, ctx->word)) {
        uri_list_add(ctx->list, doc->uri);
    }
}
```

### Preventing Duplicate Notifications

If user triggers multiple actions quickly, avoid redundant work:

```c
typedef struct {
    bool pending;
    uint64_t scheduled_time;
} RevalidationState;

void schedule_revalidation(Server *server) {
    server->revalidation.pending = true;
    server->revalidation.scheduled_time = current_time_ms() + 50;  // 50ms debounce
}

void process_pending_revalidation(Server *server) {
    if (!server->revalidation.pending) return;
    
    if (current_time_ms() >= server->revalidation.scheduled_time) {
        server->revalidation.pending = false;
        revalidate_all_documents(server);
    }
}
```

### Immediate vs Deferred

For simplicity, use immediate revalidation:

```c
JsonValue *execute_add_word(Server *server, const JsonValue *args) {
    // ... add word ...
    
    // Immediate revalidation - simple and correct
    revalidate_all_documents(server);
    
    return json_null();
}
```

Add debouncing only if performance becomes an issue with many open documents.

### Testing Revalidation

```c
void test_revalidation(void) {
    Server *server = test_server_create();
    
    // Open document with unknown word
    document_store_open(server->documents,
        "file:///test.txt", "plaintext", 1, "Hello quikc world");
    
    // Initial diagnostics: should have one for "quikc"
    JsonValue *diags1 = compute_diagnostics(server, 
        document_store_get(server->documents, "file:///test.txt"));
    assert(json_array_length(diags1) == 1);
    json_free(diags1);
    
    // Add word to dictionary
    wordlib_add_word(server->engine, "quikc");
    
    // After revalidation: should have no diagnostics
    JsonValue *diags2 = compute_diagnostics(server,
        document_store_get(server->documents, "file:///test.txt"));
    assert(json_array_length(diags2) == 0);
    json_free(diags2);
    
    test_server_destroy(server);
    printf("test_revalidation: PASS\n");
}
```

### Full Action → Revalidation Flow

```
User clicks "Add 'quikc' to dictionary"
           │
           ▼
executeCommand request arrives
           │
           ▼
execute_add_word(server, args)
           │
           ├── wordlib_add_word(engine, "quikc")
           │
           ├── wordlib_save(engine, dict_path)  // Persist
           │
           └── revalidate_all_documents(server)
                      │
                      ├── For each open document:
                      │       publish_diagnostics(server, uri)
                      │           │
                      │           ├── compute_diagnostics()
                      │           │   (now "quikc" is known)
                      │           │
                      │           └── Send notification
                      │
                      └── Done
           │
           ▼
Return response to client
           │
           ▼
Editor receives updated diagnostics
           │
           ▼
"quikc" squiggle disappears
```

## Where to Learn

1. **Other LSP servers:** How they handle revalidation
2. **Performance profiling:** Measure revalidation time

## Practice Exercise

Verify end-to-end:

```c
void test_full_action_flow(void) {
    Server *server = test_server_with_transport();
    
    // Open document
    send_did_open(server, "file:///test.txt", "Hello quikc");
    
    // Capture initial diagnostics
    JsonValue *initial = capture_notification(server, "publishDiagnostics");
    assert(json_array_length(json_get(initial, "diagnostics")) == 1);
    
    // Send executeCommand
    send_execute_command(server, "wordlib.addWord", "quikc");
    
    // Capture updated diagnostics
    JsonValue *updated = capture_notification(server, "publishDiagnostics");
    assert(json_array_length(json_get(updated, "diagnostics")) == 0);
    
    cleanup(server, initial, updated);
    printf("test_full_action_flow: PASS\n");
}
```

## Connection to Project

Revalidation makes actions feel instant:

| Action | Without Revalidation | With Revalidation |
|--------|---------------------|-------------------|
| Add word | Diagnostic stays until re-edit | Diagnostic disappears |
| Ignore word | User still sees squiggle | Squiggle gone |

This responsiveness is what makes a tool feel polished.
