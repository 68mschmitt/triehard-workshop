# Document State Storage

## Why You Need This

Your requirements say: "Maintain the full UTF-8 text of each open document" and "Track document versions as provided by the client."

You need a data structure to store open documents efficiently—the foundation for all document operations.

## What to Learn

### What to Store Per Document

```c
typedef struct {
    char *uri;          // Document identifier
    char *language_id;  // "plaintext", "markdown", etc.
    int64_t version;    // Monotonically increasing
    char *text;         // Full document content
    size_t text_len;    // Cache strlen for performance
} Document;
```

### Document Store Design

```c
// Simple array-based store (good for few documents)
typedef struct {
    Document *documents;
    size_t count;
    size_t capacity;
} DocumentStore;

// Or hash map for many documents
typedef struct {
    // Use your hash table from Course 1!
    HashMap *by_uri;
} DocumentStore;
```

### Basic Operations

```c
// document_store.h
typedef struct DocumentStore DocumentStore;
typedef struct Document Document;

struct Document {
    char *uri;
    char *language_id;
    int64_t version;
    char *text;
    size_t text_len;
};

DocumentStore *document_store_create(void);
void document_store_destroy(DocumentStore *store);

// Open a new document
bool document_store_open(DocumentStore *store, 
                        const char *uri,
                        const char *language_id,
                        int64_t version,
                        const char *text);

// Update an existing document
bool document_store_update(DocumentStore *store,
                          const char *uri,
                          int64_t version,
                          const char *text);

// Close a document
bool document_store_close(DocumentStore *store, const char *uri);

// Get a document (NULL if not open)
Document *document_store_get(DocumentStore *store, const char *uri);

// Iterate all documents
typedef void (*DocumentIterator)(Document *doc, void *userdata);
void document_store_foreach(DocumentStore *store, DocumentIterator fn, void *userdata);

// Count open documents
size_t document_store_count(const DocumentStore *store);
```

### Implementation (Array-Based)

```c
// document_store.c
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
    store->capacity = 16;
    store->docs = calloc(store->capacity, sizeof(Document));
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

static Document *find_document(DocumentStore *store, const char *uri) {
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
    // Check if already open
    if (find_document(store, uri)) {
        return false;  // Already open
    }
    
    // Grow if needed
    if (store->count >= store->capacity) {
        store->capacity *= 2;
        store->docs = realloc(store->docs, 
                             store->capacity * sizeof(Document));
    }
    
    // Add new document
    Document *doc = &store->docs[store->count++];
    doc->uri = strdup(uri);
    doc->language_id = strdup(language_id);
    doc->version = version;
    doc->text = strdup(text);
    doc->text_len = strlen(text);
    
    return true;
}

bool document_store_update(DocumentStore *store,
                          const char *uri,
                          int64_t version,
                          const char *text) {
    Document *doc = find_document(store, uri);
    if (!doc) return false;
    
    // Check version
    if (version <= doc->version) {
        return false;  // Stale update
    }
    
    // Update text
    free(doc->text);
    doc->text = strdup(text);
    doc->text_len = strlen(text);
    doc->version = version;
    
    return true;
}

bool document_store_close(DocumentStore *store, const char *uri) {
    for (size_t i = 0; i < store->count; i++) {
        if (strcmp(store->docs[i].uri, uri) == 0) {
            // Free document resources
            free(store->docs[i].uri);
            free(store->docs[i].language_id);
            free(store->docs[i].text);
            
            // Move last document to this slot
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
    return find_document(store, uri);
}
```

### Memory Considerations

For a typical editing session:
- 1-10 open documents
- Each document: ~10KB average, up to ~1MB
- Total: ~1-10MB for document storage

The simple array approach is fine for this scale. O(n) lookup on 10 documents is negligible.

### Thread Safety (Not Required)

LSP servers typically run single-threaded (message loop). If you ever need thread safety:

```c
#include <pthread.h>

struct DocumentStore {
    Document *docs;
    size_t count;
    size_t capacity;
    pthread_mutex_t lock;
};

Document *document_store_get(DocumentStore *store, const char *uri) {
    pthread_mutex_lock(&store->lock);
    Document *doc = find_document(store, uri);
    pthread_mutex_unlock(&store->lock);
    return doc;
}
```

But for your project, skip the complexity.

### Document Content Access Patterns

You'll access document text for:

1. **Diagnostics:** Tokenize entire text, check each word
2. **Completion:** Find word at cursor position
3. **Code Actions:** Extract word from diagnostic range

Design accessors for these patterns:

```c
// Get text at a specific range (byte offsets)
char *document_get_range(const Document *doc, size_t start, size_t end) {
    if (start >= doc->text_len || end > doc->text_len || start >= end) {
        return NULL;
    }
    
    size_t len = end - start;
    char *result = malloc(len + 1);
    memcpy(result, doc->text + start, len);
    result[len] = '\0';
    return result;
}

// Get word at position (for completion)
char *document_get_word_at(const Document *doc, size_t offset) {
    if (offset > doc->text_len) return NULL;
    
    // Find word start
    size_t start = offset;
    while (start > 0 && is_word_char(doc->text[start - 1])) {
        start--;
    }
    
    // Find word end
    size_t end = offset;
    while (end < doc->text_len && is_word_char(doc->text[end])) {
        end++;
    }
    
    if (start == end) return NULL;
    
    return document_get_range(doc, start, end);
}
```

## Where to Learn

1. **Your Course 1 hash table** - Could be used for O(1) URI lookup
2. **Other LSP servers** - See how they structure document storage
3. **Generic data structure resources** - Arrays vs hash maps

## Practice Exercise

Build and test the document store:

```c
void test_document_store(void) {
    DocumentStore *store = document_store_create();
    
    // Open documents
    assert(document_store_open(store, "file:///a.txt", "plaintext", 1, "Hello"));
    assert(document_store_open(store, "file:///b.txt", "markdown", 1, "# Title"));
    assert(document_store_count(store) == 2);
    
    // Duplicate open fails
    assert(!document_store_open(store, "file:///a.txt", "plaintext", 1, "World"));
    
    // Get document
    Document *doc = document_store_get(store, "file:///a.txt");
    assert(doc != NULL);
    assert(strcmp(doc->text, "Hello") == 0);
    
    // Update document
    assert(document_store_update(store, "file:///a.txt", 2, "Hello World"));
    assert(strcmp(doc->text, "Hello World") == 0);
    assert(doc->version == 2);
    
    // Stale update fails
    assert(!document_store_update(store, "file:///a.txt", 1, "Old"));
    
    // Close document
    assert(document_store_close(store, "file:///a.txt"));
    assert(document_store_count(store) == 1);
    assert(document_store_get(store, "file:///a.txt") == NULL);
    
    document_store_destroy(store);
    printf("All document store tests passed!\n");
}
```

## Connection to Project

The document store is central to your server:

```
didOpen  ──────▶  document_store_open()   ──▶  Store text
didChange ─────▶  document_store_update() ──▶  Update text
didClose  ─────▶  document_store_close()  ──▶  Remove text

completion ────▶  document_store_get()    ──▶  Get text for prefix extraction
diagnostics ───▶  document_store_get()    ──▶  Get text for spell checking
codeAction ────▶  document_store_get()    ──▶  Get text for word extraction
```

Simple, correct, efficient enough. That's the goal.
