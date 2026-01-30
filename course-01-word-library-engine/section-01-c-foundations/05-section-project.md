# Section 1 Project: Library Skeleton

## Objective

Build a complete project skeleton that demonstrates all Section 1 concepts. This becomes the foundation for everything you build afterward.

## Requirements

Create a minimal but complete C library structure with:

1. **Opaque type** with create/destroy lifecycle
2. **Proper header** with include guards and C++ compatibility
3. **Makefile** with debug/release targets
4. **Separate compilation** of at least 2 modules
5. **No memory leaks** (verifiable with Valgrind)

## Specification

### Public API (`include/wordlib.h`)

```c
typedef struct WordLib WordLib;

// Lifecycle
WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *wl);

// Basic operations (stubs for now)
int wordlib_add_word(WordLib *wl, const char *word);
size_t wordlib_count(const WordLib *wl);
```

### Internal Implementation

The `WordLib` struct should contain:
- A simple dynamic array of words (placeholder for hash set)
- A count field
- Any other fields you want to experiment with

### CLI (`cli/main.c`)

Simple test harness:
```c
int main(void) {
    WordLib *wl = wordlib_create();
    if (!wl) {
        fprintf(stderr, "Failed to create WordLib\n");
        return 1;
    }
    
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "world");
    
    printf("Word count: %zu\n", wordlib_count(wl));
    
    wordlib_destroy(wl);
    return 0;
}
```

### Directory Structure

```
triehard-workshop/
├── include/
│   └── wordlib.h
├── src/
│   ├── wordlib.c
│   └── util.c        # String utilities (strdup, etc.)
├── cli/
│   └── main.c
├── Makefile
└── README.md
```

## Acceptance Criteria

### Builds Cleanly
```bash
$ make clean && make
# No warnings with -Wall -Wextra -Wpedantic
```

### Runs Correctly
```bash
$ ./wordlib
Word count: 2
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./wordlib
# ...
# All heap blocks were freed -- no leaks are possible
```

### Debug Build Works
```bash
$ make clean && make debug
$ ./wordlib  # Should have debug symbols, no optimization
```

## Stretch Goals

1. **Add a `wordlib_has_word()` function** - Linear search is fine for now
2. **Handle NULL gracefully** - All functions should check and not crash
3. **Add a simple test target** - `make test` runs basic assertions
4. **Implement proper error return codes** - Define an enum for errors

## Hints

### String Duplication
C doesn't have `strdup` in C99 standard (it's POSIX). Either:
- Use it anyway (works on Linux/Mac)
- Write your own in `util.c`

```c
char *my_strdup(const char *s) {
    size_t len = strlen(s) + 1;
    char *copy = malloc(len);
    if (copy) memcpy(copy, s, len);
    return copy;
}
```

### Dynamic Array Growth
Start with capacity 16, double when full:

```c
if (wl->count >= wl->capacity) {
    size_t new_cap = wl->capacity * 2;
    char **new_words = realloc(wl->words, new_cap * sizeof(char *));
    if (!new_words) return -1;
    wl->words = new_words;
    wl->capacity = new_cap;
}
```

### Makefile Include Path
```makefile
CFLAGS += -Iinclude
```

## What You'll Learn

By completing this project, you'll have hands-on experience with:
- Opaque types in practice
- Memory ownership (who allocates, who frees)
- Separate compilation and linking
- Makefile basics
- Valgrind usage

## Next Section Preview

Section 2 replaces your simple dynamic array with a proper hash set - O(1) lookups instead of O(n). But the API stays the same! That's the power of opaque types.
