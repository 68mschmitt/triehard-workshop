# Cursor Context

## Why You Need This

Your requirements say: "Trigger on word boundaries" and "Respect the client's cursor position." Understanding what the user is typing at the cursor position is essential for relevant completions.

## What to Learn

### The Context Problem

When user presses Ctrl+Space at position (0, 7) in "Hello wo":

```
Hello wo
       ^ cursor at character 7
```

You need to:
1. Find the word being typed ("wo")
2. Extract just the prefix ("wo")
3. Call engine with "wo"
4. Return ["word", "world", "work", ...]

### Finding Word Start

```c
// Scan backwards from cursor to find word start
size_t find_word_start(const char *text, size_t cursor_offset) {
    size_t start = cursor_offset;
    
    // Handle UTF-8: make sure we're at a character boundary
    while (start > 0) {
        // Move back one byte
        start--;
        
        // Skip UTF-8 continuation bytes (10xxxxxx)
        while (start > 0 && (text[start] & 0xC0) == 0x80) {
            start--;
        }
        
        // Check if this character is a word character
        if (!is_word_char_at(text, start)) {
            // Found non-word character, word starts after it
            // Move forward past the non-word char
            int char_len = utf8_char_len((unsigned char)text[start]);
            return start + char_len;
        }
    }
    
    return start;  // Word starts at beginning
}

bool is_word_char_at(const char *text, size_t pos) {
    unsigned char c = text[pos];
    
    // ASCII word characters
    if ((c >= 'a' && c <= 'z') ||
        (c >= 'A' && c <= 'Z') ||
        (c >= '0' && c <= '9') ||
        c == '\'') {
        return true;
    }
    
    // UTF-8 multi-byte characters (letters from other scripts)
    if (c >= 0x80) {
        return true;  // Simplified: treat all UTF-8 as word chars
    }
    
    return false;
}
```

### Extracting the Prefix

```c
typedef struct {
    char *prefix;
    size_t start_offset;
    size_t end_offset;  // Same as cursor position
} PrefixInfo;

PrefixInfo extract_prefix_info(const char *text, size_t text_len, size_t cursor_offset) {
    PrefixInfo info = {0};
    
    if (cursor_offset > text_len) {
        cursor_offset = text_len;
    }
    
    info.end_offset = cursor_offset;
    info.start_offset = find_word_start(text, cursor_offset);
    
    size_t prefix_len = cursor_offset - info.start_offset;
    
    if (prefix_len > 0) {
        info.prefix = malloc(prefix_len + 1);
        memcpy(info.prefix, text + info.start_offset, prefix_len);
        info.prefix[prefix_len] = '\0';
    }
    
    return info;
}
```

### Handling Different Cursor Positions

```
"The quick brown"
    ^            cursor at 4 → prefix = "q"
        ^        cursor at 8 → prefix = "quic"  
          ^      cursor at 10 → prefix = "quick" (end of word)
           ^     cursor at 11 → prefix = "" (between words)
               ^ cursor at 15 → prefix = "brown"
```

```c
// Edge case: cursor between words
if (cursor_offset > 0 && !is_word_char(text[cursor_offset - 1])) {
    // No prefix at cursor
    info.prefix = NULL;
}
```

### LSP Position to Document Offset

```c
// Convert LSP position (line, character) to byte offset
size_t lsp_position_to_offset(const char *text, uint32_t line, uint32_t character) {
    size_t offset = 0;
    uint32_t current_line = 0;
    
    // Skip to target line
    while (text[offset] && current_line < line) {
        if (text[offset] == '\n') {
            current_line++;
        }
        offset++;
    }
    
    // Now count UTF-16 code units to find character
    uint32_t utf16_count = 0;
    while (text[offset] && text[offset] != '\n' && utf16_count < character) {
        int char_len = utf8_char_len((unsigned char)text[offset]);
        int utf16_len = (char_len == 4) ? 2 : 1;
        
        utf16_count += utf16_len;
        offset += char_len;
    }
    
    return offset;
}
```

### Completion Context Modes

Different trigger kinds need different handling:

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // Get trigger context
    const JsonValue *context = json_get(params, "context");
    int trigger_kind = context ? json_get_int(context, "triggerKind") : 1;
    
    switch (trigger_kind) {
        case 1:  // Invoked (Ctrl+Space)
            // Always try to provide completions
            break;
            
        case 2:  // Trigger character
            // Only provide if we have a valid prefix
            // (for word completion, usually no trigger chars)
            break;
            
        case 3:  // Re-trigger (incomplete result)
            // User typing more, refine results
            break;
    }
    
    // ... rest of completion logic
}
```

### Context-Aware Filtering

Your requirements mention: "Avoid offering completions inside code blocks or URLs (if detectable)."

```c
bool should_offer_completions(const char *text, size_t offset) {
    // Simple heuristic: check if we're in a URL
    size_t start = (offset > 50) ? offset - 50 : 0;
    const char *context = text + start;
    
    if (strstr(context, "http://") || 
        strstr(context, "https://") ||
        strstr(context, "file://")) {
        // Might be in a URL, skip completion
        return false;
    }
    
    // For Markdown: check if in code block
    // This is simplistic - proper Markdown parsing would be needed
    // for accurate detection
    
    return true;
}
```

### Complete Handler with Context

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // Extract position
    const char *uri = json_get_string(
        json_get(params, "textDocument"), "uri");
    const JsonValue *pos = json_get(params, "position");
    uint32_t line = json_get_int(pos, "line");
    uint32_t character = json_get_int(pos, "character");
    
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return json_array();
    
    // Convert LSP position to byte offset
    size_t cursor_offset = lsp_position_to_offset(
        doc->text, line, character);
    
    // Check context
    if (!should_offer_completions(doc->text, cursor_offset)) {
        return json_array();
    }
    
    // Extract prefix
    PrefixInfo prefix_info = extract_prefix_info(
        doc->text, doc->text_len, cursor_offset);
    
    if (!prefix_info.prefix || strlen(prefix_info.prefix) == 0) {
        free(prefix_info.prefix);
        return json_array();
    }
    
    // Get completions
    const char *results[100];
    size_t count = wordlib_complete(server->engine, 
                                   prefix_info.prefix, results, 100);
    
    // Build response
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        JsonValue *item = create_completion_item(results[i], i);
        json_array_push(items, item);
    }
    
    free(prefix_info.prefix);
    return items;
}
```

## Where to Learn

1. **LSP Specification - CompletionContext:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#completionContext

2. **Test with different editors:**
   - Observe when they send completion requests

## Practice Exercise

Test cursor context extraction:

```c
void test_cursor_context(void) {
    const char *text = "Hello world";
    
    // Cursor at "wor" (character 9)
    PrefixInfo info = extract_prefix_info(text, strlen(text), 9);
    assert(strcmp(info.prefix, "wor") == 0);
    assert(info.start_offset == 6);
    assert(info.end_offset == 9);
    free(info.prefix);
    
    // Cursor between words (character 6)
    info = extract_prefix_info(text, strlen(text), 6);
    assert(info.prefix == NULL || strlen(info.prefix) == 0);
    free(info.prefix);
    
    // Cursor at end of word
    info = extract_prefix_info(text, strlen(text), 5);
    assert(strcmp(info.prefix, "Hello") == 0);
    free(info.prefix);
    
    printf("test_cursor_context: PASS\n");
}
```

## Connection to Project

Cursor context determines completion relevance:

| Cursor Position | Prefix | Completions |
|-----------------|--------|-------------|
| "hel|lo" | "hel" | hello, help, ... |
| "hello |" | "" | (none or all) |
| "hello| " | "hello" | (exact match or none) |

Get this right and completions feel natural.
