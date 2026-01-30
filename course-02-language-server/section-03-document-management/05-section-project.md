# Section 3 Project: Document Store Implementation

## Objective

Build a complete document management system that tracks open documents, handles lifecycle events, and provides the text access needed by diagnostics and completion features.

## Requirements

1. **Track all open documents** with URI, version, language, and text
2. **Handle lifecycle notifications** (didOpen, didChange, didClose)
3. **Support full document synchronization**
4. **Validate document versions** to handle out-of-order updates
5. **Provide text access methods** for spell checking and completion

## Specification

### Document Store API (`include/document_store.h`)

```c
#ifndef DOCUMENT_STORE_H
#define DOCUMENT_STORE_H

#include <stdbool.h>
#include <stdint.h>
#include <stddef.h>

typedef struct DocumentStore DocumentStore;

typedef struct {
    char *uri;
    char *language_id;
    int64_t version;
    char *text;
    size_t text_len;
} Document;

// Lifecycle
DocumentStore *document_store_create(void);
void document_store_destroy(DocumentStore *store);

// Document operations
bool document_store_open(DocumentStore *store,
                        const char *uri,
                        const char *language_id,
                        int64_t version,
                        const char *text);

bool document_store_update(DocumentStore *store,
                          const char *uri,
                          int64_t version,
                          const char *text);

bool document_store_close(DocumentStore *store, const char *uri);

// Access
Document *document_store_get(DocumentStore *store, const char *uri);
size_t document_store_count(const DocumentStore *store);

// Iteration
typedef void (*DocumentCallback)(Document *doc, void *userdata);
void document_store_foreach(DocumentStore *store, 
                           DocumentCallback cb, 
                           void *userdata);

#endif
```

### URI Utilities (`include/uri.h`)

```c
#ifndef URI_H
#define URI_H

#include <stdbool.h>

// Convert file URI to path (caller frees result)
char *uri_to_path(const char *uri);

// Convert path to file URI (caller frees result)
char *path_to_uri(const char *path);

// Check if URI is a valid file URI
bool uri_is_file(const char *uri);

// Compare URIs (handles encoding differences)
bool uri_equals(const char *uri1, const char *uri2);

#endif
```

### Document Handler (`include/handlers.h`)

```c
#ifndef HANDLERS_H
#define HANDLERS_H

#include "json.h"

// Forward declaration
typedef struct Server Server;

// Document lifecycle handlers
void handle_did_open(Server *server, const JsonValue *params);
void handle_did_change(Server *server, const JsonValue *params);
void handle_did_close(Server *server, const JsonValue *params);

#endif
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── transport.h
│   ├── json.h
│   ├── document_store.h
│   ├── uri.h
│   └── handlers.h
├── src/
│   ├── transport.c
│   ├── json.c
│   ├── cJSON.c
│   ├── document_store.c
│   ├── uri.c
│   └── handlers.c
├── test/
│   ├── test_document_store.c
│   ├── test_uri.c
│   └── test_handlers.c
└── Makefile
```

## Implementation Guide

### Step 1: Implement URI Handling

```c
// src/uri.c
#include "uri.h"
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <stdio.h>

char *uri_to_path(const char *uri) {
    if (!uri || strncmp(uri, "file:///", 8) != 0) {
        return NULL;
    }
    
    // Skip file:// (7 chars), keep the leading /
    const char *encoded = uri + 7;
    size_t len = strlen(encoded);
    char *path = malloc(len + 1);
    
    // Decode percent-encoding
    size_t j = 0;
    for (size_t i = 0; i < len; i++) {
        if (encoded[i] == '%' && i + 2 < len &&
            isxdigit(encoded[i+1]) && isxdigit(encoded[i+2])) {
            char hex[3] = { encoded[i+1], encoded[i+2], '\0' };
            path[j++] = (char)strtol(hex, NULL, 16);
            i += 2;
        } else {
            path[j++] = encoded[i];
        }
    }
    path[j] = '\0';
    
    return path;
}

char *path_to_uri(const char *path) {
    if (!path || path[0] != '/') {
        return NULL;
    }
    
    // Calculate max encoded length
    size_t path_len = strlen(path);
    char *uri = malloc(7 + path_len * 3 + 1);  // file:// + encoded path
    strcpy(uri, "file://");
    
    size_t j = 7;
    for (size_t i = 0; i < path_len; i++) {
        unsigned char c = path[i];
        
        // Characters that don't need encoding
        if (isalnum(c) || c == '/' || c == '-' || c == '_' || 
            c == '.' || c == '~') {
            uri[j++] = c;
        } else {
            sprintf(uri + j, "%%%02X", c);
            j += 3;
        }
    }
    uri[j] = '\0';
    
    return uri;
}

bool uri_is_file(const char *uri) {
    return uri && strncmp(uri, "file://", 7) == 0;
}

bool uri_equals(const char *uri1, const char *uri2) {
    if (!uri1 || !uri2) return false;
    
    // Fast path: exact match
    if (strcmp(uri1, uri2) == 0) return true;
    
    // Compare decoded paths
    char *path1 = uri_to_path(uri1);
    char *path2 = uri_to_path(uri2);
    
    bool equal = path1 && path2 && strcmp(path1, path2) == 0;
    
    free(path1);
    free(path2);
    return equal;
}
```

### Step 2: Implement Document Store

```c
// src/document_store.c
#include "document_store.h"
#include <stdlib.h>
#include <string.h>

struct DocumentStore {
    Document *docs;
    size_t count;
    size_t capacity;
};

DocumentStore *document_store_create(void) {
    DocumentStore *store = calloc(1, sizeof(DocumentStore));
    if (!store) return NULL;
    
    store->capacity = 16;
    store->docs = calloc(store->capacity, sizeof(Document));
    if (!store->docs) {
        free(store);
        return NULL;
    }
    
    return store;
}

void document_store_destroy(DocumentStore *store) {
    if (!store) return;
    
    for (size_t i = 0; i < store->count; i++) {
        free(store->docs[i].uri);
        free(store->docs[i].language_id);
        free(store->docs[i].text);
    }
    free(store->docs);
    free(store);
}

static Document *find_doc(DocumentStore *store, const char *uri) {
    for (size_t i = 0; i < store->count; i++) {
        if (strcmp(store->docs[i].uri, uri) == 0) {
            return &store->docs[i];
        }
    }
    return NULL;
}

bool document_store_open(DocumentStore *store,
                        const char *uri,
                        const char *language_id,
                        int64_t version,
                        const char *text) {
    if (!store || !uri || !text) return false;
    
    // Already open?
    if (find_doc(store, uri)) return false;
    
    // Grow if needed
    if (store->count >= store->capacity) {
        size_t new_cap = store->capacity * 2;
        Document *new_docs = realloc(store->docs, new_cap * sizeof(Document));
        if (!new_docs) return false;
        store->docs = new_docs;
        store->capacity = new_cap;
    }
    
    // Add document
    Document *doc = &store->docs[store->count++];
    doc->uri = strdup(uri);
    doc->language_id = language_id ? strdup(language_id) : strdup("plaintext");
    doc->version = version;
    doc->text = strdup(text);
    doc->text_len = strlen(text);
    
    return true;
}

bool document_store_update(DocumentStore *store,
                          const char *uri,
                          int64_t version,
                          const char *text) {
    if (!store || !uri || !text) return false;
    
    Document *doc = find_doc(store, uri);
    if (!doc) return false;
    
    // Reject stale updates
    if (version <= doc->version) return false;
    
    // Update
    free(doc->text);
    doc->text = strdup(text);
    doc->text_len = strlen(text);
    doc->version = version;
    
    return true;
}

bool document_store_close(DocumentStore *store, const char *uri) {
    if (!store || !uri) return false;
    
    for (size_t i = 0; i < store->count; i++) {
        if (strcmp(store->docs[i].uri, uri) == 0) {
            // Free resources
            free(store->docs[i].uri);
            free(store->docs[i].language_id);
            free(store->docs[i].text);
            
            // Swap with last (order doesn't matter)
            if (i < store->count - 1) {
                store->docs[i] = store->docs[store->count - 1];
            }
            store->count--;
            return true;
        }
    }
    return false;
}

Document *document_store_get(DocumentStore *store, const char *uri) {
    if (!store || !uri) return NULL;
    return find_doc(store, uri);
}

size_t document_store_count(const DocumentStore *store) {
    return store ? store->count : 0;
}

void document_store_foreach(DocumentStore *store, 
                           DocumentCallback cb, 
                           void *userdata) {
    if (!store || !cb) return;
    
    for (size_t i = 0; i < store->count; i++) {
        cb(&store->docs[i], userdata);
    }
}
```

### Step 3: Implement Handlers

```c
// src/handlers.c
#include "handlers.h"
#include "document_store.h"
#include "json.h"
#include <stdio.h>

// Assumes Server has: DocumentStore *documents
extern void log_info(const char *fmt, ...);

void handle_did_open(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    if (!td) return;
    
    const char *uri = json_get_string(td, "uri");
    const char *language = json_get_string(td, "languageId");
    int64_t version = json_get_int(td, "version");
    const char *text = json_get_string(td, "text");
    
    if (!uri || !text) return;
    
    if (document_store_open(server->documents, uri, language, version, text)) {
        log_info("Opened document: %s (version %lld)", uri, version);
        // TODO: Trigger diagnostics
    }
}

void handle_did_change(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    if (!td) return;
    
    const char *uri = json_get_string(td, "uri");
    int64_t version = json_get_int(td, "version");
    
    // Full sync: get complete text from first content change
    const JsonValue *changes = json_get(params, "contentChanges");
    if (!changes || json_array_length(changes) == 0) return;
    
    const JsonValue *change = json_array_get(changes, 0);
    const char *text = json_get_string(change, "text");
    
    if (!uri || !text) return;
    
    if (document_store_update(server->documents, uri, version, text)) {
        log_info("Updated document: %s (version %lld)", uri, version);
        // TODO: Trigger diagnostics
    }
}

void handle_did_close(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    if (!td) return;
    
    const char *uri = json_get_string(td, "uri");
    if (!uri) return;
    
    if (document_store_close(server->documents, uri)) {
        log_info("Closed document: %s", uri);
        // TODO: Clear diagnostics
    }
}
```

## Test Cases

### Document Store Tests

```c
// test/test_document_store.c
#include "document_store.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>

void test_open_and_get(void) {
    DocumentStore *store = document_store_create();
    
    assert(document_store_open(store, "file:///test.txt", "plaintext", 1, "Hello"));
    
    Document *doc = document_store_get(store, "file:///test.txt");
    assert(doc != NULL);
    assert(strcmp(doc->uri, "file:///test.txt") == 0);
    assert(strcmp(doc->text, "Hello") == 0);
    assert(doc->version == 1);
    
    document_store_destroy(store);
    printf("test_open_and_get: PASS\n");
}

void test_update(void) {
    DocumentStore *store = document_store_create();
    
    document_store_open(store, "file:///test.txt", "plaintext", 1, "Hello");
    
    // Valid update
    assert(document_store_update(store, "file:///test.txt", 2, "Hello World"));
    Document *doc = document_store_get(store, "file:///test.txt");
    assert(strcmp(doc->text, "Hello World") == 0);
    assert(doc->version == 2);
    
    // Stale update rejected
    assert(!document_store_update(store, "file:///test.txt", 1, "Old"));
    assert(strcmp(doc->text, "Hello World") == 0);  // Unchanged
    
    document_store_destroy(store);
    printf("test_update: PASS\n");
}

void test_close(void) {
    DocumentStore *store = document_store_create();
    
    document_store_open(store, "file:///a.txt", "plaintext", 1, "A");
    document_store_open(store, "file:///b.txt", "plaintext", 1, "B");
    assert(document_store_count(store) == 2);
    
    assert(document_store_close(store, "file:///a.txt"));
    assert(document_store_count(store) == 1);
    assert(document_store_get(store, "file:///a.txt") == NULL);
    assert(document_store_get(store, "file:///b.txt") != NULL);
    
    // Double close fails
    assert(!document_store_close(store, "file:///a.txt"));
    
    document_store_destroy(store);
    printf("test_close: PASS\n");
}

void test_multiple_documents(void) {
    DocumentStore *store = document_store_create();
    
    // Open many documents
    for (int i = 0; i < 100; i++) {
        char uri[64];
        snprintf(uri, sizeof(uri), "file:///doc%d.txt", i);
        assert(document_store_open(store, uri, "plaintext", 1, "content"));
    }
    
    assert(document_store_count(store) == 100);
    
    // Access a random one
    Document *doc = document_store_get(store, "file:///doc50.txt");
    assert(doc != NULL);
    
    document_store_destroy(store);
    printf("test_multiple_documents: PASS\n");
}

int main(void) {
    test_open_and_get();
    test_update();
    test_close();
    test_multiple_documents();
    printf("\nAll document store tests passed!\n");
    return 0;
}
```

### URI Tests

```c
// test/test_uri.c
#include "uri.h"
#include <assert.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

void test_uri_to_path(void) {
    char *path;
    
    path = uri_to_path("file:///home/user/doc.txt");
    assert(strcmp(path, "/home/user/doc.txt") == 0);
    free(path);
    
    // With space encoding
    path = uri_to_path("file:///home/user/my%20doc.txt");
    assert(strcmp(path, "/home/user/my doc.txt") == 0);
    free(path);
    
    // Invalid URI
    assert(uri_to_path("http://example.com") == NULL);
    assert(uri_to_path(NULL) == NULL);
    
    printf("test_uri_to_path: PASS\n");
}

void test_path_to_uri(void) {
    char *uri;
    
    uri = path_to_uri("/home/user/doc.txt");
    assert(strcmp(uri, "file:///home/user/doc.txt") == 0);
    free(uri);
    
    // With space
    uri = path_to_uri("/home/user/my doc.txt");
    assert(strstr(uri, "%20") != NULL);  // Space is encoded
    free(uri);
    
    printf("test_path_to_uri: PASS\n");
}

void test_roundtrip(void) {
    const char *original = "/path/with spaces/and-special_chars.txt";
    
    char *uri = path_to_uri(original);
    char *path = uri_to_path(uri);
    
    assert(strcmp(path, original) == 0);
    
    free(uri);
    free(path);
    printf("test_roundtrip: PASS\n");
}

int main(void) {
    test_uri_to_path();
    test_path_to_uri();
    test_roundtrip();
    printf("\nAll URI tests passed!\n");
    return 0;
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_document_store && ./test_document_store
test_open_and_get: PASS
test_update: PASS
test_close: PASS
test_multiple_documents: PASS
All document store tests passed!

$ make test_uri && ./test_uri
test_uri_to_path: PASS
test_path_to_uri: PASS
test_roundtrip: PASS
All URI tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_document_store
All heap blocks were freed -- no leaks are possible

$ valgrind --leak-check=full ./test_uri
All heap blocks were freed -- no leaks are possible
```

### Integration Test

Test with LSP messages:

```bash
# Send didOpen + didChange + didClose
./test_protocol.sh | ./wordlib-lsp 2>&1 | grep -E "(Opened|Updated|Closed)"
# Should see logging output
```

## Stretch Goals

1. **Document statistics** - Track edit counts, time since last change
2. **Content hash** - Detect duplicate content across documents
3. **Line index** - Pre-compute line start offsets for faster position conversion
4. **Memory limits** - Refuse documents above a size threshold

## What You'll Learn

By completing this project:
- Document state management
- URI parsing and encoding
- Version-based change tracking
- Memory management for dynamic content

## Next Section Preview

Section 4 uses this document store to implement diagnostics. You'll tokenize document text, check words against the engine, and publish unknown words to the editor.
