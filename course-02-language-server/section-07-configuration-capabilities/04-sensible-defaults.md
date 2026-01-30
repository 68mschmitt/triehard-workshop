# Sensible Defaults

## Why You Need This

Your requirements say: "Provide sensible defaults such that it works out-of-the-box with no configuration."

A good tool works immediately. Configuration is for customization, not basic operation.

## What to Learn

### The Zero-Config Experience

User installs your server, opens a document, and it just works:
- Unknown words are highlighted
- Completions appear when typing
- Code actions available for diagnostics
- No setup required

### Default Values

```c
#define DEFAULT_DIAGNOSTIC_SEVERITY 3  // Information
#define DEFAULT_CASE_SENSITIVE false
#define DEFAULT_MAX_SUGGESTIONS 5
#define DEFAULT_MAX_COMPLETIONS 50
#define DEFAULT_MIN_PREFIX_LENGTH 1

ServerConfig default_config(void) {
    return (ServerConfig){
        .diagnostic_severity = DEFAULT_DIAGNOSTIC_SEVERITY,
        .case_sensitive = DEFAULT_CASE_SENSITIVE,
        .max_suggestion_distance = 2,
        .max_completions = DEFAULT_MAX_COMPLETIONS,
        .min_prefix_length = DEFAULT_MIN_PREFIX_LENGTH,
        .global_dictionary_path = get_default_global_dict(),
        .workspace_dictionary_path = NULL,  // Derived later
        .enabled_languages = NULL,  // All enabled
        .enabled_languages_count = 0
    };
}

char *get_default_global_dict(void) {
    const char *home = getenv("HOME");
    if (!home) home = "/tmp";
    
    size_t len = strlen(home) + 30;
    char *path = malloc(len);
    snprintf(path, len, "%s/.wordlib/dictionary.txt", home);
    return path;
}
```

### Built-in Word List

For immediate usefulness, include a basic word list:

```c
// Common English words that shouldn't trigger diagnostics
static const char *BUILTIN_WORDS[] = {
    "the", "a", "an", "is", "are", "was", "were",
    "I", "you", "he", "she", "it", "we", "they",
    "and", "or", "but", "if", "then", "else",
    "this", "that", "these", "those",
    "in", "on", "at", "by", "for", "with", "to",
    // ... more common words
    NULL
};

void load_builtin_words(WordLib *engine) {
    for (int i = 0; BUILTIN_WORDS[i]; i++) {
        wordlib_add_word(engine, BUILTIN_WORDS[i]);
    }
}
```

Or load from an embedded resource:
```c
// Generated from a word list file
extern const char _binary_words_txt_start[];
extern const char _binary_words_txt_end[];

void load_builtin_words(WordLib *engine) {
    // Parse embedded word list
}
```

### Graceful Missing Files

Don't fail if dictionary file doesn't exist:

```c
void load_dictionary_if_exists(Server *server, const char *path) {
    if (!path) return;
    
    FILE *f = fopen(path, "r");
    if (!f) {
        // File doesn't exist - that's okay
        log_debug("Dictionary not found: %s (will create on first add)", path);
        return;
    }
    
    // Load from file
    char line[256];
    while (fgets(line, sizeof(line), f)) {
        // Strip newline
        line[strcspn(line, "\n\r")] = '\0';
        if (line[0] && line[0] != '#') {  // Skip empty lines and comments
            wordlib_add_word(server->engine, line);
        }
    }
    
    fclose(f);
    log_info("Loaded dictionary: %s", path);
}
```

### Auto-Create on Save

Create dictionary file/directory on first save:

```c
bool save_dictionary(Server *server, const char *path) {
    if (!path) return false;
    
    // Ensure directory exists
    char *dir = get_directory(path);
    if (dir) {
        mkdir_p(dir);  // Create all parent directories
        free(dir);
    }
    
    // Save
    return wordlib_save(server->engine, path);
}

void mkdir_p(const char *path) {
    char *tmp = strdup(path);
    char *p = NULL;
    
    for (p = tmp + 1; *p; p++) {
        if (*p == '/') {
            *p = '\0';
            mkdir(tmp, 0755);
            *p = '/';
        }
    }
    mkdir(tmp, 0755);
    
    free(tmp);
}
```

### Default Language Support

Support common text file types by default:

```c
bool is_supported_by_default(const char *language_id) {
    // These are always supported
    if (strcmp(language_id, "plaintext") == 0) return true;
    if (strcmp(language_id, "markdown") == 0) return true;
    
    // Common text-heavy formats
    if (strcmp(language_id, "restructuredtext") == 0) return true;
    if (strcmp(language_id, "asciidoc") == 0) return true;
    if (strcmp(language_id, "latex") == 0) return true;
    
    return false;
}
```

### Self-Documenting Defaults

Log defaults so users understand behavior:

```c
void log_server_config(ServerConfig *config) {
    log_info("Server configuration:");
    log_info("  Diagnostic severity: %s", 
             severity_to_string(config->diagnostic_severity));
    log_info("  Case sensitive: %s",
             config->case_sensitive ? "yes" : "no");
    log_info("  Global dictionary: %s",
             config->global_dictionary_path ? config->global_dictionary_path : "(none)");
    log_info("  Workspace dictionary: %s",
             config->workspace_dictionary_path ? config->workspace_dictionary_path : "(none)");
}
```

### Configuration Validation

Reject invalid config, keep defaults:

```c
void apply_config_value(ServerConfig *config, const char *key, JsonValue *value) {
    if (strcmp(key, "diagnosticSeverity") == 0) {
        const char *sev = json_get_string(value, NULL);
        int parsed = parse_severity(sev);
        if (parsed >= 1 && parsed <= 4) {
            config->diagnostic_severity = parsed;
        } else {
            log_warn("Invalid severity '%s', keeping default", sev);
        }
    }
    // ... other config values
}
```

## Where to Learn

1. **UX principles:** "Convention over configuration"
2. **Other tools:** How they handle first-run experience
3. **Default behavior:** What users expect without reading docs

## Practice Exercise

Test zero-config operation:

```c
void test_zero_config(void) {
    // Create server with no config
    Server *server = server_create();
    server_initialize(server, json_parse("{\"rootUri\":null,\"capabilities\":{}}"));
    
    // Open document
    handle_did_open(server, json_parse(
        "{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\","
        "\"languageId\":\"plaintext\","
        "\"version\":1,"
        "\"text\":\"Hello wrold\"}}"));
    
    // Should produce diagnostics (unknown "wrold")
    Document *doc = document_store_get(server->documents, "file:///test.txt");
    JsonValue *diags = compute_diagnostics(server, doc);
    
    // Even without any config, should find unknown word
    assert(json_array_length(diags) == 1);
    
    json_free(diags);
    server_destroy(server);
    printf("test_zero_config: PASS\n");
}
```

## Connection to Project

Zero-config means:

```
User: <installs server, opens file>
Result: It works!

vs.

User: <installs server, opens file>
Server: "Error: dictionary path not configured"
User: "Ugh, let me read the docs..."
```

The first experience matters. Make it work, make it obvious.
