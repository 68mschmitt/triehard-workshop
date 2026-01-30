# Section 4 Project: Complete Diagnostics Provider

## Objective

Build a complete diagnostics system that identifies unknown words and reports them to the editor with accurate positions. This is the core feature that makes your spell checker visible to users.

## Requirements

1. **Tokenize document text** to find words
2. **Check words against engine** dictionary
3. **Convert byte spans to LSP positions** accurately
4. **Publish diagnostics** on document open/change
5. **Clear diagnostics** on document close
6. **Handle UTF-8 text** correctly

## Specification

### Diagnostics API (`include/diagnostics.h`)

```c
#ifndef DIAGNOSTICS_H
#define DIAGNOSTICS_H

#include "json.h"
#include "document_store.h"

// Forward declaration
typedef struct Server Server;

// Position types
typedef struct {
    uint32_t line;
    uint32_t character;
} Position;

typedef struct {
    Position start;
    Position end;
} Range;

// Coordinate conversion
Position byte_to_position(const char *text, size_t offset);
Range byte_span_to_range(const char *text, size_t start, size_t end);

// Diagnostic creation
JsonValue *create_diagnostic(Range range, const char *word);

// Diagnostic publishing
void publish_diagnostics(Server *server, const char *uri);
void clear_diagnostics(Server *server, const char *uri);

// Compute diagnostics for a document
JsonValue *compute_diagnostics(Server *server, Document *doc);

#endif
```

### Server Structure Update

```c
// server.h
typedef struct Server {
    Transport *transport;
    DocumentStore *documents;
    WordLib *engine;           // From Course 1
    ServerState state;
    char *workspace_root;
    // ... other fields
} Server;
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── transport.h
│   ├── json.h
│   ├── document_store.h
│   ├── uri.h
│   ├── diagnostics.h
│   └── server.h
├── src/
│   ├── transport.c
│   ├── json.c
│   ├── cJSON.c
│   ├── document_store.c
│   ├── uri.c
│   ├── diagnostics.c
│   ├── handlers.c
│   └── server.c
├── test/
│   ├── test_diagnostics.c
│   └── test_position.c
└── Makefile
```

## Implementation Guide

### Step 1: Position Conversion

```c
// src/diagnostics.c
#include "diagnostics.h"
#include <string.h>
#include <stdio.h>

// UTF-8 character length from first byte
static int utf8_char_len(unsigned char c) {
    if ((c & 0x80) == 0x00) return 1;
    if ((c & 0xE0) == 0xC0) return 2;
    if ((c & 0xF0) == 0xE0) return 3;
    if ((c & 0xF8) == 0xF0) return 4;
    return 1;  // Invalid, treat as single byte
}

Position byte_to_position(const char *text, size_t offset) {
    Position pos = { .line = 0, .character = 0 };
    const unsigned char *p = (const unsigned char *)text;
    size_t i = 0;
    
    while (i < offset && p[i]) {
        if (p[i] == '\n') {
            pos.line++;
            pos.character = 0;
            i++;
        } else {
            int char_len = utf8_char_len(p[i]);
            // UTF-16 code units: 4-byte UTF-8 = 2, otherwise 1
            pos.character += (char_len == 4) ? 2 : 1;
            i += char_len;
        }
    }
    
    return pos;
}

Range byte_span_to_range(const char *text, size_t start, size_t end) {
    return (Range){
        .start = byte_to_position(text, start),
        .end = byte_to_position(text, end)
    };
}
```

### Step 2: Diagnostic Creation

```c
// Diagnostic severity
#define DIAG_INFORMATION 3

JsonValue *create_diagnostic(Range range, const char *word) {
    JsonValue *diag = json_object();
    
    // Build range
    JsonValue *range_obj = json_object();
    
    JsonValue *start = json_object();
    json_set_int(start, "line", range.start.line);
    json_set_int(start, "character", range.start.character);
    
    JsonValue *end = json_object();
    json_set_int(end, "line", range.end.line);
    json_set_int(end, "character", range.end.character);
    
    json_set(range_obj, "start", start);
    json_set(range_obj, "end", end);
    json_set(diag, "range", range_obj);
    
    // Set severity
    json_set_int(diag, "severity", DIAG_INFORMATION);
    
    // Set code and source
    json_set_string(diag, "code", "wordlib.unknown");
    json_set_string(diag, "source", "wordlib");
    
    // Set message
    char message[256];
    snprintf(message, sizeof(message), "Unknown word: '%s'", word);
    json_set_string(diag, "message", message);
    
    return diag;
}
```

### Step 3: Computing Diagnostics

```c
// Simple word character check
static bool is_word_char(char c) {
    return (c >= 'a' && c <= 'z') ||
           (c >= 'A' && c <= 'Z') ||
           (c >= '0' && c <= '9') ||
           c == '\'' ||  // contractions
           (unsigned char)c >= 0x80;  // UTF-8 continuation/start
}

// Simple tokenizer for words
typedef struct {
    const char *text;
    size_t len;
    size_t pos;
} SimpleTokenizer;

static bool next_word(SimpleTokenizer *tok, size_t *start, size_t *end) {
    // Skip non-word characters
    while (tok->pos < tok->len && !is_word_char(tok->text[tok->pos])) {
        tok->pos++;
    }
    
    if (tok->pos >= tok->len) return false;
    
    *start = tok->pos;
    
    // Consume word characters
    while (tok->pos < tok->len && is_word_char(tok->text[tok->pos])) {
        // Handle UTF-8 multi-byte characters
        int char_len = utf8_char_len((unsigned char)tok->text[tok->pos]);
        tok->pos += char_len;
    }
    
    *end = tok->pos;
    return *end > *start;
}

JsonValue *compute_diagnostics(Server *server, Document *doc) {
    JsonValue *diagnostics = json_array();
    
    SimpleTokenizer tok = { 
        .text = doc->text, 
        .len = doc->text_len, 
        .pos = 0 
    };
    
    size_t start, end;
    while (next_word(&tok, &start, &end)) {
        // Extract word
        size_t word_len = end - start;
        char *word = malloc(word_len + 1);
        memcpy(word, doc->text + start, word_len);
        word[word_len] = '\0';
        
        // Check against dictionary
        if (!wordlib_contains(server->engine, word)) {
            Range range = byte_span_to_range(doc->text, start, end);
            JsonValue *diag = create_diagnostic(range, word);
            json_array_push(diagnostics, diag);
        }
        
        free(word);
    }
    
    return diagnostics;
}
```

### Step 4: Publishing

```c
void publish_diagnostics(Server *server, const char *uri) {
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return;
    
    // Compute diagnostics
    JsonValue *diagnostics = compute_diagnostics(server, doc);
    
    // Build notification params
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set_int(params, "version", doc->version);
    json_set(params, "diagnostics", diagnostics);
    
    // Build full notification
    JsonValue *notification = json_object();
    json_set_string(notification, "jsonrpc", "2.0");
    json_set_string(notification, "method", "textDocument/publishDiagnostics");
    json_set(notification, "params", params);
    
    // Send
    char *json = json_stringify(notification);
    transport_write(server->transport, json);
    free(json);
    
    json_free(notification);
}

void clear_diagnostics(Server *server, const char *uri) {
    JsonValue *params = json_object();
    json_set_string(params, "uri", uri);
    json_set(params, "diagnostics", json_array());  // Empty
    
    JsonValue *notification = json_object();
    json_set_string(notification, "jsonrpc", "2.0");
    json_set_string(notification, "method", "textDocument/publishDiagnostics");
    json_set(notification, "params", params);
    
    char *json = json_stringify(notification);
    transport_write(server->transport, json);
    free(json);
    
    json_free(notification);
}
```

### Step 5: Update Handlers

```c
// In handlers.c
void handle_did_open(Server *server, const JsonValue *params) {
    // ... existing open logic ...
    
    // Trigger diagnostics
    publish_diagnostics(server, uri);
}

void handle_did_change(Server *server, const JsonValue *params) {
    // ... existing change logic ...
    
    // Re-compute diagnostics
    publish_diagnostics(server, uri);
}

void handle_did_close(Server *server, const JsonValue *params) {
    const char *uri = /* extract */;
    
    // Clear diagnostics first
    clear_diagnostics(server, uri);
    
    // Then close document
    document_store_close(server->documents, uri);
}
```

## Test Cases

### Position Conversion Tests

```c
// test/test_position.c
#include "diagnostics.h"
#include <assert.h>
#include <stdio.h>

void test_ascii_position(void) {
    const char *text = "Hello\nWorld";
    
    Position p = byte_to_position(text, 0);  // 'H'
    assert(p.line == 0 && p.character == 0);
    
    p = byte_to_position(text, 5);  // '\n'
    assert(p.line == 0 && p.character == 5);
    
    p = byte_to_position(text, 6);  // 'W'
    assert(p.line == 1 && p.character == 0);
    
    p = byte_to_position(text, 11);  // end
    assert(p.line == 1 && p.character == 5);
    
    printf("test_ascii_position: PASS\n");
}

void test_utf8_position(void) {
    // "café" - é is 2 bytes in UTF-8
    const char *text = "caf\xc3\xa9";  // café
    
    Position p = byte_to_position(text, 0);  // 'c'
    assert(p.line == 0 && p.character == 0);
    
    p = byte_to_position(text, 3);  // start of 'é'
    assert(p.line == 0 && p.character == 3);
    
    p = byte_to_position(text, 5);  // end (5 bytes total)
    assert(p.line == 0 && p.character == 4);  // 4 characters
    
    printf("test_utf8_position: PASS\n");
}

void test_range_conversion(void) {
    const char *text = "The quikc brown fox";
    
    // "quikc" is bytes 4-9
    Range r = byte_span_to_range(text, 4, 9);
    
    assert(r.start.line == 0 && r.start.character == 4);
    assert(r.end.line == 0 && r.end.character == 9);
    
    printf("test_range_conversion: PASS\n");
}

int main(void) {
    test_ascii_position();
    test_utf8_position();
    test_range_conversion();
    printf("\nAll position tests passed!\n");
    return 0;
}
```

### Diagnostics Tests

```c
// test/test_diagnostics.c
#include "diagnostics.h"
#include "server.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>

void test_diagnostic_creation(void) {
    Range range = {
        .start = { .line = 5, .character = 10 },
        .end = { .line = 5, .character = 14 }
    };
    
    JsonValue *diag = create_diagnostic(range, "helo");
    
    // Verify structure
    assert(json_get_int(diag, "severity") == 3);
    assert(strcmp(json_get_string(diag, "source"), "wordlib") == 0);
    assert(strstr(json_get_string(diag, "message"), "helo") != NULL);
    
    JsonValue *r = json_get(diag, "range");
    assert(json_get_int(json_get(r, "start"), "line") == 5);
    
    json_free(diag);
    printf("test_diagnostic_creation: PASS\n");
}

void test_compute_diagnostics(void) {
    Server *server = test_server_create();
    
    // Add some words to dictionary
    wordlib_add_word(server->engine, "hello");
    wordlib_add_word(server->engine, "world");
    
    // Create test document
    Document doc = {
        .text = "hello wrold",  // wrold is misspelled
        .text_len = 11,
        .version = 1
    };
    
    JsonValue *diagnostics = compute_diagnostics(server, &doc);
    
    // Should have one diagnostic for "wrold"
    assert(json_array_length(diagnostics) == 1);
    
    JsonValue *diag = json_array_get(diagnostics, 0);
    assert(strstr(json_get_string(diag, "message"), "wrold") != NULL);
    
    json_free(diagnostics);
    test_server_destroy(server);
    printf("test_compute_diagnostics: PASS\n");
}

void test_no_diagnostics_for_known_words(void) {
    Server *server = test_server_create();
    
    wordlib_add_word(server->engine, "hello");
    wordlib_add_word(server->engine, "world");
    
    Document doc = {
        .text = "hello world",
        .text_len = 11,
        .version = 1
    };
    
    JsonValue *diagnostics = compute_diagnostics(server, &doc);
    
    // No diagnostics - all words known
    assert(json_array_length(diagnostics) == 0);
    
    json_free(diagnostics);
    test_server_destroy(server);
    printf("test_no_diagnostics_for_known_words: PASS\n");
}

int main(void) {
    test_diagnostic_creation();
    test_compute_diagnostics();
    test_no_diagnostics_for_known_words();
    printf("\nAll diagnostic tests passed!\n");
    return 0;
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test
./test_position
test_ascii_position: PASS
test_utf8_position: PASS
test_range_conversion: PASS
All position tests passed!

./test_diagnostics
test_diagnostic_creation: PASS
test_compute_diagnostics: PASS
test_no_diagnostics_for_known_words: PASS
All diagnostic tests passed!
```

### Integration Test with Editor

```lua
-- Neovim test
vim.lsp.start({
  name = 'wordlib',
  cmd = { './wordlib-lsp' },
  filetypes = { 'text', 'markdown' },
})

-- Open a file with unknown words
-- Verify squiggles appear under unknown words
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_diagnostics
All heap blocks were freed -- no leaks are possible
```

## Stretch Goals

1. **Debounced publishing** - Wait 100ms after changes before publishing
2. **Diagnostic caching** - Don't republish if diagnostics unchanged
3. **Configurable severity** - Let users choose Warning vs Information
4. **Ignore patterns** - Skip words matching certain patterns (URLs, etc.)

## What You'll Learn

By completing this project:
- Coordinate system conversion (byte ↔ line/char)
- UTF-8/UTF-16 encoding handling
- LSP notification publishing
- End-to-end spell checking integration

## Next Section Preview

Section 5 implements completions. Users will see word suggestions as they type, using the same engine prefix completion you built in Course 1.
