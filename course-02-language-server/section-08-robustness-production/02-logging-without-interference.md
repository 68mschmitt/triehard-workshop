# Logging Without Interference

## Why You Need This

Your requirements say: "Emit logs to stderr or a configurable file" and "Logging shall not interfere with stdio protocol communication."

Good logs help debugging. Bad logs break the protocol or drown you in noise.

## What to Learn

### The Problem

LSP uses stdout for messages. If you `printf()` debug info, you corrupt the protocol:

```
Content-Length: 42
{"jsonrpc":"2.0","id":1,"result":null}Debug: processing completion  <-- Protocol violation!
Content-Length: 50
...
```

### Solution: stderr and Files

**stderr** is safe—editors expect it for diagnostics:

```c
void log_debug(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    fprintf(stderr, "[DEBUG] ");
    vfprintf(stderr, fmt, args);
    fprintf(stderr, "\n");
    va_end(args);
}
```

**File logging** for persistent logs:

```c
static FILE *log_file = NULL;

void log_init(const char *path) {
    if (path) {
        log_file = fopen(path, "a");
    }
}

void log_message(const char *level, const char *fmt, va_list args) {
    FILE *out = log_file ? log_file : stderr;
    
    // Timestamp
    time_t now = time(NULL);
    char timestamp[32];
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", localtime(&now));
    
    fprintf(out, "[%s] [%s] ", timestamp, level);
    vfprintf(out, fmt, args);
    fprintf(out, "\n");
    
    if (log_file) {
        fflush(log_file);  // Ensure writes are visible
    }
}
```

### Log Levels

```c
typedef enum {
    LOG_ERROR = 0,
    LOG_WARN = 1,
    LOG_INFO = 2,
    LOG_DEBUG = 3,
    LOG_TRACE = 4
} LogLevel;

static LogLevel current_level = LOG_INFO;

void log_set_level(LogLevel level) {
    current_level = level;
}

void log_error(const char *fmt, ...) {
    if (current_level >= LOG_ERROR) {
        va_list args;
        va_start(args, fmt);
        log_message("ERROR", fmt, args);
        va_end(args);
    }
}

void log_warn(const char *fmt, ...) {
    if (current_level >= LOG_WARN) {
        va_list args;
        va_start(args, fmt);
        log_message("WARN", fmt, args);
        va_end(args);
    }
}

void log_info(const char *fmt, ...) {
    if (current_level >= LOG_INFO) {
        va_list args;
        va_start(args, fmt);
        log_message("INFO", fmt, args);
        va_end(args);
    }
}

void log_debug(const char *fmt, ...) {
    if (current_level >= LOG_DEBUG) {
        va_list args;
        va_start(args, fmt);
        log_message("DEBUG", fmt, args);
        va_end(args);
    }
}
```

### What to Log

**Always log:**
- Server startup/shutdown
- Errors and warnings
- Configuration applied

**Optionally log (debug level):**
- Requests received
- Responses sent
- Document operations

**Never log:**
- Full document contents (privacy, size)
- User passwords/tokens (security)
- Every single character typed (noise)

```c
void handle_did_open(Server *server, const JsonValue *params) {
    const char *uri = /* extract */;
    const char *language = /* extract */;
    
    log_info("Document opened: %s (%s)", uri, language);
    
    // Don't log: log_debug("Content: %s", text);  // Too much
}

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    const char *uri = /* extract */;
    log_debug("Completion requested: %s at %d:%d", uri, line, char);
    
    // ... compute ...
    
    log_debug("Returning %zu completions", count);
    return result;
}
```

### Log Configuration

Via environment:
```c
void log_init_from_env(void) {
    const char *level = getenv("WORDLIB_LOG_LEVEL");
    if (level) {
        if (strcmp(level, "error") == 0) log_set_level(LOG_ERROR);
        else if (strcmp(level, "warn") == 0) log_set_level(LOG_WARN);
        else if (strcmp(level, "info") == 0) log_set_level(LOG_INFO);
        else if (strcmp(level, "debug") == 0) log_set_level(LOG_DEBUG);
    }
    
    const char *file = getenv("WORDLIB_LOG_FILE");
    if (file) {
        log_init(file);
    }
}
```

Via LSP configuration:
```c
void config_apply(ServerConfig *config, const JsonValue *settings) {
    // ...
    
    const char *log_level = json_get_string(wordlib, "logLevel");
    if (log_level) {
        log_set_level(parse_log_level(log_level));
    }
}
```

### Structured Logging (Advanced)

For machine-parseable logs:

```c
void log_structured(const char *level, const char *event, ...) {
    // Output JSON to log file
    fprintf(log_file, 
            "{\"time\":\"%s\",\"level\":\"%s\",\"event\":\"%s\"",
            timestamp(), level, event);
    
    // Add key-value pairs
    va_list args;
    va_start(args, event);
    const char *key;
    while ((key = va_arg(args, const char *)) != NULL) {
        const char *value = va_arg(args, const char *);
        fprintf(log_file, ",\"%s\":\"%s\"", key, value);
    }
    va_end(args);
    
    fprintf(log_file, "}\n");
}

// Usage
log_structured("INFO", "document.opened", 
               "uri", uri, 
               "language", language,
               NULL);
```

### Testing Logs

```c
void test_logging(void) {
    // Capture stderr
    int pipefd[2];
    pipe(pipefd);
    int old_stderr = dup(STDERR_FILENO);
    dup2(pipefd[1], STDERR_FILENO);
    
    log_set_level(LOG_DEBUG);
    log_info("Test message");
    
    // Restore and read
    fflush(stderr);
    dup2(old_stderr, STDERR_FILENO);
    close(pipefd[1]);
    
    char buf[256];
    read(pipefd[0], buf, sizeof(buf));
    close(pipefd[0]);
    
    assert(strstr(buf, "Test message") != NULL);
    printf("test_logging: PASS\n");
}
```

## Where to Learn

1. **syslog patterns:** UNIX logging conventions
2. **LSP server implementations:** How others handle logging
3. **Log aggregation:** For production debugging

## Practice Exercise

Implement and test log levels:

```c
void exercise_logging(void) {
    log_set_level(LOG_WARN);
    
    log_error("This should appear");   // Yes
    log_warn("This should appear");    // Yes
    log_info("This should NOT appear"); // No (below threshold)
    log_debug("This should NOT appear"); // No
    
    log_set_level(LOG_DEBUG);
    
    log_debug("Now this appears");  // Yes
}
```

## Connection to Project

Good logging helps users and developers:

```bash
# User debugging
WORDLIB_LOG_LEVEL=debug ./wordlib-lsp 2>wordlib.log

# Check the log
tail -f wordlib.log
[2024-01-15 10:30:45] [INFO] Server starting
[2024-01-15 10:30:45] [INFO] Loaded dictionary: /home/user/.wordlib/dictionary.txt
[2024-01-15 10:30:46] [DEBUG] Document opened: file:///project/README.md
[2024-01-15 10:30:46] [DEBUG] Published 3 diagnostics
```

When something goes wrong, logs tell the story.
