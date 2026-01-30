# API Design Principles

## Why You Need This

Your requirements say: "The API must be stable and suitable for use by multiple adapters." You're building an engine that will be used by CLI, LSP, and potentially other adapters. The API design determines how pleasant (or painful) this integration will be.

## What to Learn

### Principle of Least Surprise

Users should be able to guess how your API works:

```c
// Good: consistent naming
wordlib_add_word(wl, word);
wordlib_remove_word(wl, word);
wordlib_contains_word(wl, word);  // or wordlib_has_word

// Bad: inconsistent
wordlib_add(wl, word);
wordlib_delete_word(wl, word);
wordlib_lookup(wl, word);
```

### Consistent Parameter Order

Pick a convention, stick to it:

```c
// Convention: context first, then inputs, then outputs
int wordlib_complete(WordLib *wl, const char *prefix, 
                     const char **results, size_t max);

int wordlib_suggest(WordLib *wl, const char *word,
                    Suggestion *results, size_t max);

// Not: randomly ordered parameters
int bad_api(const char *word, size_t max, WordLib *wl, char **out);
```

### Return Values

Be consistent about what return values mean:

```c
// Option 1: Return counts/success
size_t wordlib_complete(...);  // Returns number of results
int wordlib_add_word(...);     // Returns 1 if added, 0 if existed

// Option 2: Output parameters + error codes
int wordlib_complete(..., size_t *count);  // Returns error code

// Pick one style and use it throughout!
```

### Const Correctness

Mark read-only parameters and methods:

```c
// This function doesn't modify the WordLib
bool wordlib_contains(const WordLib *wl, const char *word);

// This function doesn't modify the word
int wordlib_add_word(WordLib *wl, const char *word);

// This function modifies nothing (pure query)
size_t wordlib_complete(const WordLib *wl, const char *prefix,
                        const char **results, size_t max);
```

### Ownership Rules

Document who owns what:

```c
/**
 * Add a word to the library.
 * 
 * @param wl The word library (must not be NULL)
 * @param word The word to add (copied internally, caller retains ownership)
 * @return 1 if word was added, 0 if already existed, -1 on error
 */
int wordlib_add_word(WordLib *wl, const char *word);

/**
 * Get completion suggestions.
 *
 * @param results Array filled with pointers to internal strings.
 *                Valid only until next modification to wl.
 *                Caller must NOT free these pointers.
 */
size_t wordlib_complete(const WordLib *wl, const char *prefix,
                        const char **results, size_t max);
```

### Error Handling Philosophy

**Option 1: Return codes (C style)**
```c
typedef enum {
    WL_OK = 0,
    WL_ERR_NULL = -1,
    WL_ERR_MEMORY = -2,
    WL_ERR_IO = -3,
} WLError;

WLError wordlib_load(WordLib *wl, const char *path);

// Usage
WLError err = wordlib_load(wl, path);
if (err != WL_OK) {
    fprintf(stderr, "Load failed: %d\n", err);
}
```

**Option 2: Boolean for simple operations**
```c
bool wordlib_add_word(WordLib *wl, const char *word);
bool wordlib_remove_word(WordLib *wl, const char *word);

// Usage
if (!wordlib_add_word(wl, word)) {
    // Already existed or error
}
```

**Hybrid (recommended):**
- Simple operations: return bool or count
- Complex operations: return error code
- Queries: return result directly (NULL on error)

### Handle NULL Gracefully

```c
int wordlib_add_word(WordLib *wl, const char *word) {
    if (!wl) return -1;
    if (!word) return -1;
    if (!*word) return 0;  // Empty word, ignore
    
    // ... actual implementation
}

// This prevents:
wordlib_add_word(NULL, "crash");  // No crash!
wordlib_add_word(wl, NULL);       // No crash!
```

### Symmetric APIs

If you have create, have destroy:
```c
WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *wl);
```

If you have add, have remove:
```c
int wordlib_add_word(WordLib *wl, const char *word);
int wordlib_remove_word(WordLib *wl, const char *word);
```

If you have load, have save:
```c
int wordlib_load(WordLib *wl, const char *path);
int wordlib_save(const WordLib *wl, const char *path);
```

### Information Hiding

Expose only what adapters need:

```c
// Good: Opaque type, only necessary operations
typedef struct WordLib WordLib;
WordLib *wordlib_create(void);
bool wordlib_contains(const WordLib *wl, const char *word);

// Bad: Exposing internals
typedef struct WordLib {
    HashSet *words;     // Don't expose this!
    Trie *trie;         // Or this!
    BKTree *bktree;     // Or this!
} WordLib;
```

## Where to Learn

1. **"C Interfaces and Implementations"** by David Hanson
2. **"The Art of Unix Programming"** - API design chapter
3. **Study well-designed C libraries:** SQLite, zlib, curl
4. **"API Design for C++"** by Martin Reddy (concepts apply to C)

## Practice Exercise

Design the complete WordLib API on paper first:

1. List all operations an adapter might need
2. Group them logically (lifecycle, words, queries, persistence)
3. Define consistent naming convention
4. Specify parameter order convention
5. Document ownership for each function

Review against these questions:
- Can I guess what each function does from its name?
- Is parameter order consistent?
- Are ownership rules clear?
- Can any function crash on NULL input?

## Connection to Project

Your API is the contract between engine and adapters. Get it right and:
- CLI adapter is easy to write
- LSP adapter is easy to write  
- Future adapters are easy to write
- Tests are easy to write

Get it wrong and you'll be fighting your own code.
