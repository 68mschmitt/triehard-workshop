# Opaque Types & API Design

## Why You Need This

Your requirements demand "clean separation between logic and integration." Opaque types are HOW you achieve this in C. Users of your API see a pointer - they can't peek inside, can't depend on internals, can't break when you refactor.

## What to Learn

### The Opaque Pointer Pattern

**In the header (public):**
```c
// wordlib.h
typedef struct WordLib WordLib;  // Forward declaration only!

WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *ctx);
int wordlib_add_word(WordLib *ctx, const char *word);
```

**In the source (private):**
```c
// wordlib.c
struct WordLib {
    HashSet *words;
    Trie *trie;
    BKTree *bktree;
    size_t word_count;
    // Add fields freely - ABI won't break
};
```

Users can't do `ctx->word_count` - they only have the incomplete type!

### Why This Matters

1. **Implementation freedom:** Change internals without recompiling users
2. **Encapsulation:** Prevent misuse of internal state
3. **Clear contracts:** API is the only way to interact
4. **Testability:** Easy to mock or stub

### Context Objects vs Global State

**Bad (global state):**
```c
static HashSet *g_words;  // Hidden global

void init(void);
void add_word(const char *word);  // Which dictionary? The global one...
```

Problems: Not thread-safe, can't have multiple instances, hard to test.

**Good (explicit context):**
```c
WordLib *ctx1 = wordlib_create();
WordLib *ctx2 = wordlib_create();  // Two independent instances!
wordlib_add_word(ctx1, "hello");
wordlib_add_word(ctx2, "world");
```

Your requirements explicitly say "no global mutable state" - this is how.

### Accessor Functions

When users DO need internal info, provide explicit access:

```c
// Header
size_t wordlib_count(const WordLib *ctx);
bool wordlib_is_empty(const WordLib *ctx);

// Implementation
size_t wordlib_count(const WordLib *ctx) {
    return ctx->word_count;
}
```

### Error Handling in Opaque APIs

Two common patterns:

**Return codes:**
```c
typedef enum {
    WORDLIB_OK = 0,
    WORDLIB_ERR_NULL,
    WORDLIB_ERR_MEMORY,
    WORDLIB_ERR_INVALID
} WordLibError;

WordLibError wordlib_add_word(WordLib *ctx, const char *word);
```

**Structured results:**
```c
typedef struct {
    bool success;
    const char *error_msg;
    // result data...
} WordLibResult;

WordLibResult wordlib_check_text(WordLib *ctx, const char *text);
```

Your project will likely use both: simple return codes for simple operations, structured results for complex queries.

### Header File Best Practices

```c
#ifndef WORDLIB_H
#define WORDLIB_H

#include <stddef.h>   // size_t
#include <stdbool.h>  // bool

#ifdef __cplusplus
extern "C" {
#endif

// Opaque type
typedef struct WordLib WordLib;

// Lifecycle
WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *ctx);

// Operations
int wordlib_add_word(WordLib *ctx, const char *word);
bool wordlib_contains(const WordLib *ctx, const char *word);

#ifdef __cplusplus
}
#endif

#endif // WORDLIB_H
```

## Where to Learn

1. **GLib Source Code** - Excellent example of opaque API design in C
2. **SQLite Source** - `sqlite3*` is the classic opaque context
3. **"C Interfaces and Implementations" by David Hanson** - Deep dive on this pattern

## Practice Exercise

Design an opaque API for a simple "counter" object:

```c
// counter.h - Write this
typedef struct Counter Counter;

Counter *counter_create(int initial);
void counter_destroy(Counter *c);
void counter_increment(Counter *c);
void counter_decrement(Counter *c);
int counter_get(const Counter *c);
void counter_reset(Counter *c, int value);
```

Implement it in `counter.c` with the struct definition hidden.

**Bonus:** Add error handling - what if `NULL` is passed?

## Connection to Project

Your entire engine will be exposed through an opaque `WordLib` type. The CLI adapter won't know (or care) whether you use a trie or a hash table inside. Future LSP adapter won't need changes if you optimize internals. This is the foundation of "adapter layer doesn't depend on implementation details."
