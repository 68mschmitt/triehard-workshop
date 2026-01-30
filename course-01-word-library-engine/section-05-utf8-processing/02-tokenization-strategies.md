# Tokenization Strategies

## Why You Need This

Your requirements say: "Tokenize the text into word units" and "Define a consistent tokenization strategy." Editors send you raw text. You need to extract words from it reliably.

## What to Learn

### What Is a Word?

Seems obvious until you think about it:

```
"Hello, world!"           → ["Hello", "world"]
"don't"                   → ["don't"] or ["don", "t"]?
"state-of-the-art"        → One word or four?
"user@example.com"        → Word? Email?
"3.14159"                 → Number? Word?
"C++"                     → Programming language name
"naïve"                   → Unicode word
"北京"                    → Chinese word (no spaces!)
```

There's no universal answer. You define your rules.

### Simple Strategy: Alphabetic Runs

```c
bool is_word_char(char c) {
    return (c >= 'a' && c <= 'z') ||
           (c >= 'A' && c <= 'Z');
}

void tokenize_simple(const char *text, TokenCallback cb) {
    const char *start = NULL;
    
    for (const char *p = text; ; p++) {
        bool is_word = *p && is_word_char(*p);
        
        if (is_word && !start) {
            start = p;  // Word begins
        } else if (!is_word && start) {
            cb(start, p - start);  // Word ends
            start = NULL;
        }
        
        if (!*p) break;
    }
}
```

**Pros:** Dead simple
**Cons:** Breaks on "don't", ignores numbers, no Unicode

### Extended Strategy: Include Apostrophes

```c
bool is_word_char(const char *p) {
    char c = *p;
    if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')) return true;
    
    // Apostrophe inside word: "don't", "it's"
    if (c == '\'' && p[-1] && is_alpha(p[-1]) && p[1] && is_alpha(p[1])) {
        return true;
    }
    
    return false;
}
```

Now "don't" is one token.

### UTF-8 Aware Strategy

```c
bool is_word_char_utf8(const char *p) {
    unsigned char c = *p;
    
    // ASCII letters
    if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')) return true;
    
    // UTF-8 multi-byte sequences starting with 110xxxxx or higher
    // (Assume non-ASCII characters are letters - simplified)
    if (c >= 0xC0) return true;
    
    // Apostrophe handling...
    
    return false;
}
```

**Better:** Use Unicode categories (Letter, Mark, etc.)

### Reporting Byte Spans

Your requirements: "Return detected issues with precise byte-level spans."

```c
typedef struct {
    size_t start;  // Byte offset
    size_t end;    // Byte offset (exclusive)
} Span;

typedef struct {
    Span span;
    const char *text;  // Pointer into original string
} Token;

void tokenize(const char *text, Token *tokens, size_t *count) {
    const char *base = text;
    const char *start = NULL;
    size_t i = 0;
    
    for (const char *p = text; ; p++) {
        bool is_word = *p && is_word_char_utf8(p);
        
        if (is_word && !start) {
            start = p;
        } else if (!is_word && start) {
            tokens[i].span.start = start - base;
            tokens[i].span.end = p - base;
            tokens[i].text = start;
            i++;
            start = NULL;
        }
        
        if (!*p) break;
    }
    
    *count = i;
}
```

### Handling Edge Cases

**Hyphens:**
```c
// Option 1: Split on hyphens
"state-of-the-art" → ["state", "of", "the", "art"]

// Option 2: Keep hyphenated compounds
"state-of-the-art" → ["state-of-the-art"]

// Option 3: Keep simple hyphens, split complex
"well-known" → ["well-known"]
"this-is-too-many" → ["this", "is", "too", "many"]
```

**Numbers:**
```c
// Option 1: Skip numbers entirely
"test123" → ["test"]

// Option 2: Include digits in words
"test123" → ["test123"]

// Option 3: Separate numbers as tokens
"test123abc" → ["test", "123", "abc"]
```

**Underscores (for code):**
```c
// For code-like text
"my_variable_name" → ["my_variable_name"]  // One identifier
```

### Configurable Tokenizer

```c
typedef struct {
    bool include_apostrophes;
    bool include_hyphens;
    bool include_digits;
    bool include_underscores;
    int min_word_length;
} TokenizerConfig;

void tokenize_with_config(const char *text, const TokenizerConfig *config,
                          Token *tokens, size_t *count);
```

For note-taking (your use case), a good default:
- Include apostrophes: yes
- Include hyphens: simple ones (word-word)
- Include digits: no (numbers aren't "words")
- Include underscores: no
- Min length: 1

## Where to Learn

1. **Unicode Standard Annex #29** - Text segmentation rules
2. **ICU Library documentation** - Industrial-strength tokenization
3. **Lucene analyzers** - Search engine tokenization
4. **"Introduction to Information Retrieval"** - Chapter on tokenization

## Practice Exercise

Implement a configurable tokenizer:

```c
typedef struct {
    size_t start;
    size_t end;
} Span;

size_t tokenize(const char *text, Span *spans, size_t max_spans);
```

Test with:
```c
"Hello, world!"           → [(0,5), (7,12)]
"don't stop"              → [(0,5), (6,10)]
"café au lait"            → [(0,5), (6,8), (9,13)]  // Note: café is 5 bytes
"test123 foo"             → [(0,4), (8,11)] or [(0,7), (8,11)]
```

Verify byte offsets by extracting substrings.

## Connection to Project

Your tokenizer is the front line:
1. Editor sends text
2. Tokenizer extracts words with byte spans
3. Each word checked against dictionary
4. Unknown words reported with spans

Byte spans let the LSP adapter convert to editor positions without your engine knowing about line numbers.
