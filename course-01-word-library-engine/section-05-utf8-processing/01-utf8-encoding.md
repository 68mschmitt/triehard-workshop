# UTF-8 Encoding

## Why You Need This

Your requirements say: "Words shall be stored internally as UTF-8 strings" and "Operate on byte offsets rather than visual columns." The world isn't ASCII. Users write "naïve", "café", and "北京". You need to handle this correctly.

## What to Learn

### ASCII vs UTF-8

**ASCII:** 7 bits, 128 characters, English only
```
'A' = 0x41 = 01000001 (1 byte)
```

**UTF-8:** Variable length, backwards compatible with ASCII, supports all Unicode
```
'A' = 0x41              (1 byte)
'é' = 0xC3 0xA9         (2 bytes)
'€' = 0xE2 0x82 0xAC    (3 bytes)
'𝄞' = 0xF0 0x9D 0x84 0x9E (4 bytes)
```

### The UTF-8 Encoding Scheme

```
Codepoint Range       Byte 1    Byte 2    Byte 3    Byte 4
U+0000   - U+007F     0xxxxxxx
U+0080   - U+07FF     110xxxxx  10xxxxxx
U+0800   - U+FFFF     1110xxxx  10xxxxxx  10xxxxxx
U+10000  - U+10FFFF   11110xxx  10xxxxxx  10xxxxxx  10xxxxxx
```

**Key insight:** First byte tells you the length:
- `0xxxxxxx` → 1 byte (ASCII)
- `110xxxxx` → 2 bytes
- `1110xxxx` → 3 bytes
- `11110xxx` → 4 bytes
- `10xxxxxx` → continuation byte (never starts a character)

### Detecting Character Boundaries

```c
bool is_utf8_start(unsigned char c) {
    // Not a continuation byte (10xxxxxx)
    return (c & 0xC0) != 0x80;
}

int utf8_char_length(unsigned char c) {
    if ((c & 0x80) == 0x00) return 1;  // 0xxxxxxx
    if ((c & 0xE0) == 0xC0) return 2;  // 110xxxxx
    if ((c & 0xF0) == 0xE0) return 3;  // 1110xxxx
    if ((c & 0xF8) == 0xF0) return 4;  // 11110xxx
    return 1;  // Invalid, treat as 1 byte
}
```

### Iterating UTF-8 Strings

**Wrong (breaks multi-byte characters):**
```c
for (int i = 0; str[i]; i++) {
    process_char(str[i]);  // Processes bytes, not characters!
}
```

**Right (character by character):**
```c
const char *p = str;
while (*p) {
    int len = utf8_char_length(*p);
    process_character(p, len);  // Pass start and length
    p += len;
}
```

### Byte Offset vs Character Index

```
String: "café"
         c  a  f  é
Bytes:   0  1  2  3,4
Chars:   0  1  2  3
```

- Byte offset of 'é' = 3
- Character index of 'é' = 3
- Length in bytes = 5
- Length in characters = 4

**Your API should use byte offsets** - they're unambiguous and don't require scanning.

### Counting Characters (When Needed)

```c
size_t utf8_strlen(const char *str) {
    size_t count = 0;
    while (*str) {
        if (is_utf8_start(*str)) count++;
        str++;
    }
    return count;
}
```

### Common Pitfalls

**Truncating mid-character:**
```c
// WRONG: Might cut in middle of UTF-8 sequence
strncpy(buf, str, 10);

// BETTER: Find character boundary
size_t safe_len = find_utf8_boundary(str, 10);
memcpy(buf, str, safe_len);
buf[safe_len] = '\0';
```

**Assuming character == byte:**
```c
// WRONG
char c = str[i];  // Might be continuation byte!

// RIGHT
int len = utf8_char_length(str[i]);
// Work with str[i] through str[i+len-1]
```

**Breaking on printf:**
```c
// Usually fine - printf handles UTF-8
printf("%s\n", "café");  // Works!

// But width specifiers don't work right
printf("%-10s", "café");  // May not align properly
```

### Valid UTF-8 Checking

```c
bool is_valid_utf8(const char *str) {
    const unsigned char *p = (const unsigned char *)str;
    while (*p) {
        int len = utf8_char_length(*p);
        
        // Check continuation bytes
        for (int i = 1; i < len; i++) {
            if ((p[i] & 0xC0) != 0x80) return false;
        }
        
        // Check for overlong encodings (optional, strict)
        // Check for invalid codepoints (optional, strict)
        
        p += len;
    }
    return true;
}
```

## Where to Learn

1. **utf8everywhere.org** - The manifesto, excellent explanation
2. **"The Absolute Minimum Every Developer Must Know About Unicode"** by Joel Spolsky
3. **Wikipedia "UTF-8"** - Detailed encoding table
4. **RFC 3629** - The actual UTF-8 standard

## Practice Exercise

Implement UTF-8 utilities:

```c
bool is_utf8_start(unsigned char c);
int utf8_char_length(unsigned char c);
size_t utf8_strlen(const char *str);
const char *utf8_next(const char *str);  // Advance to next character
bool is_valid_utf8(const char *str);
```

Test with:
- "hello" (ASCII only)
- "café" (2-byte character)
- "中文" (3-byte characters)
- "👍" (4-byte character)

Verify:
- `utf8_strlen("café")` returns 4
- `utf8_strlen("中文")` returns 2
- Iteration visits each character once

## Connection to Project

Your tokenizer needs to:
1. Not break words at multi-byte boundaries
2. Report byte offsets (not character indices)
3. Handle international text gracefully

Getting UTF-8 right means "café" tokenizes as one word, not "caf" + garbage.
