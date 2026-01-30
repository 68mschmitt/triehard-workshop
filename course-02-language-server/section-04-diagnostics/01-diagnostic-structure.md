# Diagnostic Structure

## Why You Need This

Your requirements say: "Identify unknown words" and "Publish diagnostics via `textDocument/publishDiagnostics`." Diagnostics are how your server tells the editor about problems—in your case, unknown words that need attention.

## What to Learn

### What Is a Diagnostic?

A diagnostic is a structured report of an issue at a specific location in a document. The editor displays it as:
- Underlines/squiggles in the text
- Entries in a problems panel
- Hover information

### Diagnostic Structure

```json
{
  "range": {
    "start": { "line": 2, "character": 10 },
    "end": { "line": 2, "character": 15 }
  },
  "severity": 3,
  "code": "wordlib.unknown",
  "source": "wordlib",
  "message": "Unknown word: 'helo'"
}
```

### Fields Explained

**range** (required): Location in the document
```c
typedef struct {
    uint32_t line;       // 0-indexed
    uint32_t character;  // 0-indexed, UTF-16 code units
} Position;

typedef struct {
    Position start;
    Position end;
} Range;
```

**severity** (optional): How serious is this?
```c
typedef enum {
    DIAG_ERROR = 1,       // Red squiggle
    DIAG_WARNING = 2,     // Yellow squiggle
    DIAG_INFORMATION = 3, // Blue squiggle (default for unknown words)
    DIAG_HINT = 4         // Subtle indication
} DiagnosticSeverity;
```

**code** (optional): Machine-readable identifier
```c
// Use a consistent code for your diagnostic type
#define DIAG_CODE_UNKNOWN_WORD "wordlib.unknown"
```

**source** (optional): Who reported this diagnostic?
```c
#define DIAG_SOURCE "wordlib"
```

**message** (required): Human-readable description
```c
char message[256];
snprintf(message, sizeof(message), "Unknown word: '%s'", word);
```

### Additional Fields (Optional)

**tags**: Hint to editor about the nature
```json
"tags": [1]  // 1 = Unnecessary, 2 = Deprecated
```
Not applicable for spell checking.

**relatedInformation**: Additional locations related to this diagnostic
```json
"relatedInformation": [
  {
    "location": { "uri": "...", "range": {...} },
    "message": "Did you mean 'hello'?"
  }
]
```
Could be used for suggestions, but typically handled via code actions instead.

### Building a Diagnostic in C

```c
JsonValue *create_diagnostic(Range range, const char *word) {
    JsonValue *diag = json_object();
    
    // Range
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
    
    // Severity (Information by default)
    json_set_int(diag, "severity", DIAG_INFORMATION);
    
    // Code
    json_set_string(diag, "code", DIAG_CODE_UNKNOWN_WORD);
    
    // Source
    json_set_string(diag, "source", DIAG_SOURCE);
    
    // Message
    char message[256];
    snprintf(message, sizeof(message), "Unknown word: '%s'", word);
    json_set_string(diag, "message", message);
    
    return diag;
}
```

### Helper for Building Range

```c
JsonValue *range_to_json(Range range) {
    JsonValue *r = json_object();
    
    JsonValue *start = json_object();
    json_set_int(start, "line", range.start.line);
    json_set_int(start, "character", range.start.character);
    
    JsonValue *end = json_object();
    json_set_int(end, "line", range.end.line);
    json_set_int(end, "character", range.end.character);
    
    json_set(r, "start", start);
    json_set(r, "end", end);
    
    return r;
}
```

### Severity Configuration

Your requirements mention "Configurable severity (default: Information or Warning)":

```c
typedef struct {
    DiagnosticSeverity unknown_word_severity;
    // ... other config
} ServerConfig;

// Initialize with default
ServerConfig config = {
    .unknown_word_severity = DIAG_INFORMATION
};

// Use in diagnostic creation
json_set_int(diag, "severity", config.unknown_word_severity);
```

## Where to Learn

1. **LSP Specification - Diagnostics:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#diagnostic

2. **See how editors render them:**
   - Open a file with syntax errors in VS Code or Neovim
   - Note the different severity indicators

## Practice Exercise

Create diagnostic builders:

```c
void test_diagnostic_creation(void) {
    // Build a diagnostic for "helo" at line 5, chars 10-14
    Range range = {
        .start = { .line = 5, .character = 10 },
        .end = { .line = 5, .character = 14 }
    };
    
    JsonValue *diag = create_diagnostic(range, "helo");
    
    char *json = json_stringify(diag);
    printf("Diagnostic:\n%s\n", json);
    
    // Verify structure
    assert(json_get_int(diag, "severity") == DIAG_INFORMATION);
    assert(strcmp(json_get_string(diag, "source"), "wordlib") == 0);
    
    free(json);
    json_free(diag);
    printf("test_diagnostic_creation: PASS\n");
}
```

## Connection to Project

Diagnostics are the primary output of your spell checker:

```
Document text → Tokenize → Check each word → Unknown? → Create Diagnostic → Publish
```

Each unknown word becomes one diagnostic. The editor collects them all and displays them to the user.
