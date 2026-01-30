# Byte Span to LSP Position

## Why You Need This

Your requirements say: "Convert engine byte spans into LSP positions." Your engine reports word locations as byte offsets. LSP uses line/character positions with UTF-16 encoding. This conversion is crucial for accurate squiggle placement.

## What to Learn

### The Coordinate Systems

**Engine output (byte offsets):**
```c
typedef struct {
    size_t start;  // Byte offset from text start
    size_t end;    // Byte offset, exclusive
} ByteSpan;

// "quikc" in "The quikc brown fox"
// ByteSpan { .start = 4, .end = 9 }
```

**LSP requirement (line/character):**
```json
{
  "start": { "line": 0, "character": 4 },
  "end": { "line": 0, "character": 9 }
}
```

### The Conversion Challenge

LSP uses **UTF-16 code units** for the character offset, not bytes:

| Character | UTF-8 bytes | UTF-16 code units |
|-----------|-------------|-------------------|
| `a` | 1 | 1 |
| `é` | 2 | 1 |
| `你` | 3 | 1 |
| `😀` | 4 | 2 (surrogate pair) |

For ASCII text, bytes = characters. For international text, they differ.

### Basic Conversion (ASCII)

For ASCII-only text, conversion is simpler:

```c
typedef struct {
    uint32_t line;
    uint32_t character;
} Position;

typedef struct {
    Position start;
    Position end;
} Range;

// Convert byte offset to line/character (ASCII only)
Position byte_to_position_ascii(const char *text, size_t offset) {
    Position pos = { .line = 0, .character = 0 };
    
    for (size_t i = 0; i < offset && text[i]; i++) {
        if (text[i] == '\n') {
            pos.line++;
            pos.character = 0;
        } else {
            pos.character++;
        }
    }
    
    return pos;
}

Range byte_span_to_range_ascii(const char *text, size_t start, size_t end) {
    return (Range){
        .start = byte_to_position_ascii(text, start),
        .end = byte_to_position_ascii(text, end)
    };
}
```

### Full UTF-8 to UTF-16 Conversion

```c
// Get UTF-8 character byte length from first byte
int utf8_char_length(unsigned char c) {
    if ((c & 0x80) == 0x00) return 1;      // 0xxxxxxx
    if ((c & 0xE0) == 0xC0) return 2;      // 110xxxxx
    if ((c & 0xF0) == 0xE0) return 3;      // 1110xxxx
    if ((c & 0xF8) == 0xF0) return 4;      // 11110xxx
    return 1;  // Invalid, treat as single byte
}

// Get UTF-16 length of a UTF-8 character
int utf8_to_utf16_length(const char *utf8, int utf8_len) {
    // 4-byte UTF-8 = characters outside BMP = 2 UTF-16 code units
    return (utf8_len == 4) ? 2 : 1;
}

Position byte_to_position(const char *text, size_t offset) {
    Position pos = { .line = 0, .character = 0 };
    const unsigned char *p = (const unsigned char *)text;
    size_t i = 0;
    
    while (i < offset && p[i]) {
        if (p[i] == '\n') {
            pos.line++;
            pos.character = 0;
            i++;
        } else {
            int char_len = utf8_char_length(p[i]);
            int utf16_len = utf8_to_utf16_length((const char *)p + i, char_len);
            pos.character += utf16_len;
            i += char_len;
        }
    }
    
    return pos;
}

Range byte_span_to_range(const char *text, size_t start, size_t end) {
    return (Range){
        .start = byte_to_position(text, start),
        .end = byte_to_position(text, end)
    };
}
```

### Optimization: Incremental Conversion

For multiple spans in the same document, avoid rescanning from the beginning:

```c
typedef struct {
    const char *text;
    size_t text_len;
    
    // Current position
    size_t byte_offset;
    uint32_t line;
    uint32_t character;
} PositionTracker;

PositionTracker *tracker_create(const char *text, size_t len) {
    PositionTracker *t = calloc(1, sizeof(PositionTracker));
    t->text = text;
    t->text_len = len;
    return t;
}

Position tracker_to_position(PositionTracker *t, size_t target_offset) {
    // If target is before current, restart
    if (target_offset < t->byte_offset) {
        t->byte_offset = 0;
        t->line = 0;
        t->character = 0;
    }
    
    // Advance to target
    const unsigned char *text = (const unsigned char *)t->text;
    while (t->byte_offset < target_offset && t->byte_offset < t->text_len) {
        if (text[t->byte_offset] == '\n') {
            t->line++;
            t->character = 0;
            t->byte_offset++;
        } else {
            int char_len = utf8_char_length(text[t->byte_offset]);
            int utf16_len = utf8_to_utf16_length(t->text + t->byte_offset, char_len);
            t->character += utf16_len;
            t->byte_offset += char_len;
        }
    }
    
    return (Position){ .line = t->line, .character = t->character };
}
```

### Pre-Computing Line Starts

For large documents with many diagnostics:

```c
typedef struct {
    size_t *line_starts;   // Byte offset of each line start
    size_t line_count;
    size_t capacity;
} LineIndex;

LineIndex *build_line_index(const char *text, size_t len) {
    LineIndex *idx = malloc(sizeof(LineIndex));
    idx->capacity = 1024;
    idx->line_starts = malloc(idx->capacity * sizeof(size_t));
    idx->line_count = 0;
    
    // Line 0 starts at offset 0
    idx->line_starts[idx->line_count++] = 0;
    
    for (size_t i = 0; i < len; i++) {
        if (text[i] == '\n' && i + 1 < len) {
            if (idx->line_count >= idx->capacity) {
                idx->capacity *= 2;
                idx->line_starts = realloc(idx->line_starts, 
                                          idx->capacity * sizeof(size_t));
            }
            idx->line_starts[idx->line_count++] = i + 1;
        }
    }
    
    return idx;
}

// Fast line lookup with binary search
uint32_t offset_to_line(LineIndex *idx, size_t offset) {
    size_t lo = 0, hi = idx->line_count;
    
    while (lo < hi) {
        size_t mid = (lo + hi) / 2;
        if (idx->line_starts[mid] <= offset) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    
    return (uint32_t)(lo - 1);
}
```

### Testing Coordinate Conversion

```c
void test_conversion(void) {
    // ASCII
    const char *ascii = "Hello\nWorld";
    Position p = byte_to_position(ascii, 6);  // 'W'
    assert(p.line == 1 && p.character == 0);
    
    p = byte_to_position(ascii, 8);  // 'r'
    assert(p.line == 1 && p.character == 2);
    
    // UTF-8 with multi-byte characters
    const char *utf8 = "café\ntest";  // é is 2 bytes
    p = byte_to_position(utf8, 5);  // '\n' (bytes: c=1, a=1, f=1, é=2)
    assert(p.line == 0 && p.character == 4);  // 4 characters before newline
    
    p = byte_to_position(utf8, 6);  // 't' on line 2
    assert(p.line == 1 && p.character == 0);
    
    printf("test_conversion: PASS\n");
}
```

## Where to Learn

1. **UTF-8 encoding details:** Course 1 section on UTF-8
2. **UTF-16 surrogate pairs:** Wikipedia "UTF-16"
3. **LSP position encoding:** LSP specification, PositionEncodingKind

## Practice Exercise

Write comprehensive tests for the conversion:

```c
void test_all_conversions(void) {
    // Empty text
    assert(byte_to_position("", 0).line == 0);
    assert(byte_to_position("", 0).character == 0);
    
    // Single line
    Range r = byte_span_to_range("Hello World", 6, 11);
    assert(r.start.line == 0 && r.start.character == 6);
    assert(r.end.line == 0 && r.end.character == 11);
    
    // Multi-line
    const char *text = "Line one\nLine two\nLine three";
    r = byte_span_to_range(text, 14, 17);  // "two"
    assert(r.start.line == 1 && r.start.character == 5);
    assert(r.end.line == 1 && r.end.character == 8);
    
    // With emoji (4-byte UTF-8 = 2 UTF-16 code units)
    const char *emoji = "Hi 😀 there";
    // H=0, i=1, space=2, 😀=3-4 (2 UTF-16), space=5, t=6...
    Position p = byte_to_position(emoji, 8);  // 't' in "there"
    // Bytes: H(1) i(1) space(1) 😀(4) space(1) = 8 bytes to 't'
    assert(p.line == 0);
    assert(p.character == 6);  // 4 chars + 2 for emoji
    
    printf("test_all_conversions: PASS\n");
}
```

## Connection to Project

This conversion bridges engine and editor:

```
Engine: "Unknown word at bytes 4-9"
                ↓
        byte_span_to_range()
                ↓
LSP:    "Unknown word at line 0, chars 4-9"
                ↓
Editor: Squiggle under the right text
```

Get this wrong and squiggles appear in the wrong place. Get it right and your diagnostics are pixel-perfect.
