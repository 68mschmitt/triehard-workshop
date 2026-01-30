# String Interning

## Why You Need This

Your requirements state: "internal data structures must derive from a single canonical word store." This means one copy of each word string, shared by hash set, trie, and BK-tree. String interning is how.

## What to Learn

### The Problem

Without interning:
```c
hashset_add(hs, "hello");   // Makes copy #1
trie_insert(trie, "hello"); // Makes copy #2
bktree_add(bk, "hello");    // Makes copy #3
// 3x memory for the same word!
```

With interning:
```c
const char *word = intern("hello");  // One canonical copy
hashset_add(hs, word);   // Stores pointer
trie_insert(trie, word); // Same pointer
bktree_add(bk, word);    // Same pointer
// 1x memory, plus pointer comparison works!
```

### The String Pool

```c
typedef struct StringPool {
    HashSet *strings;  // Uses your hash set!
} StringPool;

StringPool *stringpool_create(void) {
    StringPool *pool = malloc(sizeof(*pool));
    pool->strings = hashset_create(1024);
    return pool;
}

const char *stringpool_intern(StringPool *pool, const char *str) {
    // Check if already interned
    const char *existing = hashset_get(pool->strings, str);
    if (existing) return existing;
    
    // Add new string
    char *copy = strdup(str);
    hashset_add(pool->strings, copy);
    return copy;
}

void stringpool_destroy(StringPool *pool) {
    // Free all interned strings
    hashset_destroy_with_values(pool->strings);
    free(pool);
}
```

### Pointer Comparison Bonus

With interning, string comparison becomes pointer comparison:

```c
// Without interning: O(n) string compare
if (strcmp(word1, word2) == 0) { ... }

// With interning: O(1) pointer compare
if (word1 == word2) { ... }  // Same pointer = same string!
```

This is a massive speedup for trie and BK-tree operations.

### Memory Ownership

Clear rule: **the pool owns all strings.**

```c
const char *word = stringpool_intern(pool, user_input);
// word is valid until pool is destroyed
// DO NOT free(word)!

stringpool_destroy(pool);
// word is now invalid - dangling pointer
```

Your `WordLib` context owns the pool. Everything else borrows.

### Integration Pattern

```c
struct WordLib {
    StringPool *pool;     // Owns all word strings
    HashSet *words;       // Contains interned pointers
    Trie *completions;    // Contains interned pointers  
    BKTree *suggestions;  // Contains interned pointers
};

int wordlib_add_word(WordLib *wl, const char *word) {
    // Intern first - now we have canonical pointer
    const char *interned = stringpool_intern(wl->pool, word);
    
    // All structures get same pointer
    hashset_add(wl->words, interned);
    trie_insert(wl->completions, interned);
    bktree_add(wl->suggestions, interned);
    
    return 0;
}
```

### Deduplication

Interning automatically deduplicates:

```c
const char *a = stringpool_intern(pool, "hello");
const char *b = stringpool_intern(pool, "hello");
assert(a == b);  // Same pointer!
```

Adding a word twice? No extra memory used.

### Performance Considerations

**Interning cost:** One hash lookup + possible allocation
**Subsequent uses:** Just pointer operations

For your use case (words added once, used many times), this is a huge win.

## Where to Learn

1. **Lua source code** - Beautiful string interning implementation
2. **CPython string interning** - How Python handles it
3. **"Game Programming Patterns" by Nystrom** - "Flyweight" pattern chapter
4. **Java String.intern()** - Conceptually similar

## Practice Exercise

Build a string pool:

```c
StringPool *stringpool_create(void);
void stringpool_destroy(StringPool *pool);
const char *stringpool_intern(StringPool *pool, const char *str);
size_t stringpool_count(StringPool *pool);
```

Test:
1. Intern 10,000 words from a dictionary
2. Intern same words again - count shouldn't change
3. Verify all pointers for same word are identical
4. Measure memory: should be ~1x word data, not 3x

## Connection to Project

This is the glue that makes your architecture work:
- One canonical store (requirement satisfied)
- Memory efficient (only one copy of each word)
- Fast comparisons (pointer equality)
- Clean ownership (pool owns strings, destroys them all)

Your hash set, trie, and BK-tree become lightweight - they just store pointers, not strings.
