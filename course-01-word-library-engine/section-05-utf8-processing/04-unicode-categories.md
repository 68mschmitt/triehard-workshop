# Unicode Categories

## Why You Need This

Your requirements say tokenization should be "configurable or extensible." Knowing about Unicode categories helps you make principled decisions about what constitutes a "word character."

## What to Learn

### What Are Unicode Categories?

Every Unicode codepoint belongs to a "General Category":

| Category | Code | Examples |
|----------|------|----------|
| Letter, uppercase | Lu | A, B, C, Α, Β, Γ |
| Letter, lowercase | Ll | a, b, c, α, β, γ |
| Letter, titlecase | Lt | ǅ, ǈ, ǋ |
| Letter, modifier | Lm | ʰ, ʱ, ʲ |
| Letter, other | Lo | 中, 文, あ, ア |
| Mark, nonspacing | Mn | ̀, ́, ̂ (combining accents) |
| Mark, spacing combining | Mc | ः, ौ |
| Number, decimal digit | Nd | 0-9, ٠-٩, ০-৯ |
| Number, letter | Nl | Ⅰ, Ⅱ, Ⅲ |
| Punctuation, connector | Pc | _ |
| Punctuation, dash | Pd | -, – |
| Separator, space | Zs | (space), (non-breaking space) |

### Why This Matters for Tokenization

"Word characters" could mean:
1. **Just ASCII letters:** `[A-Za-z]`
2. **All letters:** Lu + Ll + Lt + Lm + Lo
3. **Letters + marks:** Above + Mn + Mc (for accented characters)
4. **Identifier-like:** Letters + marks + Nd + Pc

For note-taking, option 3 is usually right.

### Practical Approach: ASCII Fast Path

Full Unicode lookup is expensive. Most text is ASCII:

```c
bool is_word_char(const unsigned char *p) {
    unsigned char c = *p;
    
    // Fast path: ASCII
    if (c < 0x80) {
        return (c >= 'A' && c <= 'Z') ||
               (c >= 'a' && c <= 'z');
    }
    
    // Slow path: non-ASCII
    return is_unicode_letter(p);
}
```

### Simple Unicode Letter Detection

Without a full Unicode database, approximate:

```c
bool is_unicode_letter(const unsigned char *p) {
    // If it's a valid UTF-8 start byte (not continuation), 
    // assume it's a letter for simplicity
    unsigned char c = *p;
    
    // Skip continuation bytes (10xxxxxx)
    if ((c & 0xC0) == 0x80) return false;
    
    // Common non-letter ranges in first byte:
    // E2 80 xx - General punctuation
    // E2 84 xx - Letterlike symbols (some letters, some not)
    // EF BB BF - BOM
    
    // For simplicity: if it's multi-byte and not obviously punctuation,
    // treat as letter. Imperfect but practical.
    return c >= 0xC0;
}
```

### Using a Lookup Table

For precise categorization, use precomputed tables:

```c
// Generated from Unicode database
static const uint8_t category_table[...] = { ... };

typedef enum {
    CAT_LETTER = 1,
    CAT_MARK = 2,
    CAT_NUMBER = 4,
    CAT_PUNCTUATION = 8,
    // ...
} CategoryFlag;

uint8_t get_category(uint32_t codepoint) {
    if (codepoint < TABLE_SIZE) {
        return category_table[codepoint];
    }
    // Default for unknown
    return 0;
}

bool is_word_char_precise(const unsigned char *p) {
    uint32_t codepoint = decode_utf8(p);
    uint8_t cat = get_category(codepoint);
    return (cat & (CAT_LETTER | CAT_MARK)) != 0;
}
```

Table generation is a one-time task using Unicode data files.

### Grapheme Clusters (Advanced)

A "character" can span multiple codepoints:
```
"é" can be:
- U+00E9 (single codepoint, precomposed)
- U+0065 U+0301 (e + combining acute accent)

"👨‍👩‍👧‍👦" (family emoji) is:
- U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466
- 7 codepoints, 1 visual character!
```

For word tokenization, you usually don't need grapheme awareness - letter boundaries suffice.

### External Libraries

If you need full Unicode support:

**ICU (International Components for Unicode):**
- Industrial strength
- Large dependency
- Correct handling of everything

**utf8proc:**
- Lighter weight
- Good for normalization and category lookup
- C library

**Roll your own:**
- Subset of categories you need
- Smallest footprint
- Adequate for most note-taking use cases

### Recommendation for Your Project

1. **Start simple:** ASCII letters + assume non-ASCII is letters
2. **Handle combining marks:** Don't break in middle of combining sequence
3. **Add precision later:** If users report issues with specific scripts

```c
bool is_word_char(const unsigned char *p) {
    unsigned char c = *p;
    
    // ASCII letters
    if ((c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z')) return true;
    
    // ASCII non-letters (fast rejection)
    if (c < 0x80) return false;
    
    // Multi-byte UTF-8: assume letter for now
    // Skip continuation bytes
    if ((c & 0xC0) == 0x80) return false;
    
    // Start of multi-byte sequence: probably a letter
    return true;
}
```

This handles "café", "naïve", "北京" correctly. It might incorrectly include some punctuation, but that's rare in practice.

## Where to Learn

1. **Unicode Standard** - Chapter 4, Character Properties
2. **Unicode data files** - UnicodeData.txt has all categories
3. **utf8proc source** - See how they handle categories
4. **"Programming with Unicode"** - Good book on the topic

## Practice Exercise

Implement category detection:

```c
bool is_letter(const char *p);      // Letter categories (L*)
bool is_letter_or_mark(const char *p);  // L* + M*
```

Test with:
- ASCII: "Hello" → all letters
- Accented: "café" → all letters (é is one character)
- Chinese: "中文" → all letters
- Mixed: "Hello!" → 5 letters, 1 non-letter

## Connection to Project

Unicode awareness makes your tokenizer handle international text gracefully. Users writing notes in French, German, or Chinese get proper word detection, not garbage.
