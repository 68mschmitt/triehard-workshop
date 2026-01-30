# Document Lifecycle Events

## Why You Need This

Your requirements specify support for `textDocument/didOpen`, `textDocument/didChange`, and `textDocument/didClose`. These notifications tell you which documents the editor has open and what their contents are. Without this, you can't check spelling or provide completions.

## What to Learn

### The Document Lifecycle

```
User opens file   ──────▶  didOpen   ──────▶  Server stores text
User edits file   ──────▶  didChange ──────▶  Server updates text
User closes file  ──────▶  didClose  ──────▶  Server forgets text
```

### textDocument/didOpen

Sent when user opens a file:

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didOpen",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt",
      "languageId": "plaintext",
      "version": 1,
      "text": "The full document content\ngoes here."
    }
  }
}
```

**Fields:**
- `uri`: Unique identifier for the document (file path as URI)
- `languageId`: File type ("plaintext", "markdown", etc.)
- `version`: Starts at 1, increments with each change
- `text`: Complete document content

**Your response:** None (it's a notification).

**Your action:** Store the document in your document store.

```c
void handle_did_open(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    
    const char *uri = json_get_string(td, "uri");
    const char *language = json_get_string(td, "languageId");
    int64_t version = json_get_int(td, "version");
    const char *text = json_get_string(td, "text");
    
    // Store document
    document_store_open(server->documents, uri, language, version, text);
    
    // Immediately check for unknown words
    publish_diagnostics(server, uri);
}
```

### textDocument/didChange

Sent when user modifies the document:

**Full sync (TextDocumentSyncKind = 1):**
```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didChange",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt",
      "version": 2
    },
    "contentChanges": [
      {
        "text": "The complete new document content"
      }
    ]
  }
}
```

**Incremental sync (TextDocumentSyncKind = 2):**
```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didChange",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt",
      "version": 2
    },
    "contentChanges": [
      {
        "range": {
          "start": { "line": 0, "character": 4 },
          "end": { "line": 0, "character": 4 }
        },
        "text": "inserted "
      }
    ]
  }
}
```

**Your action (full sync):**
```c
void handle_did_change(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    const char *uri = json_get_string(td, "uri");
    int64_t version = json_get_int(td, "version");
    
    // With full sync, just get the new text
    const JsonValue *changes = json_get(params, "contentChanges");
    const JsonValue *change = json_array_get(changes, 0);
    const char *new_text = json_get_string(change, "text");
    
    // Update stored document
    document_store_update(server->documents, uri, version, new_text);
    
    // Re-check for unknown words
    publish_diagnostics(server, uri);
}
```

### textDocument/didClose

Sent when user closes the file:

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didClose",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt"
    }
  }
}
```

**Your action:**
```c
void handle_did_close(Server *server, const JsonValue *params) {
    const JsonValue *td = json_get(params, "textDocument");
    const char *uri = json_get_string(td, "uri");
    
    // Remove from store
    document_store_close(server->documents, uri);
    
    // Clear diagnostics for this document
    clear_diagnostics(server, uri);
}
```

### Version Numbers

Versions help detect out-of-order messages:

```c
void document_store_update(DocumentStore *store, const char *uri, 
                           int64_t version, const char *text) {
    Document *doc = document_store_get(store, uri);
    if (!doc) {
        // Document not open - ignore or log warning
        return;
    }
    
    // Only accept newer versions
    if (version <= doc->version) {
        log_warn("Ignoring stale update: version %lld <= %lld",
                 version, doc->version);
        return;
    }
    
    doc->version = version;
    free(doc->text);
    doc->text = strdup(text);
}
```

### Language ID Filtering

Your requirements say: "Plain text documents, Markdown documents."

```c
bool is_supported_language(const char *language_id) {
    return strcmp(language_id, "plaintext") == 0 ||
           strcmp(language_id, "markdown") == 0;
}

void handle_did_open(Server *server, const JsonValue *params) {
    // ...
    const char *language = json_get_string(td, "languageId");
    
    if (!is_supported_language(language)) {
        // Don't process unsupported languages
        return;
    }
    // ...
}
```

## Where to Learn

1. **LSP Specification - Text Document Synchronization:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization

2. **Trace real editors:** Watch what Neovim sends

3. **Reference implementations:** See how others handle document state

## Practice Exercise

Create a minimal document handler that:
1. Logs all document events to stderr
2. Tracks open document count
3. Prints document content on change

```c
void test_document_events(void) {
    // Simulate events
    JsonValue *open_params = json_parse(
        "{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\","
        "\"languageId\":\"plaintext\","
        "\"version\":1,"
        "\"text\":\"Hello world\"}}"
    );
    
    handle_did_open(server, open_params);
    // Should print: "Opened: file:///test.txt (plaintext)"
    
    JsonValue *change_params = json_parse(
        "{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\","
        "\"version\":2},"
        "\"contentChanges\":[{\"text\":\"Hello there world\"}]}"
    );
    
    handle_did_change(server, change_params);
    // Should print: "Changed: file:///test.txt to version 2"
    
    JsonValue *close_params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"}}"
    );
    
    handle_did_close(server, close_params);
    // Should print: "Closed: file:///test.txt"
}
```

## Connection to Project

Document events are the input to your spell checker:

```
User types   →   didChange   →   Update text   →   Re-tokenize   →   Check words   →   Diagnostics
```

Every feature depends on accurate document tracking. Get this right and diagnostics/completions become straightforward.
