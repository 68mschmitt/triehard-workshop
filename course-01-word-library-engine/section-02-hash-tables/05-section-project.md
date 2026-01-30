# Section 2 Project: Hash Set Implementation

## Objective

Build a production-quality hash set that will serve as the foundation for your word dictionary. This replaces the simple array from Section 1.

## Requirements

1. **O(1) average case** for add, contains, remove
2. **Dynamic resizing** at 70% load factor
3. **String interning** - stores copies, returns canonical pointers
4. **Memory safe** - Valgrind clean
5. **Well-tested** - Unit tests for all operations

## Specification

### Public API (`include/hashset.h`)

```c
typedef struct HashSet HashSet;

// Lifecycle
HashSet *hashset_create(void);
HashSet *hashset_create_with_capacity(size_t initial_capacity);
void hashset_destroy(HashSet *hs);

// Core operations
bool hashset_add(HashSet *hs, const char *key);
bool hashset_contains(const HashSet *hs, const char *key);
bool hashset_remove(HashSet *hs, const char *key);

// Interning (returns canonical pointer)
const char *hashset_intern(HashSet *hs, const char *key);

// Info
size_t hashset_count(const HashSet *hs);
size_t hashset_capacity(const HashSet *hs);
bool hashset_is_empty(const HashSet *hs);

// Iteration
typedef void (*HashSetIterator)(const char *key, void *userdata);
void hashset_foreach(const HashSet *hs, HashSetIterator fn, void *userdata);
```

### Internal Structure (suggested)

```c
// src/hashset.c
typedef struct Entry {
    char *key;
    struct Entry *next;
} Entry;

struct HashSet {
    Entry **buckets;
    size_t bucket_count;
    size_t entry_count;
};
```

### Behavior Specifications

**hashset_add:**
- Returns `true` if word was newly added
- Returns `false` if word already existed
- Stores a copy of the string (owns the memory)
- Triggers resize if load factor exceeds 0.7

**hashset_contains:**
- Returns `true` if word is in set
- Case-sensitive comparison
- Does not modify the set

**hashset_remove:**
- Returns `true` if word was found and removed
- Returns `false` if word wasn't present
- Frees the stored string copy

**hashset_intern:**
- If word exists, returns pointer to stored copy
- If word doesn't exist, adds it and returns pointer to new copy
- Returned pointer valid until word is removed or set is destroyed

## Test Cases

Create `test/test_hashset.c`:

```c
void test_basic_operations(void) {
    HashSet *hs = hashset_create();
    
    assert(hashset_is_empty(hs));
    assert(hashset_add(hs, "hello"));
    assert(!hashset_is_empty(hs));
    assert(hashset_count(hs) == 1);
    assert(hashset_contains(hs, "hello"));
    assert(!hashset_contains(hs, "world"));
    
    // Duplicate add returns false
    assert(!hashset_add(hs, "hello"));
    assert(hashset_count(hs) == 1);
    
    hashset_destroy(hs);
}

void test_interning(void) {
    HashSet *hs = hashset_create();
    
    const char *a = hashset_intern(hs, "test");
    const char *b = hashset_intern(hs, "test");
    assert(a == b);  // Same pointer!
    
    hashset_destroy(hs);
}

void test_resize(void) {
    HashSet *hs = hashset_create_with_capacity(4);
    
    // Add enough to trigger multiple resizes
    for (int i = 0; i < 1000; i++) {
        char buf[32];
        snprintf(buf, sizeof(buf), "word%d", i);
        hashset_add(hs, buf);
    }
    
    assert(hashset_count(hs) == 1000);
    
    // Verify all words still findable
    for (int i = 0; i < 1000; i++) {
        char buf[32];
        snprintf(buf, sizeof(buf), "word%d", i);
        assert(hashset_contains(hs, buf));
    }
    
    hashset_destroy(hs);
}

void test_remove(void) {
    HashSet *hs = hashset_create();
    
    hashset_add(hs, "hello");
    hashset_add(hs, "world");
    assert(hashset_count(hs) == 2);
    
    assert(hashset_remove(hs, "hello"));
    assert(hashset_count(hs) == 1);
    assert(!hashset_contains(hs, "hello"));
    assert(hashset_contains(hs, "world"));
    
    // Remove non-existent
    assert(!hashset_remove(hs, "hello"));
    
    hashset_destroy(hs);
}

void test_large_dictionary(void) {
    HashSet *hs = hashset_create();
    
    // Load real dictionary
    FILE *f = fopen("/usr/share/dict/words", "r");
    if (!f) {
        printf("Skipping dictionary test - file not found\n");
        hashset_destroy(hs);
        return;
    }
    
    char line[256];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = 0;  // Remove newline
        hashset_add(hs, line);
    }
    fclose(f);
    
    printf("Loaded %zu words\n", hashset_count(hs));
    
    // Spot check
    assert(hashset_contains(hs, "hello"));
    assert(hashset_contains(hs, "world"));
    assert(!hashset_contains(hs, "xyzzy123"));
    
    hashset_destroy(hs);
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test
Running hash set tests...
test_basic_operations: PASS
test_interning: PASS
test_resize: PASS
test_remove: PASS
test_large_dictionary: Loaded 235886 words - PASS
All tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_hashset
...
All heap blocks were freed -- no leaks are possible
```

### Performance
```bash
$ time ./test_hashset
# Loading 200k+ words should take < 1 second
# Individual lookups should be sub-microsecond
```

## Stretch Goals

1. **Case-insensitive mode** - Optional flag at creation
2. **Statistics** - Track collision count, max chain length, resize count
3. **Serialization** - `hashset_save(hs, file)` and `hashset_load(file)`
4. **Robin Hood variant** - Reimplement with open addressing for comparison

## Implementation Hints

### FNV-1a Hash (Use This)
```c
static uint32_t fnv1a(const char *str) {
    uint32_t hash = 2166136261u;
    while (*str) {
        hash ^= (uint8_t)*str++;
        hash *= 16777619u;
    }
    return hash;
}
```

### Safe String Duplication
```c
static char *safe_strdup(const char *s) {
    size_t len = strlen(s) + 1;
    char *copy = malloc(len);
    if (copy) memcpy(copy, s, len);
    return copy;
}
```

### Resize Check
```c
static bool should_resize(HashSet *hs) {
    return (float)hs->entry_count / hs->bucket_count > 0.7f;
}
```

## What You'll Learn

By completing this project, you'll have:
- Implemented a real data structure from scratch
- Handled dynamic memory management with multiple ownership patterns
- Written comprehensive tests
- Benchmarked performance on real data

## Next Section Preview

Section 3 builds a trie on top of this hash set. Your trie nodes will store interned strings from the hash set, giving you O(k) prefix completion where k is prefix length. The foundation you built here makes that clean and efficient.
