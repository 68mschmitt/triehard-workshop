# Resource Management

## Why You Need This

Your requirements say: "Memory usage shall scale with open documents and dictionary size" and "Cleanly release engine resources."

Memory leaks accumulate over long sessions. Unbounded growth can crash the editor or system.

## What to Learn

### Memory Budget

Understand expected usage:

```
Per document:
  - URI string: ~100 bytes
  - Text content: 1-100 KB typical
  - Metadata: ~50 bytes
  - Total: ~1-100 KB per document

Dictionary:
  - 100K words × 10 bytes avg = 1 MB
  - Trie overhead: ~2 MB
  - Total: ~3-5 MB

Server overhead:
  - Transport buffers: 64 KB
  - JSON parsing: temporary
  - Total: ~100 KB

Expected total: 5-50 MB for typical usage
```

### Tracking Allocations

Simple allocation counter:

```c
#ifdef DEBUG
static size_t total_allocated = 0;
static size_t allocation_count = 0;

void *tracked_malloc(size_t size) {
    void *ptr = malloc(size + sizeof(size_t));
    if (ptr) {
        *(size_t *)ptr = size;
        total_allocated += size;
        allocation_count++;
        return (char *)ptr + sizeof(size_t);
    }
    return NULL;
}

void tracked_free(void *ptr) {
    if (ptr) {
        void *real_ptr = (char *)ptr - sizeof(size_t);
        size_t size = *(size_t *)real_ptr;
        total_allocated -= size;
        allocation_count--;
        free(real_ptr);
    }
}

void print_memory_stats(void) {
    fprintf(stderr, "Memory: %zu bytes in %zu allocations\n",
            total_allocated, allocation_count);
}
#endif
```

### Cleanup Patterns

**RAII-style cleanup:**

```c
// Create/destroy pairs
Server *server_create(void);
void server_destroy(Server *server);

DocumentStore *document_store_create(void);
void document_store_destroy(DocumentStore *store);

// Always call destroy
void server_destroy(Server *server) {
    if (!server) return;
    
    // Destroy owned resources in reverse order of creation
    ignore_list_destroy(server->session_ignore);
    document_store_destroy(server->documents);
    wordlib_destroy(server->engine);
    transport_destroy(server->transport);
    config_destroy(&server->config);
    paths_destroy(&server->paths);
    
    free(server->workspace_root);
    free(server);
}
```

### Avoiding Leaks

**Common leak sources:**

```c
// Leak: forgetting to free returned string
char *uri = json_get_string_dup(params, "uri");  // Allocates
// ... use uri ...
free(uri);  // Don't forget!

// Leak: early return without cleanup
JsonValue *handle_something(Server *server, const JsonValue *params) {
    char *temp = strdup(something);
    
    if (error_condition) {
        free(temp);  // Must free before return!
        return NULL;
    }
    
    // ... use temp ...
    free(temp);
    return result;
}

// Better: use goto cleanup
JsonValue *handle_something_better(Server *server, const JsonValue *params) {
    JsonValue *result = NULL;
    char *temp = strdup(something);
    
    if (error_condition) {
        goto cleanup;
    }
    
    result = compute_result(temp);
    
cleanup:
    free(temp);
    return result;
}
```

### Bounded Resources

**Limit document storage:**

```c
#define MAX_OPEN_DOCUMENTS 100
#define MAX_DOCUMENT_SIZE (10 * 1024 * 1024)  // 10 MB

bool document_store_open(DocumentStore *store, const char *uri,
                        const char *language, int64_t version, 
                        const char *text) {
    // Limit document count
    if (store->count >= MAX_OPEN_DOCUMENTS) {
        log_warn("Too many open documents (%zu), rejecting", store->count);
        return false;
    }
    
    // Limit document size
    size_t len = strlen(text);
    if (len > MAX_DOCUMENT_SIZE) {
        log_warn("Document too large (%zu bytes), rejecting", len);
        return false;
    }
    
    // ... proceed with storage
}
```

**Limit completion results:**

```c
#define MAX_COMPLETION_RESULTS 100

// Already implemented in completion handler
```

### Resource Cleanup on Shutdown

```c
void handle_shutdown(Server *server, const JsonValue *params) {
    server->state = SERVER_SHUTTING_DOWN;
    
    // Save state
    save_dictionary(server);
    
    // Resources will be freed in handle_exit
    // (but could log stats here)
    log_info("Shutdown requested, %zu documents open", 
             document_store_count(server->documents));
}

void handle_exit(Server *server, const JsonValue *params) {
    // Log final stats
    log_info("Server exiting");
    
    #ifdef DEBUG
    print_memory_stats();
    #endif
    
    // Destroy server (frees all resources)
    server_destroy(server);
    
    // Exit with appropriate code
    exit(server->state == SERVER_SHUTTING_DOWN ? 0 : 1);
}
```

### Detecting Memory Issues

**Valgrind during development:**

```bash
valgrind --leak-check=full --show-leak-kinds=all ./wordlib-lsp
```

**AddressSanitizer:**

```makefile
ifdef ASAN
CFLAGS += -fsanitize=address -fno-omit-frame-pointer
LDFLAGS += -fsanitize=address
endif
```

**Regular testing:**

```c
void test_no_leaks(void) {
    Server *server = server_create();
    
    // Simulate typical session
    for (int i = 0; i < 100; i++) {
        char uri[64];
        snprintf(uri, sizeof(uri), "file:///test%d.txt", i);
        
        document_store_open(server->documents, uri, "text", 1, "content");
        publish_diagnostics(server, uri);
        document_store_close(server->documents, uri);
    }
    
    server_destroy(server);
    // Valgrind will report any leaks
}
```

## Where to Learn

1. **Valgrind documentation:** Memory debugging tools
2. **ASAN documentation:** AddressSanitizer usage
3. **"Writing Solid Code":** Microsoft's classic on defensive programming

## Practice Exercise

Test resource cleanup:

```c
void test_cleanup(void) {
    // Track allocations before
    size_t before_allocs = get_allocation_count();
    
    // Create and destroy server
    Server *server = server_create();
    
    // Do some work
    document_store_open(server->documents, "file:///test.txt", 
                       "plaintext", 1, "Hello world");
    wordlib_add_word(server->engine, "testword");
    
    server_destroy(server);
    
    // Should be back to baseline
    size_t after_allocs = get_allocation_count();
    assert(after_allocs == before_allocs);
    
    printf("test_cleanup: PASS\n");
}
```

## Connection to Project

Good resource management means:

| Bad | Good |
|-----|------|
| Memory grows over time | Memory stays bounded |
| Slow after hours of use | Consistent performance |
| Crashes on large projects | Handles any project |
| Leaks in Valgrind | Clean Valgrind output |

Your server runs for hours or days. Make sure it can.
