# Section 5 Project: Full Completion Provider

## Objective

Build a complete word completion system that responds to user typing with relevant suggestions from your dictionary, with proper context handling and performance limits.

## Requirements

1. **Handle textDocument/completion requests**
2. **Extract word prefix at cursor position**
3. **Call engine's prefix completion API**
4. **Return properly formatted completion items**
5. **Limit results for performance**
6. **Handle edge cases gracefully**

## Specification

### Completion API (`include/completion.h`)

```c
#ifndef COMPLETION_H
#define COMPLETION_H

#include "json.h"
#include "diagnostics.h"  // For Position type

// Forward declaration
typedef struct Server Server;

// Configuration
#define MAX_COMPLETION_RESULTS 50
#define MIN_PREFIX_LENGTH 1

// Prefix extraction result
typedef struct {
    char *prefix;
    size_t start_offset;
    size_t end_offset;
} PrefixInfo;

// Position conversion
size_t lsp_position_to_offset(const char *text, uint32_t line, uint32_t character);

// Prefix extraction
PrefixInfo extract_prefix_info(const char *text, size_t text_len, size_t cursor_offset);
void prefix_info_free(PrefixInfo *info);

// Completion item creation
JsonValue *create_completion_item(const char *word, size_t index);

// Main handler
JsonValue *handle_completion(Server *server, const JsonValue *params);

#endif
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── completion.h
│   └── ... (previous headers)
├── src/
│   ├── completion.c
│   └── ... (previous sources)
├── test/
│   └── test_completion.c
└── Makefile
```

## Implementation Guide

### Step 1: Position Conversion

```c
// src/completion.c
#include "completion.h"
#include "document_store.h"
#include <stdlib.h>
#include <string.h>

static int utf8_char_len(unsigned char c) {
    if ((c & 0x80) == 0x00) return 1;
    if ((c & 0xE0) == 0xC0) return 2;
    if ((c & 0xF0) == 0xE0) return 3;
    if ((c & 0xF8) == 0xF0) return 4;
    return 1;
}

size_t lsp_position_to_offset(const char *text, uint32_t line, uint32_t character) {
    if (!text) return 0;
    
    size_t offset = 0;
    uint32_t current_line = 0;
    
    // Find target line
    while (text[offset] && current_line < line) {
        if (text[offset] == '\n') {
            current_line++;
        }
        offset++;
    }
    
    // Count UTF-16 code units to character position
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

### Step 2: Prefix Extraction

```c
static bool is_word_char(unsigned char c) {
    return (c >= 'a' && c <= 'z') ||
           (c >= 'A' && c <= 'Z') ||
           (c >= '0' && c <= '9') ||
           c == '\'' ||
           c >= 0x80;  // UTF-8 multi-byte
}

static size_t find_word_start(const char *text, size_t offset) {
    size_t start = offset;
    
    while (start > 0) {
        // Move back one character (handle UTF-8)
        size_t prev = start - 1;
        while (prev > 0 && ((unsigned char)text[prev] & 0xC0) == 0x80) {
            prev--;
        }
        
        if (!is_word_char((unsigned char)text[prev])) {
            break;  // Found non-word character
        }
        
        start = prev;
    }
    
    return start;
}

PrefixInfo extract_prefix_info(const char *text, size_t text_len, size_t cursor_offset) {
    PrefixInfo info = { .prefix = NULL, .start_offset = 0, .end_offset = 0 };
    
    if (!text || cursor_offset > text_len) {
        return info;
    }
    
    info.end_offset = cursor_offset;
    info.start_offset = find_word_start(text, cursor_offset);
    
    size_t prefix_len = info.end_offset - info.start_offset;
    
    if (prefix_len > 0) {
        info.prefix = malloc(prefix_len + 1);
        if (info.prefix) {
            memcpy(info.prefix, text + info.start_offset, prefix_len);
            info.prefix[prefix_len] = '\0';
        }
    }
    
    return info;
}

void prefix_info_free(PrefixInfo *info) {
    if (info && info->prefix) {
        free(info->prefix);
        info->prefix = NULL;
    }
}
```

### Step 3: Completion Item Creation

```c
#define COMPLETION_KIND_TEXT 1

JsonValue *create_completion_item(const char *word, size_t index) {
    JsonValue *item = json_object();
    
    // Label (required)
    json_set_string(item, "label", word);
    
    // Kind
    json_set_int(item, "kind", COMPLETION_KIND_TEXT);
    
    // Sort text (preserves engine ordering)
    char sort_text[16];
    snprintf(sort_text, sizeof(sort_text), "%06zu", index);
    json_set_string(item, "sortText", sort_text);
    
    return item;
}
```

### Step 4: Main Handler

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // Extract document URI
    const JsonValue *text_doc = json_get(params, "textDocument");
    const char *uri = json_get_string(text_doc, "uri");
    
    // Extract position
    const JsonValue *pos = json_get(params, "position");
    uint32_t line = (uint32_t)json_get_int(pos, "line");
    uint32_t character = (uint32_t)json_get_int(pos, "character");
    
    // Get document
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) {
        return json_array();
    }
    
    // Convert position to offset
    size_t cursor_offset = lsp_position_to_offset(doc->text, line, character);
    
    // Extract prefix
    PrefixInfo prefix_info = extract_prefix_info(
        doc->text, doc->text_len, cursor_offset);
    
    // Check minimum length
    if (!prefix_info.prefix || strlen(prefix_info.prefix) < MIN_PREFIX_LENGTH) {
        prefix_info_free(&prefix_info);
        return json_array();
    }
    
    // Get completions from engine
    const char *results[MAX_COMPLETION_RESULTS + 1];
    size_t count = wordlib_complete(server->engine, prefix_info.prefix,
                                   results, MAX_COMPLETION_RESULTS + 1);
    
    // Check if incomplete
    bool is_incomplete = count > MAX_COMPLETION_RESULTS;
    if (is_incomplete) {
        count = MAX_COMPLETION_RESULTS;
    }
    
    // Build completion list
    JsonValue *list = json_object();
    json_set_bool(list, "isIncomplete", is_incomplete);
    
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        JsonValue *item = create_completion_item(results[i], i);
        json_array_push(items, item);
    }
    json_set(list, "items", items);
    
    prefix_info_free(&prefix_info);
    return list;
}
```

### Step 5: Register Handler

```c
// In your request dispatcher
JsonValue *dispatch_request(Server *server, int64_t id, 
                           const char *method, const JsonValue *params) {
    // ... existing handlers ...
    
    if (strcmp(method, "textDocument/completion") == 0) {
        return handle_completion(server, params);
    }
    
    // ... method not found ...
}
```

## Test Cases

```c
// test/test_completion.c
#include "completion.h"
#include "server.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>

void test_position_to_offset(void) {
    const char *text = "Hello\nWorld";
    
    // (0, 0) -> 0
    assert(lsp_position_to_offset(text, 0, 0) == 0);
    
    // (0, 5) -> 5
    assert(lsp_position_to_offset(text, 0, 5) == 5);
    
    // (1, 0) -> 6
    assert(lsp_position_to_offset(text, 1, 0) == 6);
    
    // (1, 3) -> 9
    assert(lsp_position_to_offset(text, 1, 3) == 9);
    
    printf("test_position_to_offset: PASS\n");
}

void test_prefix_extraction(void) {
    const char *text = "Hello world";
    
    // Cursor at "wor" (offset 9)
    PrefixInfo info = extract_prefix_info(text, strlen(text), 9);
    assert(info.prefix != NULL);
    assert(strcmp(info.prefix, "wor") == 0);
    assert(info.start_offset == 6);
    prefix_info_free(&info);
    
    // Cursor at start of word
    info = extract_prefix_info(text, strlen(text), 6);
    assert(info.prefix == NULL || strlen(info.prefix) == 0);
    prefix_info_free(&info);
    
    // Cursor at end of first word
    info = extract_prefix_info(text, strlen(text), 5);
    assert(strcmp(info.prefix, "Hello") == 0);
    prefix_info_free(&info);
    
    printf("test_prefix_extraction: PASS\n");
}

void test_completion_item(void) {
    JsonValue *item = create_completion_item("hello", 5);
    
    assert(strcmp(json_get_string(item, "label"), "hello") == 0);
    assert(json_get_int(item, "kind") == COMPLETION_KIND_TEXT);
    assert(strcmp(json_get_string(item, "sortText"), "000005") == 0);
    
    json_free(item);
    printf("test_completion_item: PASS\n");
}

void test_completion_handler(void) {
    Server *server = test_server_create();
    
    // Add words to dictionary
    wordlib_add_word(server->engine, "hello");
    wordlib_add_word(server->engine, "help");
    wordlib_add_word(server->engine, "helicopter");
    wordlib_add_word(server->engine, "world");
    
    // Open document
    document_store_open(server->documents, 
        "file:///test.txt", "plaintext", 1, "hel");
    
    // Create completion request
    JsonValue *params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":3}}"
    );
    
    JsonValue *result = handle_completion(server, params);
    
    // Should be a CompletionList
    JsonValue *items = json_get(result, "items");
    assert(items != NULL);
    assert(json_array_length(items) == 3);  // hello, help, helicopter
    
    // First item should be one of the "hel*" words
    JsonValue *first = json_array_get(items, 0);
    const char *label = json_get_string(first, "label");
    assert(strncmp(label, "hel", 3) == 0);
    
    json_free(params);
    json_free(result);
    test_server_destroy(server);
    
    printf("test_completion_handler: PASS\n");
}

void test_empty_prefix(void) {
    Server *server = test_server_create();
    wordlib_add_word(server->engine, "hello");
    
    // Document with cursor between words
    document_store_open(server->documents,
        "file:///test.txt", "plaintext", 1, "hello ");
    
    // Cursor at position 6 (after space)
    JsonValue *params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":6}}"
    );
    
    JsonValue *result = handle_completion(server, params);
    
    // Should return empty array
    assert(json_is_array(result) && json_array_length(result) == 0);
    
    json_free(params);
    json_free(result);
    test_server_destroy(server);
    
    printf("test_empty_prefix: PASS\n");
}

int main(void) {
    test_position_to_offset();
    test_prefix_extraction();
    test_completion_item();
    test_completion_handler();
    test_empty_prefix();
    printf("\nAll completion tests passed!\n");
    return 0;
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_completion && ./test_completion
test_position_to_offset: PASS
test_prefix_extraction: PASS
test_completion_item: PASS
test_completion_handler: PASS
test_empty_prefix: PASS
All completion tests passed!
```

### Integration Test with Editor
```lua
-- In Neovim
-- 1. Open a text file
-- 2. Type "hel"
-- 3. Press Ctrl+Space
-- 4. See completion menu with "hello", "help", etc.
```

### Performance
```bash
$ ./benchmark_completion
Prefix 'a': 50 results in 2.34 ms
Prefix 'ab': 50 results in 1.12 ms
Prefix 'abc': 23 results in 0.45 ms
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_completion
All heap blocks were freed -- no leaks are possible
```

## Stretch Goals

1. **Text edit support** - Include range to replace prefix
2. **Documentation** - Add word definitions if available
3. **Frequency ranking** - Prioritize common words
4. **Context filtering** - Skip completions in URLs/code blocks

## What You'll Learn

By completing this project:
- LSP position to byte offset conversion
- Word boundary detection
- Completion item formatting
- Performance-conscious result limiting

## Next Section Preview

Section 6 implements code actions, letting users add unknown words to the dictionary with a single click. This completes the spell check cycle: see problem → fix problem.
