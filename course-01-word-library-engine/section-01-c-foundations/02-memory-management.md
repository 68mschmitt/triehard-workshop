# Memory Management Patterns

## Why You Need This

Your requirements state: "explicit memory ownership rules." In C, you ARE the garbage collector. Get this wrong and you'll chase memory leaks and use-after-free bugs for days.

## What to Learn

### Stack vs Heap - The Mental Model

**Stack:** Automatic, fast, limited size, dies with scope
```c
void func() {
    char buffer[256];  // Stack - gone when func returns
}
```

**Heap:** Manual, slower, unlimited*, lives until you free it
```c
char *buffer = malloc(256);  // Heap - lives until free(buffer)
```

**Rule of thumb:** Small, fixed-size, short-lived → stack. Everything else → heap.

### Ownership Semantics

The million-dollar question: **who frees this memory?**

Three patterns:

1. **Caller allocates, caller frees:**
```c
char *buffer = malloc(1024);
read_into(buffer, 1024);  // Function just uses the buffer
free(buffer);
```

2. **Callee allocates, caller frees:**
```c
char *result = get_copy();  // Function allocates
// ... use result ...
free(result);  // Caller responsible for cleanup
```

3. **Context owns everything:**
```c
WordLib *ctx = wordlib_create();
const char *word = wordlib_get(ctx, 0);  // Don't free this!
wordlib_destroy(ctx);  // This frees everything
```

Your project will primarily use pattern #3 - the context owns all memory.

### The Create/Destroy Pattern

```c
// wordlib.h
typedef struct WordLib WordLib;  // Opaque
WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *ctx);

// wordlib.c
struct WordLib {
    HashSet *words;
    Trie *trie;
    BKTree *bktree;
};

WordLib *wordlib_create(void) {
    WordLib *ctx = malloc(sizeof(*ctx));
    if (!ctx) return NULL;
    ctx->words = hashset_create();
    ctx->trie = trie_create();
    ctx->bktree = bktree_create();
    return ctx;
}

void wordlib_destroy(WordLib *ctx) {
    if (!ctx) return;
    hashset_destroy(ctx->words);
    trie_destroy(ctx->trie);
    bktree_destroy(ctx->bktree);
    free(ctx);
}
```

### Common Mistakes to Avoid

```c
// WRONG: Returning stack memory
char *get_name() {
    char name[64];
    strcpy(name, "hello");
    return name;  // BOOM - dangling pointer
}

// WRONG: Double free
free(ptr);
do_something();
free(ptr);  // BOOM - undefined behavior

// WRONG: Use after free
free(ptr);
printf("%s\n", ptr);  // BOOM - garbage or crash

// WRONG: Forgetting to free
void leak() {
    char *data = malloc(1024);
    if (error) return;  // Leaked!
    free(data);
}
```

### Arena Allocators (Optional but Cool)

For batch operations, allocate a big chunk and hand out pieces:

```c
Arena *arena = arena_create(1024 * 1024);  // 1MB
char *word1 = arena_alloc(arena, 32);
char *word2 = arena_alloc(arena, 64);
// No individual frees needed
arena_destroy(arena);  // Frees everything at once
```

Great for tokenization results that all share a lifetime.

## Where to Learn

1. **"Modern C" Chapter 6** - Memory management deep dive
2. **Valgrind Tutorial** - Learn to detect your mistakes
   - `valgrind --leak-check=full ./your_program`
3. **"21st Century C" by Ben Klemens** - Chapter on memory

## Practice Exercise

Implement a simple "string pool":
```c
StringPool *pool_create(void);
const char *pool_intern(StringPool *pool, const char *str);
void pool_destroy(StringPool *pool);
```

The pool should:
- Store copies of strings (caller keeps ownership of originals)
- Return the same pointer for duplicate strings
- Free all memory on destroy

This is exactly what you'll need for your word dictionary!

## Connection to Project

Your `WordLib` context will own:
- The hash set of known words
- The trie for completions
- The BK-tree for suggestions
- All the word strings themselves

One `wordlib_destroy()` cleans up everything. Clean, predictable, testable.
