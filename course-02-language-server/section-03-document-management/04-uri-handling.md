# URI Handling

## Why You Need This

Every document in LSP is identified by a URI, typically `file:///path/to/file.txt`. You need to parse, compare, and convert these URIs correctly for document management and workspace dictionary paths.

## What to Learn

### URI Format

Standard file URIs follow this pattern:
```
file:///absolute/path/to/file.txt
      └── Three slashes for local files
```

On Windows:
```
file:///C:/Users/name/file.txt
file:///c%3A/Users/name/file.txt  (encoded colon)
```

### Parsing File URIs

```c
// Extract file path from URI
// Returns malloc'd string, caller must free
// Returns NULL if not a file URI
char *uri_to_path(const char *uri) {
    // Check for file:// prefix
    if (strncmp(uri, "file://", 7) != 0) {
        return NULL;  // Not a file URI
    }
    
    const char *path_start = uri + 7;
    
    // Skip empty authority (localhost)
    // file:///path or file://localhost/path
    if (path_start[0] == '/') {
        path_start++;  // Skip authority separator
    } else if (strncmp(path_start, "localhost/", 10) == 0) {
        path_start += 10;
    }
    
    // Decode percent-encoded characters
    return uri_decode(path_start);
}

// Decode percent-encoded string
char *uri_decode(const char *encoded) {
    size_t len = strlen(encoded);
    char *decoded = malloc(len + 1);
    
    size_t j = 0;
    for (size_t i = 0; i < len; i++) {
        if (encoded[i] == '%' && i + 2 < len) {
            // Decode %XX
            char hex[3] = { encoded[i+1], encoded[i+2], '\0' };
            decoded[j++] = (char)strtol(hex, NULL, 16);
            i += 2;
        } else {
            decoded[j++] = encoded[i];
        }
    }
    decoded[j] = '\0';
    
    return decoded;
}
```

### Creating File URIs

```c
// Convert path to file URI
char *path_to_uri(const char *path) {
    // Simple case: already absolute path on Unix
    if (path[0] == '/') {
        size_t len = strlen(path);
        // Allocate space for encoding (worst case: 3x)
        char *uri = malloc(7 + len * 3 + 1);
        strcpy(uri, "file://");
        uri_encode(path, uri + 7);
        return uri;
    }
    
    // Handle Windows paths (C:\...)
    // ...
    
    return NULL;
}

// Encode path for URI (only encode necessary characters)
void uri_encode(const char *path, char *out) {
    static const char *safe = "/-._~";  // Don't encode these
    
    size_t j = 0;
    for (size_t i = 0; path[i]; i++) {
        char c = path[i];
        
        if (isalnum((unsigned char)c) || strchr(safe, c)) {
            out[j++] = c;
        } else {
            // Percent-encode
            sprintf(out + j, "%%%02X", (unsigned char)c);
            j += 3;
        }
    }
    out[j] = '\0';
}
```

### Comparing URIs

URIs should be compared carefully:

```c
// Compare two URIs for equality
bool uri_equals(const char *uri1, const char *uri2) {
    // Simple case: exact match
    if (strcmp(uri1, uri2) == 0) {
        return true;
    }
    
    // Normalize and compare
    // Different encodings might represent same path
    char *path1 = uri_to_path(uri1);
    char *path2 = uri_to_path(uri2);
    
    if (!path1 || !path2) {
        free(path1);
        free(path2);
        return false;
    }
    
    bool equal = strcmp(path1, path2) == 0;
    free(path1);
    free(path2);
    return equal;
}
```

### Workspace Root Handling

Your requirements mention workspace-specific dictionaries:

```c
// Get workspace dictionary path
char *get_workspace_dict_path(const char *workspace_root_uri) {
    char *root_path = uri_to_path(workspace_root_uri);
    if (!root_path) return NULL;
    
    // Create path: {workspace}/.wordlib/dictionary.txt
    size_t len = strlen(root_path) + 30;  // Room for suffix
    char *dict_path = malloc(len);
    snprintf(dict_path, len, "%s/.wordlib/dictionary.txt", root_path);
    
    free(root_path);
    return dict_path;
}
```

### Special Characters

Common characters that need encoding:

| Character | Encoded |
|-----------|---------|
| Space | `%20` |
| `#` | `%23` |
| `%` | `%25` |
| `?` | `%3F` |
| `:` | `%3A` |

### Cross-Platform Considerations

```c
#ifdef _WIN32
// Windows: Handle drive letters
char *uri_to_path(const char *uri) {
    // file:///C:/path/to/file
    const char *path_start = uri + 8;  // Skip "file:///"
    
    // Check for drive letter
    if (isalpha(path_start[0]) && path_start[1] == ':') {
        // Keep the drive letter, decode rest
        return uri_decode(path_start);
    }
    
    // UNC path: file://server/share/path
    // ...
}
#else
// Unix: Simpler handling
char *uri_to_path(const char *uri) {
    if (strncmp(uri, "file:///", 8) != 0) {
        return NULL;
    }
    return uri_decode(uri + 7);  // Keep leading /
}
#endif
```

### Validation

```c
bool is_valid_file_uri(const char *uri) {
    if (!uri) return false;
    if (strncmp(uri, "file://", 7) != 0) return false;
    
    // Should have a path
    if (strlen(uri) <= 8) return false;  // file:/// minimum
    
    return true;
}
```

## Where to Learn

1. **RFC 3986** - URI Generic Syntax
2. **RFC 8089** - The "file" URI Scheme
3. **LSP Specification** - DocumentUri type

## Practice Exercise

Test URI handling:

```c
void test_uri_handling(void) {
    // Basic path to URI
    char *uri = path_to_uri("/home/user/doc.txt");
    assert(strcmp(uri, "file:///home/user/doc.txt") == 0);
    free(uri);
    
    // URI to path
    char *path = uri_to_path("file:///home/user/doc.txt");
    assert(strcmp(path, "/home/user/doc.txt") == 0);
    free(path);
    
    // Encoded spaces
    path = uri_to_path("file:///home/user/my%20file.txt");
    assert(strcmp(path, "/home/user/my file.txt") == 0);
    free(path);
    
    // Roundtrip
    uri = path_to_uri("/path/with spaces/file.txt");
    path = uri_to_path(uri);
    assert(strcmp(path, "/path/with spaces/file.txt") == 0);
    free(uri);
    free(path);
    
    // Comparison
    assert(uri_equals("file:///test.txt", "file:///test.txt"));
    assert(uri_equals("file:///my%20file.txt", "file:///my file.txt"));
    
    printf("All URI tests passed!\n");
}
```

## Connection to Project

URIs are used throughout:

1. **Document identification:** Every document has a URI
2. **Diagnostics:** Report issues with document URI
3. **Workspace dictionaries:** Convert workspace root URI to file path
4. **Code actions:** Reference document URI for edits

Get URI handling right once, use it everywhere:

```c
// In handler
void handle_did_open(Server *server, const JsonValue *params) {
    const char *uri = json_get_string(json_get(params, "textDocument"), "uri");
    
    // Validate
    if (!is_valid_file_uri(uri)) {
        log_warn("Invalid URI: %s", uri);
        return;
    }
    
    // Store (using URI as-is for comparison)
    document_store_open(server->documents, uri, ...);
}

// For workspace dictionary
void setup_workspace(Server *server, const char *root_uri) {
    char *dict_path = get_workspace_dict_path(root_uri);
    if (dict_path && file_exists(dict_path)) {
        wordlib_load(server->engine, dict_path);
    }
    free(dict_path);
}
```
