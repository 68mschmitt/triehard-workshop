# Diagnostic Stability

## Why You Need This

Your requirements say: "Avoid flickering diagnostics on minor edits" and "Recompute diagnostics deterministically per document version."

Users hate flickering—diagnostics appearing and disappearing erratically during typing. Stable, predictable diagnostics build trust in your tool.

## What to Learn

### What Causes Flickering?

1. **Incomplete words:** User typing "hello" triggers diagnostics for "h", "he", "hel", "hell"
2. **Race conditions:** Multiple diagnostic publishes for same document version
3. **Stale diagnostics:** Publishing diagnostics computed from old document state
4. **Non-deterministic ordering:** Different diagnostic order on each publish

### Strategy 1: Include Document Version

Always include the document version:

```c
void publish_diagnostics(Server *server, const char *uri) {
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set_int(params, "version", doc->version);  // Critical!
    json_set(params, "diagnostics", compute_diagnostics(server, doc));
    
    // Send notification...
}
```

Editors can discard diagnostics if the version is stale.

### Strategy 2: Deterministic Diagnostic Order

Same input = same output order:

```c
// Sort diagnostics by position for consistent ordering
int compare_diagnostics(const void *a, const void *b) {
    const JsonValue *da = *(const JsonValue **)a;
    const JsonValue *db = *(const JsonValue **)b;
    
    JsonValue *ra = json_get(da, "range");
    JsonValue *rb = json_get(db, "range");
    
    int64_t line_a = json_get_int(json_get(ra, "start"), "line");
    int64_t line_b = json_get_int(json_get(rb, "start"), "line");
    
    if (line_a != line_b) return (line_a < line_b) ? -1 : 1;
    
    int64_t char_a = json_get_int(json_get(ra, "start"), "character");
    int64_t char_b = json_get_int(json_get(rb, "start"), "character");
    
    return (char_a < char_b) ? -1 : (char_a > char_b) ? 1 : 0;
}

JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    // ... compute diagnostics ...
    
    // Sort before returning
    sort_json_array(diagnostics, compare_diagnostics);
    
    return diagnostics;
}
```

### Strategy 3: Debouncing

Don't publish on every keystroke:

```c
typedef struct {
    char *uri;
    int64_t version;
    uint64_t scheduled_time_ms;
} PendingDiagnostic;

#define DEBOUNCE_MS 100  // Wait 100ms after last change

void schedule_diagnostics(Server *server, const char *uri, int64_t version) {
    uint64_t now = current_time_ms();
    
    // Find or create pending entry
    PendingDiagnostic *pending = find_pending(server, uri);
    if (!pending) {
        pending = create_pending(server, uri);
    }
    
    pending->version = version;
    pending->scheduled_time_ms = now + DEBOUNCE_MS;
}

void process_pending_diagnostics(Server *server) {
    uint64_t now = current_time_ms();
    
    for (size_t i = 0; i < server->pending_count; i++) {
        PendingDiagnostic *p = &server->pending[i];
        
        if (now >= p->scheduled_time_ms) {
            Document *doc = document_store_get(server->documents, p->uri);
            
            // Only publish if version matches (no new edits)
            if (doc && doc->version == p->version) {
                publish_diagnostics(server, p->uri);
            }
            
            remove_pending(server, i);
            i--;  // Adjust for removal
        }
    }
}
```

### Strategy 4: Minimal Diff

Only publish when diagnostics actually change:

```c
typedef struct {
    char *uri;
    char *last_diagnostics_hash;  // Hash of last published diagnostics
} DiagnosticCache;

bool should_publish(Server *server, const char *uri, JsonValue *diagnostics) {
    char *new_hash = compute_diagnostics_hash(diagnostics);
    
    DiagnosticCache *cache = find_cache(server, uri);
    if (cache && cache->last_diagnostics_hash) {
        if (strcmp(cache->last_diagnostics_hash, new_hash) == 0) {
            free(new_hash);
            return false;  // No change, skip publishing
        }
    }
    
    // Update cache
    if (!cache) cache = create_cache(server, uri);
    free(cache->last_diagnostics_hash);
    cache->last_diagnostics_hash = new_hash;
    
    return true;
}
```

### Strategy 5: Word Boundary Detection

Don't diagnose partial words:

```c
bool is_complete_word(const char *text, size_t text_len, size_t start, size_t end) {
    // Check if there's more word characters after
    if (end < text_len && is_word_char(text[end])) {
        return false;  // User still typing this word
    }
    return true;
}

JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    Tokenizer *tok = tokenizer_create(doc->text, doc->text_len);
    Token token;
    
    while (tokenizer_next(tok, &token)) {
        // Skip incomplete words at end of document
        if (!is_complete_word(doc->text, doc->text_len, 
                             token.start, token.end)) {
            continue;
        }
        
        char *word = extract_word(doc->text, token.start, token.end);
        
        if (!wordlib_contains(server->engine, word)) {
            // Create diagnostic...
        }
        
        free(word);
    }
    
    tokenizer_destroy(tok);
    return diagnostics;
}
```

### Practical Approach

For your project, start simple:

```c
// Simple, stable approach:
// 1. Always include version
// 2. Sort diagnostics by position
// 3. Compute immediately (debounce later if needed)

void publish_diagnostics(Server *server, const char *uri) {
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    // Compute sorted diagnostics
    JsonValue *diagnostics = compute_sorted_diagnostics(server, doc);
    
    // Build and send notification with version
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set_int(params, "version", doc->version);
    json_set(params, "diagnostics", diagnostics);
    
    send_notification(server, "textDocument/publishDiagnostics", params);
    json_free(params);
}
```

Add debouncing or caching only if you observe problems.

## Where to Learn

1. **User experience research:** Perceived latency and responsiveness
2. **Other LSP servers:** See how `clangd` handles diagnostic stability
3. **Editor documentation:** How Neovim/VS Code handle diagnostic updates

## Practice Exercise

Test diagnostic stability:

```c
void test_stability(void) {
    Server *server = create_test_server();
    
    // Simulate rapid typing
    const char *versions[] = {
        "h",
        "he", 
        "hel",
        "hell",
        "hello"
    };
    
    JsonValue *last_diagnostics = NULL;
    
    for (int i = 0; i < 5; i++) {
        update_document(server, "file:///test.txt", i + 1, versions[i]);
        
        // Capture diagnostics
        JsonValue *diags = capture_diagnostics(server, "file:///test.txt");
        
        // After "hello" (complete word), should have no diagnostics
        // During typing, might have diagnostics for partial word
        
        json_free(last_diagnostics);
        last_diagnostics = diags;
    }
    
    // Final "hello" should have 0 or 1 diagnostic (depending on dictionary)
    // Not 5 different diagnostic states!
    
    json_free(last_diagnostics);
    destroy_test_server(server);
}
```

## Connection to Project

Stable diagnostics make your spell checker feel professional:

| Bad UX | Good UX |
|--------|---------|
| Squiggles flicker during typing | Squiggles appear when word is complete |
| Diagnostics jump around | Diagnostics stay in predictable positions |
| Stale diagnostics shown | Diagnostics match current document |

Your users won't notice when it works well—they'll just have a smooth experience.
