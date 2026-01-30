# Code Action Fundamentals

## Why You Need This

Your requirements say: "For each unknown word diagnostic, the server shall offer code actions: Add word to dictionary, (Optional) Ignore word for this session."

Code actions let users fix problems with a single click. They're the "quick fix" feature in editors.

## What to Learn

### What Is a Code Action?

A code action is a command the editor can execute to fix or improve code. In your case:
- User sees "Unknown word: quikc" diagnostic
- User clicks lightbulb / presses shortcut
- Menu shows "Add 'quikc' to dictionary"
- User selects it
- Word is added, diagnostic disappears

### Code Action Request

Editor requests actions for a range:

```json
{
  "jsonrpc": "2.0",
  "id": 20,
  "method": "textDocument/codeAction",
  "params": {
    "textDocument": {
      "uri": "file:///home/user/doc.txt"
    },
    "range": {
      "start": { "line": 5, "character": 10 },
      "end": { "line": 5, "character": 15 }
    },
    "context": {
      "diagnostics": [
        {
          "range": {...},
          "message": "Unknown word: 'quikc'",
          "code": "wordlib.unknown"
        }
      ],
      "only": ["quickfix"]
    }
  }
}
```

**Key fields:**
- `range`: Text range user selected/cursor is in
- `context.diagnostics`: Diagnostics at this range
- `context.only`: Filter for specific action kinds (optional)

### Code Action Response

```json
{
  "jsonrpc": "2.0",
  "id": 20,
  "result": [
    {
      "title": "Add 'quikc' to dictionary",
      "kind": "quickfix",
      "diagnostics": [{...}],
      "command": {
        "title": "Add word",
        "command": "wordlib.addWord",
        "arguments": ["quikc"]
      }
    }
  ]
}
```

Or with workspace edit (changes applied directly):

```json
{
  "title": "Replace with 'quick'",
  "kind": "quickfix",
  "edit": {
    "changes": {
      "file:///doc.txt": [
        {
          "range": {...},
          "newText": "quick"
        }
      ]
    }
  }
}
```

### Code Action Kinds

```c
// Standard kinds
#define CODE_ACTION_QUICKFIX    "quickfix"
#define CODE_ACTION_REFACTOR    "refactor"
#define CODE_ACTION_SOURCE      "source"

// For spell checking, use quickfix
```

### Code Action Structure

```c
typedef struct {
    const char *title;      // User-visible description
    const char *kind;       // quickfix, refactor, etc.
    JsonValue *diagnostics; // Related diagnostics
    JsonValue *command;     // Command to execute (optional)
    JsonValue *edit;        // Workspace edit to apply (optional)
} CodeAction;
```

### Commands vs Edits

**Command:** Server receives `workspace/executeCommand` request
```json
{
  "command": {
    "command": "wordlib.addWord",
    "arguments": ["quikc"]
  }
}
```
- Editor sends command to server
- Server executes and responds
- Good for: adding to dictionary, complex operations

**Edit:** Editor applies changes directly
```json
{
  "edit": {
    "changes": {
      "file:///doc.txt": [{"range": {...}, "newText": "quick"}]
    }
  }
}
```
- Editor applies text changes immediately
- Good for: replacing text

### Handler Structure

```c
JsonValue *handle_code_action(Server *server, const JsonValue *params) {
    // Extract parameters
    const char *uri = json_get_string(
        json_get(params, "textDocument"), "uri");
    const JsonValue *context = json_get(params, "context");
    const JsonValue *diagnostics = json_get(context, "diagnostics");
    
    JsonValue *actions = json_array();
    
    // For each diagnostic, offer actions
    size_t diag_count = json_array_length(diagnostics);
    for (size_t i = 0; i < diag_count; i++) {
        const JsonValue *diag = json_array_get(diagnostics, i);
        const char *code = json_get_string(diag, "code");
        
        // Only handle our diagnostics
        if (!code || strcmp(code, "wordlib.unknown") != 0) {
            continue;
        }
        
        // Extract word from message
        const char *message = json_get_string(diag, "message");
        char *word = extract_word_from_message(message);
        
        if (word) {
            JsonValue *action = create_add_word_action(word, diag);
            json_array_push(actions, action);
            free(word);
        }
    }
    
    return actions;
}
```

### Creating an Action

```c
JsonValue *create_add_word_action(const char *word, const JsonValue *diagnostic) {
    JsonValue *action = json_object();
    
    // Title
    char title[256];
    snprintf(title, sizeof(title), "Add '%s' to dictionary", word);
    json_set_string(action, "title", title);
    
    // Kind
    json_set_string(action, "kind", CODE_ACTION_QUICKFIX);
    
    // Related diagnostics
    JsonValue *diags = json_array();
    json_array_push(diags, json_clone(diagnostic));
    json_set(action, "diagnostics", diags);
    
    // Command to execute
    JsonValue *command = json_object();
    json_set_string(command, "title", "Add word");
    json_set_string(command, "command", "wordlib.addWord");
    
    JsonValue *args = json_array();
    json_array_push_string(args, word);
    json_set(command, "arguments", args);
    
    json_set(action, "command", command);
    
    return action;
}
```

## Where to Learn

1. **LSP Specification - Code Action:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_codeAction

2. **Test with editors:**
   - Trigger code actions on diagnostics
   - Observe the lightbulb behavior

## Practice Exercise

Create and test a code action:

```c
void test_code_action_creation(void) {
    JsonValue *diag = create_test_diagnostic("quikc");
    JsonValue *action = create_add_word_action("quikc", diag);
    
    // Verify structure
    assert(strstr(json_get_string(action, "title"), "quikc") != NULL);
    assert(strcmp(json_get_string(action, "kind"), "quickfix") == 0);
    
    JsonValue *cmd = json_get(action, "command");
    assert(strcmp(json_get_string(cmd, "command"), "wordlib.addWord") == 0);
    
    json_free(action);
    json_free(diag);
    printf("test_code_action_creation: PASS\n");
}
```

## Connection to Project

Code actions complete the user workflow:

```
1. User sees diagnostic: "Unknown word: quikc"
2. User clicks lightbulb
3. Menu: "Add 'quikc' to dictionary"
4. User selects
5. Server adds word
6. Diagnostic disappears
```

This transforms your spell checker from "it shows errors" to "it helps fix errors."
