# Error Handling Patterns

## Why You Need This

Your requirements say: "Error reporting via return codes or structured results." In C, you don't have exceptions. You need clear patterns for communicating errors to callers.

## What to Learn

### The Basic Options

**1. Return codes (errno style):**
```c
int result = wordlib_load(wl, path);
if (result < 0) {
    // Error occurred
}
```

**2. Boolean success:**
```c
if (!wordlib_add_word(wl, word)) {
    // Failed
}
```

**3. NULL for failure (allocation/lookup):**
```c
WordLib *wl = wordlib_create();
if (!wl) {
    // Out of memory
}
```

**4. Out parameter:**
```c
WLError err;
size_t count = wordlib_complete(wl, prefix, results, max, &err);
if (err != WL_OK) {
    // Error, count is undefined
}
```

### Error Code Enum

```c
typedef enum {
    WL_OK = 0,
    
    // Input errors (caller's fault)
    WL_ERR_NULL = -1,        // NULL argument
    WL_ERR_INVALID = -2,     // Invalid argument value
    
    // Resource errors
    WL_ERR_MEMORY = -10,     // Out of memory
    WL_ERR_IO = -11,         // I/O error
    WL_ERR_PERMISSION = -12, // Permission denied
    
    // State errors
    WL_ERR_NOT_FOUND = -20,  // Word not in dictionary
    WL_ERR_EXISTS = -21,     // Word already in dictionary
    
    // Format errors
    WL_ERR_FORMAT = -30,     // Invalid file format
    WL_ERR_VERSION = -31,    // Unsupported version
} WLError;
```

### Error Messages

```c
const char *wl_strerror(WLError err) {
    switch (err) {
        case WL_OK: return "Success";
        case WL_ERR_NULL: return "NULL argument";
        case WL_ERR_INVALID: return "Invalid argument";
        case WL_ERR_MEMORY: return "Out of memory";
        case WL_ERR_IO: return "I/O error";
        case WL_ERR_PERMISSION: return "Permission denied";
        case WL_ERR_NOT_FOUND: return "Not found";
        case WL_ERR_EXISTS: return "Already exists";
        case WL_ERR_FORMAT: return "Invalid format";
        case WL_ERR_VERSION: return "Unsupported version";
        default: return "Unknown error";
    }
}
```

### Structured Results

For complex operations, return a struct:

```c
typedef struct {
    bool success;
    WLError error;
    union {
        struct {
            const char **words;
            size_t count;
        } completion;
        struct {
            Span *spans;
            size_t count;
        } unknown_words;
    } data;
} WLResult;

WLResult wordlib_check_text(const WordLib *wl, const char *text) {
    WLResult result = { .success = true, .error = WL_OK };
    
    // ... do work ...
    
    if (error_occurred) {
        result.success = false;
        result.error = WL_ERR_MEMORY;
        return result;
    }
    
    result.data.unknown_words.spans = spans;
    result.data.unknown_words.count = count;
    return result;
}
```

### Callback Error Handling

For iteration, let callback signal stop:

```c
typedef bool (*WordCallback)(const char *word, void *ctx);

void wordlib_foreach(const WordLib *wl, WordCallback cb, void *ctx) {
    for (/* each word */) {
        if (!cb(word, ctx)) {
            break;  // Callback requested stop
        }
    }
}

// Usage
bool print_word(const char *word, void *ctx) {
    printf("%s\n", word);
    return true;  // Continue
}

bool find_word(const char *word, void *ctx) {
    const char *target = ctx;
    if (strcmp(word, target) == 0) {
        return false;  // Found it, stop
    }
    return true;  // Keep looking
}
```

### Defensive Programming

```c
int wordlib_add_word(WordLib *wl, const char *word) {
    // Validate inputs early
    if (!wl) {
        return WL_ERR_NULL;
    }
    if (!word) {
        return WL_ERR_NULL;
    }
    if (*word == '\0') {
        return WL_ERR_INVALID;  // Empty word
    }
    if (strlen(word) > MAX_WORD_LENGTH) {
        return WL_ERR_INVALID;  // Too long
    }
    
    // Now safe to proceed
    // ...
}
```

### Error Propagation

```c
int wordlib_load(WordLib *wl, const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) {
        if (errno == ENOENT) return WL_OK;  // Missing is OK
        if (errno == EACCES) return WL_ERR_PERMISSION;
        return WL_ERR_IO;
    }
    
    char line[4096];
    while (fgets(line, sizeof(line), f)) {
        int err = process_line(wl, line);
        if (err != WL_OK) {
            fclose(f);
            return err;  // Propagate error up
        }
    }
    
    if (ferror(f)) {
        fclose(f);
        return WL_ERR_IO;
    }
    
    fclose(f);
    return WL_OK;
}
```

### Logging vs Returning

```c
// Bad: logs error, returns nothing useful
void wordlib_load(WordLib *wl, const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) {
        fprintf(stderr, "Error: cannot open %s\n", path);
        return;  // Caller has no idea what happened!
    }
}

// Good: returns error, lets caller decide what to do
int wordlib_load(WordLib *wl, const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) {
        return WL_ERR_IO;  // Caller can log, retry, whatever
    }
}
```

The engine should never print. Let adapters handle user communication.

## Where to Learn

1. **"C Programming: A Modern Approach"** - Error handling chapter
2. **SQLite source code** - Excellent error handling patterns
3. **Linux kernel coding style** - Error handling guidelines
4. **"Patterns in C"** - Error handling patterns

## Practice Exercise

Design error handling for your API:

1. Define error enum with meaningful codes
2. Implement `wl_strerror()` function
3. Add validation to all public functions
4. Ensure errors propagate correctly (no silent failures)

Test by intentionally triggering each error case.

## Connection to Project

Clean error handling means:
- CLI can give helpful messages: "Error: permission denied opening ~/.wordlib/dict.txt"
- LSP adapter can report diagnostics appropriately
- Debugging is easy (error codes tell you what went wrong)
- Users trust your tool (it fails gracefully, not mysteriously)
