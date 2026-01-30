# Full vs Incremental Sync

## Why You Need This

Your requirements say: "At minimum, the server shall support full document synchronization. Incremental sync support is recommended but optional."

Understanding the trade-offs helps you decide which to implement and how to advertise your capability.

## What to Learn

### Full Synchronization (TextDocumentSyncKind = 1)

Every change sends the entire document:

```json
{
  "method": "textDocument/didChange",
  "params": {
    "textDocument": { "uri": "file:///doc.txt", "version": 2 },
    "contentChanges": [
      { "text": "Complete document text after change..." }
    ]
  }
}
```

**Pros:**
- Simple to implement
- No state synchronization bugs
- Always have consistent view

**Cons:**
- More data over the wire
- Larger documents = more overhead
- Slightly higher latency

### Incremental Synchronization (TextDocumentSyncKind = 2)

Each change describes only what changed:

```json
{
  "method": "textDocument/didChange",
  "params": {
    "textDocument": { "uri": "file:///doc.txt", "version": 2 },
    "contentChanges": [
      {
        "range": {
          "start": { "line": 5, "character": 10 },
          "end": { "line": 5, "character": 15 }
        },
        "rangeLength": 5,
        "text": "new text"
      }
    ]
  }
}
```

**Pros:**
- Less data for small edits
- More efficient for large documents
- Lower latency

**Cons:**
- Complex to implement correctly
- Must apply changes precisely
- State can get out of sync

### Recommendation: Start with Full Sync

For a spell checker, full sync is fine:
- Documents are typically < 1MB
- Simplicity beats efficiency here
- You can optimize later if needed

### Declaring Your Capability

```c
// In initialize response
JsonValue *text_sync = json_object();
json_set_bool(text_sync, "openClose", true);  // didOpen/didClose
json_set_int(text_sync, "change", 1);          // 1 = Full, 2 = Incremental
json_set(capabilities, "textDocumentSync", text_sync);
```

### Handling Full Sync

```c
void handle_did_change_full(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    const char *uri = json_get_string(td, "uri");
    int64_t version = json_get_int(td, "version");
    
    const JsonValue *changes = json_get(params, "contentChanges");
    const JsonValue *change = json_array_get(changes, 0);
    const char *text = json_get_string(change, "text");
    
    document_store_update(server->documents, uri, version, text);
}
```

### Handling Incremental Sync (Optional)

If you want incremental:

```c
void handle_did_change_incremental(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    const char *uri = json_get_string(td, "uri");
    int64_t version = json_get_int(td, "version");
    
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    const JsonValue *changes = json_get(params, "contentChanges");
    size_t change_count = json_array_length(changes);
    
    // Apply each change in order
    for (size_t i = 0; i < change_count; i++) {
        const JsonValue *change = json_array_get(changes, i);
        
        const JsonValue *range = json_get(change, "range");
        const char *new_text = json_get_string(change, "text");
        
        if (range) {
            // Incremental change
            Position start = parse_position(json_get(range, "start"));
            Position end = parse_position(json_get(range, "end"));
            apply_edit(doc, start, end, new_text);
        } else {
            // Full change (fallback)
            free(doc->text);
            doc->text = strdup(new_text);
        }
    }
    
    doc->version = version;
}

void apply_edit(Document *doc, Position start, Position end, const char *new_text) {
    // Convert LSP positions to byte offsets
    size_t start_offset = position_to_offset(doc->text, start);
    size_t end_offset = position_to_offset(doc->text, end);
    
    size_t old_len = strlen(doc->text);
    size_t new_text_len = strlen(new_text);
    size_t delete_len = end_offset - start_offset;
    size_t new_len = old_len - delete_len + new_text_len;
    
    char *new_doc = malloc(new_len + 1);
    
    // Copy prefix
    memcpy(new_doc, doc->text, start_offset);
    // Copy new text
    memcpy(new_doc + start_offset, new_text, new_text_len);
    // Copy suffix
    memcpy(new_doc + start_offset + new_text_len, 
           doc->text + end_offset, 
           old_len - end_offset);
    new_doc[new_len] = '\0';
    
    free(doc->text);
    doc->text = new_doc;
}
```

### Position to Offset Conversion

```c
typedef struct {
    uint32_t line;
    uint32_t character;  // UTF-16 code units
} Position;

size_t position_to_offset(const char *text, Position pos) {
    size_t offset = 0;
    uint32_t line = 0;
    
    // Find the line
    while (line < pos.line && text[offset]) {
        if (text[offset] == '\n') {
            line++;
        }
        offset++;
    }
    
    // Now count characters (UTF-16 code units)
    uint32_t char_count = 0;
    while (char_count < pos.character && text[offset] && text[offset] != '\n') {
        // Get UTF-8 character length
        int char_len = utf8_char_length((unsigned char)text[offset]);
        
        // UTF-16 length
        if (char_len == 4) {
            char_count += 2;  // Surrogate pair
        } else {
            char_count += 1;
        }
        
        offset += char_len;
    }
    
    return offset;
}
```

### When Incremental Is Worth It

- Very large documents (>100KB)
- Frequent small edits
- Latency-sensitive applications

For a spell checker on typical text files, full sync is sufficient.

## Where to Learn

1. **LSP Specification - TextDocumentSyncKind:**
   - Details on sync kinds

2. **Real implementations:**
   - See how other servers handle incremental sync
   - `clangd` has robust incremental support

## Practice Exercise

Implement both sync modes and compare:

```c
void benchmark_sync(void) {
    // Large document (100KB of text)
    char *large_doc = generate_lorem_ipsum(100 * 1024);
    
    Document doc = { .text = strdup(large_doc) };
    
    // Full sync: replace entire text
    clock_t start = clock();
    for (int i = 0; i < 1000; i++) {
        free(doc.text);
        doc.text = strdup(large_doc);
    }
    printf("Full sync 1000x: %.2f ms\n", 
           (double)(clock() - start) / CLOCKS_PER_SEC * 1000);
    
    // Incremental: small edit
    start = clock();
    for (int i = 0; i < 1000; i++) {
        apply_edit(&doc, 
                   (Position){.line=50, .character=10},
                   (Position){.line=50, .character=15},
                   "test");
    }
    printf("Incremental sync 1000x: %.2f ms\n",
           (double)(clock() - start) / CLOCKS_PER_SEC * 1000);
    
    free(doc.text);
    free(large_doc);
}
```

## Connection to Project

For your spell checker:

1. **Start with full sync** - It's simpler and sufficient
2. **Track document versions** - Prevents applying stale changes
3. **Consider incremental later** - If performance requires it

The sync mode affects how you receive changes, but your spell checking logic stays the same—you always work with the current document text.
