# Section 5 Project: UTF-8 Tokenizer

## Objective

Build a UTF-8 aware tokenizer that extracts words from text with byte-level spans. This is the front-end to your text checking feature.

## Requirements

1. **UTF-8 correct** - Never breaks multi-byte characters
2. **Byte spans** - Reports exact byte offsets for each token
3. **Configurable** - Basic options for what constitutes a word
4. **Efficient** - Single pass through text
5. **Memory safe** - No allocations (uses caller-provided buffers)

## Specification

### Public API (`include/tokenizer.h`)

```c
// Span in byte offsets
typedef struct {
    size_t start;  // Inclusive
    size_t end;    // Exclusive
} Span;

// Token with span and convenience pointer
typedef struct {
    Span span;
    const char *text;  // Points into original string
    size_t length;     // Byte length (end - start)
} Token;

// Configuration
typedef struct {
    bool include_apostrophes;  // "don't" as one token
    bool include_hyphens;      // "well-known" as one token
    int min_length;            // Skip tokens shorter than this
} TokenizerConfig;

// Default configuration
TokenizerConfig tokenizer_default_config(void);

// Tokenize text into tokens array
// Returns number of tokens found
size_t tokenize(const char *text, const TokenizerConfig *config,
                Token *tokens, size_t max_tokens);

// Callback-based tokenization (no buffer needed)
typedef bool (*TokenCallback)(const Token *token, void *userdata);
void tokenize_foreach(const char *text, const TokenizerConfig *config,
                      TokenCallback cb, void *userdata);

// UTF-8 utilities (also useful elsewhere)
bool utf8_is_valid(const char *str);
size_t utf8_char_length(unsigned char byte);
const char *utf8_next(const char *p);  // Advance to next character
size_t utf8_strlen(const char *str);   // Character count
```

### Internal Functions

```c
// src/tokenizer.c

// Check if position is start of word character
static bool is_word_char(const char *p, const TokenizerConfig *config);

// Find end of current word
static const char *find_word_end(const char *start, const TokenizerConfig *config);

// Skip to next potential word start
static const char *skip_non_word(const char *p);
```

### Behavior Specifications

**tokenize:**
- Returns actual token count (may be less than max_tokens)
- Tokens are in document order
- Token `text` pointers are into original string (no copies)
- Does not modify input text
- Empty string returns 0 tokens

**is_word_char:**
- ASCII letters always count
- Multi-byte UTF-8 sequences count (simplified: assume letters)
- Apostrophe counts if `include_apostrophes` and surrounded by letters
- Hyphen counts if `include_hyphens` and surrounded by letters

**utf8_is_valid:**
- Returns false for invalid UTF-8 sequences
- Checks continuation bytes, overlong encodings (basic checks)

## Test Cases

Create `test/test_tokenizer.c`:

```c
void test_simple_ascii(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("Hello world", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].span.start == 0);
    assert(tokens[0].span.end == 5);
    assert(tokens[0].length == 5);
    assert(strncmp(tokens[0].text, "Hello", 5) == 0);
    
    assert(tokens[1].span.start == 6);
    assert(tokens[1].span.end == 11);
}

void test_punctuation(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("Hello, world!", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].span.start == 0);
    assert(tokens[0].span.end == 5);  // "Hello" not "Hello,"
}

void test_apostrophes(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    cfg.include_apostrophes = true;
    
    size_t count = tokenize("don't stop", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].length == 5);  // "don't"
    assert(strncmp(tokens[0].text, "don't", 5) == 0);
}

void test_apostrophes_disabled(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    cfg.include_apostrophes = false;
    
    size_t count = tokenize("don't stop", &cfg, tokens, 10);
    
    assert(count == 3);  // "don", "t", "stop"
}

void test_utf8_basic(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    // "café" is 5 bytes: c(1) a(1) f(1) é(2)
    size_t count = tokenize("café", &cfg, tokens, 10);
    
    assert(count == 1);
    assert(tokens[0].span.start == 0);
    assert(tokens[0].span.end == 5);  // Bytes, not characters!
    assert(tokens[0].length == 5);
}

void test_utf8_sentence(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("Bon café!", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].span.start == 0);
    assert(tokens[0].span.end == 3);   // "Bon"
    assert(tokens[1].span.start == 4);
    assert(tokens[1].span.end == 9);   // "café" (5 bytes)
}

void test_utf8_multibyte(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    // Chinese characters: each is 3 bytes
    size_t count = tokenize("中文", &cfg, tokens, 10);
    
    assert(count == 1);
    assert(tokens[0].length == 6);  // 2 characters × 3 bytes
}

void test_mixed_script(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("Hello 世界", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].length == 5);  // "Hello"
    assert(tokens[1].length == 6);  // "世界"
}

void test_hyphens(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    cfg.include_hyphens = true;
    
    size_t count = tokenize("well-known fact", &cfg, tokens, 10);
    
    assert(count == 2);
    assert(tokens[0].length == 10);  // "well-known"
    assert(strncmp(tokens[0].text, "well-known", 10) == 0);
}

void test_min_length(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    cfg.min_length = 3;
    
    size_t count = tokenize("I am a test", &cfg, tokens, 10);
    
    assert(count == 1);  // Only "test" (I, am, a are too short)
}

void test_empty_string(void) {
    Token tokens[10];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("", &cfg, tokens, 10);
    assert(count == 0);
    
    count = tokenize("   ", &cfg, tokens, 10);
    assert(count == 0);
}

void test_max_tokens(void) {
    Token tokens[2];
    TokenizerConfig cfg = tokenizer_default_config();
    
    size_t count = tokenize("one two three four", &cfg, tokens, 2);
    
    assert(count == 2);  // Limited by buffer
    assert(strncmp(tokens[0].text, "one", 3) == 0);
    assert(strncmp(tokens[1].text, "two", 3) == 0);
}

void test_foreach(void) {
    TokenizerConfig cfg = tokenizer_default_config();
    int count = 0;
    
    tokenize_foreach("one two three", &cfg, 
        (TokenCallback)^(const Token *t, void *ctx) {
            (*(int *)ctx)++;
            return true;
        }, &count);
    
    assert(count == 3);
}

void test_utf8_validation(void) {
    assert(utf8_is_valid("Hello"));
    assert(utf8_is_valid("café"));
    assert(utf8_is_valid("中文"));
    assert(utf8_is_valid(""));
    
    // Invalid sequences
    assert(!utf8_is_valid("\x80"));          // Orphan continuation
    assert(!utf8_is_valid("\xC0\xAF"));      // Overlong
    assert(!utf8_is_valid("Hello\xFF"));     // Invalid byte
}

void test_utf8_strlen(void) {
    assert(utf8_strlen("Hello") == 5);
    assert(utf8_strlen("café") == 4);       // 4 characters, 5 bytes
    assert(utf8_strlen("中文") == 2);        // 2 characters, 6 bytes
    assert(utf8_strlen("") == 0);
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_tokenizer
Running tokenizer tests...
test_simple_ascii: PASS
test_punctuation: PASS
test_apostrophes: PASS
test_utf8_basic: PASS
test_utf8_sentence: PASS
...
All tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_tokenizer
...
All heap blocks were freed -- no leaks are possible
```

### Performance
Tokenize 1MB of text in < 100ms

## Stretch Goals

1. **More apostrophe cases** - Handle "it's", "they're", but not "'twas" (leading apostrophe)
2. **Number handling** - Option to extract numbers as tokens
3. **Email/URL detection** - Skip or special-handle these patterns
4. **Line-aware mode** - Option to track line numbers for each token

## Implementation Hints

### UTF-8 Character Length
```c
size_t utf8_char_length(unsigned char byte) {
    if ((byte & 0x80) == 0x00) return 1;
    if ((byte & 0xE0) == 0xC0) return 2;
    if ((byte & 0xF0) == 0xE0) return 3;
    if ((byte & 0xF8) == 0xF0) return 4;
    return 1;  // Invalid, consume one byte
}
```

### Word Character Check
```c
static bool is_word_char(const char *p, const TokenizerConfig *cfg) {
    unsigned char c = *(unsigned char *)p;
    
    // ASCII letters
    if ((c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z')) return true;
    
    // ASCII non-letters: quick reject
    if (c < 0x80) {
        // Maybe apostrophe or hyphen
        if (c == '\'' && cfg->include_apostrophes) {
            // Check if surrounded by letters
            return is_alpha(p[-1]) && is_alpha(p[1]);
        }
        if (c == '-' && cfg->include_hyphens) {
            return is_alpha(p[-1]) && is_alpha(p[1]);
        }
        return false;
    }
    
    // Multi-byte UTF-8: assume it's a letter
    // Skip continuation bytes
    if ((c & 0xC0) == 0x80) return false;
    
    return true;
}
```

### Main Tokenization Loop
```c
size_t tokenize(const char *text, const TokenizerConfig *cfg,
                Token *tokens, size_t max) {
    size_t count = 0;
    const char *p = text;
    
    while (*p && count < max) {
        // Skip non-word characters
        while (*p && !is_word_char(p, cfg)) {
            p += utf8_char_length(*p);
        }
        
        if (!*p) break;
        
        // Found word start
        const char *start = p;
        
        // Find word end
        while (*p && is_word_char(p, cfg)) {
            p += utf8_char_length(*p);
        }
        
        size_t len = p - start;
        if (len >= cfg->min_length) {
            tokens[count].span.start = start - text;
            tokens[count].span.end = p - text;
            tokens[count].text = start;
            tokens[count].length = len;
            count++;
        }
    }
    
    return count;
}
```

## What You'll Learn

By completing this project:
- UTF-8 encoding and decoding
- Text processing without character counting
- Byte offset arithmetic
- Configurable parsing

## Next Section Preview

Section 6 adds persistence - saving and loading your word dictionary. You'll implement a simple, stable file format that won't corrupt data on crash.
