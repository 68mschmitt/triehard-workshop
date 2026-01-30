# Completion Request Flow

## Why You Need This

Your requirements say: "Implement `textDocument/completion`" and "Use the engine's prefix completion API." Completions help users by suggesting words as they type. Understanding the request/response flow is essential.

## What to Learn

### The Completion Flow

```
User types "hel" → Editor sends completion request → Server extracts prefix
                                                           ↓
                                                   Engine: trie lookup
                                                           ↓
User sees menu ← Editor displays ← Server formats ← Results: [hello, help, ...]
```

### Completion Request

```json
{
  "jsonrpc": "2.0",
  "id": 15,
  "method": "textDocument/completion",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt"
    },
    "position": {
      "line": 5,
      "character": 10
    },
    "context": {
      "triggerKind": 1,
      "triggerCharacter": null
    }
  }
}
```

**Key fields:**
- `textDocument.uri`: Which document
- `position`: Cursor location (line/character)
- `context.triggerKind`: Why completion triggered
  - 1: Invoked manually (Ctrl+Space)
  - 2: Trigger character typed
  - 3: Re-trigger (incomplete)

### Completion Response

```json
{
  "jsonrpc": "2.0",
  "id": 15,
  "result": [
    {
      "label": "hello",
      "kind": 1,
      "detail": "from dictionary"
    },
    {
      "label": "help",
      "kind": 1
    },
    {
      "label": "helicopter",
      "kind": 1
    }
  ]
}
```

Or as a CompletionList:

```json
{
  "jsonrpc": "2.0",
  "id": 15,
  "result": {
    "isIncomplete": false,
    "items": [
      { "label": "hello", "kind": 1 },
      { "label": "help", "kind": 1 }
    ]
  }
}
```

### Handler Structure

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // 1. Extract document and position
    const char *uri = json_get_string(
        json_get(params, "textDocument"), "uri");
    const JsonValue *pos = json_get(params, "position");
    uint32_t line = json_get_int(pos, "line");
    uint32_t character = json_get_int(pos, "character");
    
    // 2. Get document
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) {
        return json_array();  // Empty result
    }
    
    // 3. Convert position to byte offset
    size_t offset = position_to_offset(doc->text, line, character);
    
    // 4. Extract prefix at cursor
    char *prefix = extract_prefix(doc->text, offset);
    if (!prefix || strlen(prefix) == 0) {
        free(prefix);
        return json_array();
    }
    
    // 5. Get completions from engine
    const char *results[100];
    size_t count = wordlib_complete(server->engine, prefix, results, 100);
    
    // 6. Build response
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        JsonValue *item = json_object();
        json_set_string(item, "label", results[i]);
        json_set_int(item, "kind", 1);  // Text
        json_array_push(items, item);
    }
    
    free(prefix);
    return items;
}
```

### Position to Byte Offset (Reverse Conversion)

```c
// Convert LSP position to byte offset
size_t position_to_offset(const char *text, uint32_t line, uint32_t character) {
    const unsigned char *p = (const unsigned char *)text;
    size_t offset = 0;
    uint32_t current_line = 0;
    
    // Find the line
    while (current_line < line && p[offset]) {
        if (p[offset] == '\n') {
            current_line++;
        }
        offset++;
    }
    
    // Count characters on this line (UTF-16 code units)
    uint32_t char_count = 0;
    while (char_count < character && p[offset] && p[offset] != '\n') {
        int char_len = utf8_char_len(p[offset]);
        int utf16_len = (char_len == 4) ? 2 : 1;
        
        char_count += utf16_len;
        offset += char_len;
    }
    
    return offset;
}
```

### Extracting the Prefix

```c
char *extract_prefix(const char *text, size_t offset) {
    // Find word start (scan backwards)
    size_t start = offset;
    while (start > 0 && is_word_char(text[start - 1])) {
        // Handle UTF-8: find the start of this character
        start--;
        while (start > 0 && (text[start] & 0xC0) == 0x80) {
            start--;  // Skip continuation bytes
        }
    }
    
    // Extract prefix
    size_t len = offset - start;
    if (len == 0) return NULL;
    
    char *prefix = malloc(len + 1);
    memcpy(prefix, text + start, len);
    prefix[len] = '\0';
    
    return prefix;
}
```

### Trigger Characters

Your server capabilities can specify trigger characters:

```c
// In initialize response
JsonValue *completion = json_object();
JsonValue *triggers = json_array();
// Add trigger characters if needed:
// json_array_push_string(triggers, ".");
json_set(completion, "triggerCharacters", triggers);
```

For word completion, no trigger characters needed—completion triggers on word boundaries or manually.

## Where to Learn

1. **LSP Specification - Completion:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_completion

2. **Test with your editor:**
   - Watch completion requests in LSP logs

## Practice Exercise

Test the complete flow:

```c
void test_completion_request(void) {
    Server *server = test_server_create();
    
    // Add words to dictionary
    wordlib_add_word(server->engine, "hello");
    wordlib_add_word(server->engine, "help");
    wordlib_add_word(server->engine, "helicopter");
    wordlib_add_word(server->engine, "world");
    
    // Open document
    document_store_open(server->documents, 
        "file:///test.txt", "plaintext", 1, "hel");
    
    // Simulate completion request at position (0, 3)
    JsonValue *params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":3}}"
    );
    
    JsonValue *result = handle_completion(server, params);
    
    // Should have 3 results: hello, help, helicopter
    assert(json_array_length(result) == 3);
    
    json_free(params);
    json_free(result);
    test_server_destroy(server);
}
```

## Connection to Project

Completions make your dictionary interactive:

```
Without completion: User types "hel" + full word from memory
With completion:    User types "hel" + sees suggestions + picks one
```

The engine's trie does the heavy lifting; you just wire it up.
