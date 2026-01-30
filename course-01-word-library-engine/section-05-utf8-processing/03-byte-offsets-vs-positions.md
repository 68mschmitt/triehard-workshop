# Byte Offsets vs Positions

## Why You Need This

Your requirements say: "Return detected issues with precise byte-level spans" and "The engine must NOT depend on editor-specific coordinate systems (line/column)."

This is crucial for clean separation between engine and adapters.

## What to Learn

### The Coordinate Systems

**Byte offset:** Position in raw bytes from start of text
```
Text: "Hello\nWorld"
       H e l l o \n W o r l  d
Byte:  0 1 2 3 4  5 6 7 8 9 10
```

**Line/Column:** Human-readable position
```
Line 1: "Hello"
Line 2: "World"
"W" is at line 2, column 1
```

**Character offset:** Position in Unicode codepoints
```
Text: "café"
       c  a  f  é
Char:  0  1  2  3
Byte:  0  1  2  3,4
```

### Why Byte Offsets for Your Engine

1. **Unambiguous:** Every byte has exactly one offset
2. **Fast:** No scanning to find position
3. **Portable:** Same on all systems
4. **UTF-8 friendly:** Works regardless of encoding details

### Why NOT Line/Column in Engine

1. **Line delimiter varies:** `\n`, `\r\n`, `\r`
2. **Column definition varies:** Bytes? Characters? Display width?
3. **Tab handling varies:** 4 spaces? 8? Variable?
4. **Couples engine to editor:** Different editors have different rules

### LSP Position Conversion

LSP uses line/character positions. Your adapter converts:

```c
// Adapter code (NOT in engine)
typedef struct {
    uint32_t line;       // 0-based
    uint32_t character;  // UTF-16 code units from line start
} LSPPosition;

typedef struct {
    LSPPosition start;
    LSPPosition end;
} LSPRange;

// Convert byte span to LSP range
LSPRange byte_span_to_lsp(const char *text, size_t start, size_t end) {
    LSPRange range = {0};
    size_t line = 0;
    size_t line_start = 0;
    
    // Find start position
    for (size_t i = 0; i < start; i++) {
        if (text[i] == '\n') {
            line++;
            line_start = i + 1;
        }
    }
    range.start.line = line;
    range.start.character = utf16_length(text + line_start, start - line_start);
    
    // Continue to find end position
    for (size_t i = start; i < end; i++) {
        if (text[i] == '\n') {
            line++;
            line_start = i + 1;
        }
    }
    range.end.line = line;
    range.end.character = utf16_length(text + line_start, end - line_start);
    
    return range;
}

// UTF-16 length (LSP uses UTF-16 code units)
size_t utf16_length(const char *utf8, size_t byte_len) {
    size_t count = 0;
    const unsigned char *p = (const unsigned char *)utf8;
    const unsigned char *end = p + byte_len;
    
    while (p < end) {
        int char_len = utf8_char_length(*p);
        // Characters outside BMP need 2 UTF-16 code units
        if (char_len == 4) count += 2;
        else count += 1;
        p += char_len;
    }
    return count;
}
```

### Example Flow

```
User types: "The quikc brown fox"
                 ^^^^^
                 Unknown word

1. Engine tokenizes, finds "quikc" at bytes [4, 9]
2. Engine checks dictionary: "quikc" not found
3. Engine returns: {span: {start: 4, end: 9}, word: "quikc"}
4. LSP adapter converts: {start: {line: 0, char: 4}, end: {line: 0, char: 9}}
5. Editor highlights "quikc"
```

### Multi-line Example

```
Text: "Hello\nquikc world"
       H e l l o \n q  u  i  k  c     w  o  r  l  d
Byte:  0 1 2 3 4  5 6  7  8  9 10 11 12 13 14 15 16

"quikc" span: {start: 6, end: 11}

LSP conversion:
- Byte 6 is after newline, so line 1, char 0
- Result: {start: {line: 1, char: 0}, end: {line: 1, char: 5}}
```

### Why UTF-16 in LSP?

Historical reasons (JavaScript, Java). Your engine doesn't care - it's the adapter's problem.

Most characters: 1 UTF-8 char = 1 UTF-16 unit
Emoji and rare chars: 1 UTF-8 char (4 bytes) = 2 UTF-16 units

### The Clean Separation

```
┌─────────────────────────────────────────────────────┐
│                    LSP Adapter                       │
│  - Receives text from editor                        │
│  - Converts LSP positions to byte offsets           │
│  - Calls engine with byte offsets                   │
│  - Converts engine results to LSP positions         │
└─────────────────────────────────────────────────────┘
                          │
                    (byte offsets only)
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                      Engine                          │
│  - Knows nothing about lines or columns             │
│  - Works purely with byte offsets                   │
│  - Returns spans as byte offset pairs               │
└─────────────────────────────────────────────────────┘
```

## Where to Learn

1. **LSP Specification** - Position encoding details
2. **"UTF-8 Everywhere" manifesto** - Why byte offsets are cleaner
3. **VSCode source code** - How they handle position conversion
4. **neovim LSP docs** - Client-side position handling

## Practice Exercise

Implement position conversion (adapter-level code):

```c
typedef struct { size_t start, end; } ByteSpan;
typedef struct { uint32_t line, character; } Position;
typedef struct { Position start, end; } Range;

Range byte_span_to_range(const char *text, ByteSpan span);
ByteSpan range_to_byte_span(const char *text, Range range);
```

Test with:
```c
// "Hello\nWorld"
// "World" is bytes [6, 11], should be line 1, chars 0-5
```

Verify round-trip: span → range → span = original span

## Connection to Project

Your engine returns byte spans. Future you (or someone else) writes the LSP adapter that converts these to editor positions. By keeping the engine position-agnostic, you can:
- Use it with any editor
- Test without editor integration
- Change editors without changing engine
