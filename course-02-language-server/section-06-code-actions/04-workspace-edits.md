# Workspace Edits

## Why You Need This

Some code actions modify document text directly (like replacing a misspelled word with the correct spelling). Understanding workspace edits lets you offer "Replace with..." suggestions.

## What to Learn

### Command vs Edit

**Command:** Server executes logic, might have side effects
```json
{
  "command": {
    "command": "wordlib.addWord",
    "arguments": ["quikc"]
  }
}
```
Good for: Dictionary changes, session state

**Edit:** Editor applies text changes directly
```json
{
  "edit": {
    "changes": {
      "file:///doc.txt": [
        { "range": {...}, "newText": "quick" }
      ]
    }
  }
}
```
Good for: Text replacements

### Workspace Edit Structure

```json
{
  "changes": {
    "file:///doc1.txt": [
      {
        "range": {
          "start": { "line": 0, "character": 6 },
          "end": { "line": 0, "character": 11 }
        },
        "newText": "quick"
      }
    ],
    "file:///doc2.txt": [
      // ... changes for another file
    ]
  }
}
```

### Creating a Replace Action

```c
JsonValue *create_replace_action(const char *misspelled, 
                                 const char *suggestion,
                                 const char *uri,
                                 Range range,
                                 const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    // Title
    char title[256];
    snprintf(title, sizeof(title), "Replace with '%s'", suggestion);
    json_set_string(action, "title", title);
    json_set_string(action, "kind", "quickfix");
    
    // Link to diagnostic
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    // Workspace edit
    JsonValue *edit = json_object();
    JsonValue *changes = json_object();
    
    // Text edit for this file
    JsonValue *file_edits = json_array();
    JsonValue *text_edit = json_object();
    
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
    
    json_set(text_edit, "range", range_obj);
    json_set_string(text_edit, "newText", suggestion);
    
    json_array_push(file_edits, text_edit);
    json_set(changes, uri, file_edits);
    json_set(edit, "changes", changes);
    
    json_set(action, "edit", edit);
    
    return action;
}
```

### Getting Suggestions from Engine

If your engine has suggestion capability (BK-tree from Course 1):

```c
JsonValue *handle_code_action(Server *server, const JsonValue *params) {
    // ... extract diagnostic info ...
    
    JsonValue *actions = json_array();
    
    // Add "add to dictionary" action
    json_array_push(actions, create_add_word_action(word, diagnostic));
    
    // Add "ignore" action
    json_array_push(actions, create_ignore_action(word, diagnostic));
    
    // Add replacement suggestions (if available)
    Suggestion suggestions[5];
    size_t count = wordlib_suggest(server->engine, word, suggestions, 5);
    
    for (size_t i = 0; i < count; i++) {
        JsonValue *replace = create_replace_action(
            word,
            suggestions[i].word,
            uri,
            diag_range,
            diagnostic
        );
        json_array_push(actions, replace);
    }
    
    return actions;
}
```

### Document Version in Edits

For safety, include document version:

```json
{
  "documentChanges": [
    {
      "textDocument": {
        "uri": "file:///doc.txt",
        "version": 5
      },
      "edits": [
        { "range": {...}, "newText": "quick" }
      ]
    }
  ]
}
```

This ensures edits aren't applied to stale documents.

```c
JsonValue *create_versioned_edit(const char *uri, int64_t version,
                                 Range range, const char *new_text) {
    JsonValue *workspace_edit = json_object();
    JsonValue *doc_changes = json_array();
    
    JsonValue *doc_edit = json_object();
    
    // Versioned text document identifier
    JsonValue *text_doc = json_object();
    json_set_string(text_doc, "uri", uri);
    json_set_int(text_doc, "version", version);
    json_set(doc_edit, "textDocument", text_doc);
    
    // Edits array
    JsonValue *edits = json_array();
    JsonValue *edit = json_object();
    json_set(edit, "range", range_to_json(range));
    json_set_string(edit, "newText", new_text);
    json_array_push(edits, edit);
    json_set(doc_edit, "edits", edits);
    
    json_array_push(doc_changes, doc_edit);
    json_set(workspace_edit, "documentChanges", doc_changes);
    
    return workspace_edit;
}
```

### Multiple Replacements

If the same word appears multiple times:

```c
// Option 1: Separate action for each occurrence
// (Each code action request is for one range)

// Option 2: "Replace all" action
JsonValue *create_replace_all_action(const char *word,
                                     const char *suggestion,
                                     const char *uri,
                                     Document *doc) {
    JsonValue *action = json_object();
    
    char title[256];
    snprintf(title, sizeof(title), "Replace all '%s' with '%s'", 
             word, suggestion);
    json_set_string(action, "title", title);
    json_set_string(action, "kind", "quickfix");
    
    // Find all occurrences
    JsonValue *file_edits = json_array();
    size_t offset = 0;
    
    while ((offset = find_word(doc->text, word, offset)) != (size_t)-1) {
        Range range = byte_span_to_range(doc->text, offset, offset + strlen(word));
        
        JsonValue *edit = json_object();
        json_set(edit, "range", range_to_json(range));
        json_set_string(edit, "newText", suggestion);
        json_array_push(file_edits, edit);
        
        offset += strlen(word);
    }
    
    JsonValue *edit = json_object();
    JsonValue *changes = json_object();
    json_set(changes, uri, file_edits);
    json_set(edit, "changes", changes);
    json_set(action, "edit", edit);
    
    return action;
}
```

### When to Use Edits vs Commands

| Scenario | Use |
|----------|-----|
| Add word to dictionary | Command |
| Ignore word | Command |
| Replace with suggestion | Edit |
| Complex multi-step operation | Command |
| Simple text replacement | Edit |

### Simpler Alternative: Skip Replacement Actions

If you don't have suggestions or want to keep it simple:

```c
JsonValue *handle_code_action(Server *server, const JsonValue *params) {
    // ... 
    
    JsonValue *actions = json_array();
    
    // Just these two actions are enough for a spell checker
    json_array_push(actions, create_add_word_action(word, diagnostic));
    json_array_push(actions, create_ignore_action(word, diagnostic));
    
    return actions;
}
```

Add replacement suggestions later as an enhancement.

## Where to Learn

1. **LSP Specification - WorkspaceEdit:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspaceEdit

2. **TextEdit specification:**
   - Range and newText handling

## Practice Exercise

Test workspace edit creation:

```c
void test_replace_action(void) {
    Range range = {
        .start = { .line = 0, .character = 6 },
        .end = { .line = 0, .character = 11 }
    };
    
    JsonValue *diag = create_test_diagnostic("quikc");
    JsonValue *action = create_replace_action(
        "quikc", "quick", "file:///test.txt", range, diag);
    
    // Verify it has an edit, not a command
    assert(json_has_key(action, "edit"));
    assert(!json_has_key(action, "command"));
    
    // Verify edit content
    JsonValue *edit = json_get(action, "edit");
    JsonValue *changes = json_get(edit, "changes");
    JsonValue *file_edits = json_get(changes, "file:///test.txt");
    JsonValue *text_edit = json_array_get(file_edits, 0);
    
    assert(strcmp(json_get_string(text_edit, "newText"), "quick") == 0);
    
    json_free(action);
    json_free(diag);
    printf("test_replace_action: PASS\n");
}
```

## Connection to Project

Workspace edits enable "quick fix" replacements:

```
User sees: "Unknown word: 'quikc'"
Menu shows:
  - Add 'quikc' to dictionary [command]
  - Ignore 'quikc' [command]
  - Replace with 'quick' [edit]
  - Replace with 'quack' [edit]
```

This turns your spell checker into a complete writing assistant.
