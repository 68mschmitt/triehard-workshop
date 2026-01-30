# Completion Items

## Why You Need This

Your requirements say: "Completion items shall insert plain text, be ranked or ordered based on engine results, use a consistent completion item kind."

Understanding completion item structure ensures users see well-formatted suggestions.

## What to Learn

### Completion Item Structure

```json
{
  "label": "hello",
  "kind": 1,
  "detail": "from dictionary",
  "documentation": "A greeting word",
  "sortText": "0001",
  "filterText": "hello",
  "insertText": "hello",
  "insertTextFormat": 1,
  "textEdit": {
    "range": { "start": {...}, "end": {...} },
    "newText": "hello"
  }
}
```

### Required Fields

**label** (required): What the user sees in the menu
```c
json_set_string(item, "label", word);
```

### Optional but Useful Fields

**kind**: Icon indicator
```c
typedef enum {
    COMPLETION_TEXT = 1,
    COMPLETION_METHOD = 2,
    COMPLETION_FUNCTION = 3,
    COMPLETION_CONSTRUCTOR = 4,
    COMPLETION_FIELD = 5,
    COMPLETION_VARIABLE = 6,
    COMPLETION_CLASS = 7,
    COMPLETION_INTERFACE = 8,
    COMPLETION_MODULE = 9,
    COMPLETION_PROPERTY = 10,
    COMPLETION_UNIT = 11,
    COMPLETION_VALUE = 12,
    COMPLETION_ENUM = 13,
    COMPLETION_KEYWORD = 14,
    COMPLETION_SNIPPET = 15,
    COMPLETION_COLOR = 16,
    COMPLETION_FILE = 17,
    COMPLETION_REFERENCE = 18,
    // ... more
} CompletionItemKind;

// For words, use Text
json_set_int(item, "kind", COMPLETION_TEXT);
```

**detail**: Secondary text shown next to label
```c
json_set_string(item, "detail", "dictionary");
```

**sortText**: Controls ordering (lexicographic sort)
```c
// To preserve engine ordering, use index-based sort text
char sort_text[16];
snprintf(sort_text, sizeof(sort_text), "%04zu", index);
json_set_string(item, "sortText", sort_text);
```

**filterText**: What to match when user types more
```c
// Usually same as label; omit if identical
// Useful for special cases (e.g., match "cls" to "class")
```

**insertText**: What gets inserted when selected
```c
// Usually same as label; omit if identical
json_set_string(item, "insertText", word);
```

### Text Edit vs Insert Text

**Simple (insertText):** Just insert at cursor
```json
{ "label": "hello", "insertText": "hello" }
```
Editor replaces the typed prefix with insertText.

**Precise (textEdit):** Replace specific range
```json
{
  "label": "hello",
  "textEdit": {
    "range": {
      "start": { "line": 0, "character": 0 },
      "end": { "line": 0, "character": 3 }
    },
    "newText": "hello"
  }
}
```
Use textEdit when you need precise control.

### Building Completion Items

```c
JsonValue *create_completion_item(const char *word, size_t index) {
    JsonValue *item = json_object();
    
    // Required: label
    json_set_string(item, "label", word);
    
    // Kind: Text
    json_set_int(item, "kind", COMPLETION_TEXT);
    
    // Sort text to preserve engine ordering
    char sort_text[16];
    snprintf(sort_text, sizeof(sort_text), "%06zu", index);
    json_set_string(item, "sortText", sort_text);
    
    return item;
}
```

### With Text Edit for Prefix Replacement

```c
JsonValue *create_completion_item_with_edit(
    const char *word, 
    size_t index,
    Position prefix_start,
    Position prefix_end
) {
    JsonValue *item = json_object();
    
    json_set_string(item, "label", word);
    json_set_int(item, "kind", COMPLETION_TEXT);
    
    // Text edit to replace prefix
    JsonValue *edit = json_object();
    
    JsonValue *range = json_object();
    JsonValue *start = json_object();
    json_set_int(start, "line", prefix_start.line);
    json_set_int(start, "character", prefix_start.character);
    JsonValue *end = json_object();
    json_set_int(end, "line", prefix_end.line);
    json_set_int(end, "character", prefix_end.character);
    json_set(range, "start", start);
    json_set(range, "end", end);
    
    json_set(edit, "range", range);
    json_set_string(edit, "newText", word);
    
    json_set(item, "textEdit", edit);
    
    char sort_text[16];
    snprintf(sort_text, sizeof(sort_text), "%06zu", index);
    json_set_string(item, "sortText", sort_text);
    
    return item;
}
```

### Completion Item Resolve (Advanced)

For expensive-to-compute details (documentation), use resolve:

1. Send minimal items in completion response
2. When user hovers on item, editor sends `completionItem/resolve`
3. Server adds full details

```c
// Initial response - minimal
{ "label": "hello", "kind": 1, "data": { "word": "hello" } }

// Resolve request comes when user focuses on item
// Resolve response - add details
{ "label": "hello", "kind": 1, "documentation": "A greeting" }
```

For word completion, skip resolve—just include everything upfront.

### Example Complete Handler

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // ... extract uri, position ...
    
    Document *doc = document_store_get(server->documents, uri);
    if (!doc) return json_array();
    
    // Convert position and extract prefix
    size_t offset = position_to_offset(doc->text, line, character);
    size_t prefix_start;
    char *prefix = extract_prefix_with_start(doc->text, offset, &prefix_start);
    
    if (!prefix || strlen(prefix) == 0) {
        free(prefix);
        return json_array();
    }
    
    // Get completions
    const char *results[100];
    size_t count = wordlib_complete(server->engine, prefix, results, 100);
    
    // Build response with text edits
    Position start_pos = byte_to_position(doc->text, prefix_start);
    Position end_pos = byte_to_position(doc->text, offset);
    
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        JsonValue *item = create_completion_item_with_edit(
            results[i], i, start_pos, end_pos);
        json_array_push(items, item);
    }
    
    free(prefix);
    return items;
}
```

## Where to Learn

1. **LSP Specification - CompletionItem:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#completionItem

2. **CompletionItemKind values:**
   - Full list in spec

3. **Editor behavior:**
   - Test how different fields affect the UI

## Practice Exercise

Build and test completion items:

```c
void test_completion_items(void) {
    JsonValue *item = create_completion_item("hello", 0);
    
    assert(strcmp(json_get_string(item, "label"), "hello") == 0);
    assert(json_get_int(item, "kind") == COMPLETION_TEXT);
    assert(strcmp(json_get_string(item, "sortText"), "000000") == 0);
    
    char *json = json_stringify(item);
    printf("Item: %s\n", json);
    free(json);
    
    json_free(item);
    printf("test_completion_items: PASS\n");
}
```

## Connection to Project

Well-structured completion items ensure:
- Users see clean, consistent suggestions
- Ordering matches engine relevance
- Selection inserts the right text

The engine provides the words; you provide the presentation.
