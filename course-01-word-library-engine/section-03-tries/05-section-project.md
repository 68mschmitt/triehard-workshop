# Section 3 Project: Trie Implementation

## Objective

Build a trie that provides fast prefix-based completion. This will power your editor's autocomplete feature.

## Requirements

1. **O(k) insertion and lookup** where k = word length
2. **Efficient completion** - O(k + m) where m = number of matches
3. **Memory reasonable** - Not 50MB for 10k words
4. **Integrates with string interning** - Stores interned pointers
5. **Bounded results** - Completion limited by caller

## Specification

### Public API (`include/trie.h`)

```c
typedef struct Trie Trie;

// Lifecycle
Trie *trie_create(void);
void trie_destroy(Trie *t);

// Core operations
bool trie_insert(Trie *t, const char *word);
bool trie_contains(const Trie *t, const char *word);
bool trie_remove(Trie *t, const char *word);

// Completion
size_t trie_complete(const Trie *t, const char *prefix,
                     const char **results, size_t max_results);

// Info
size_t trie_count(const Trie *t);
bool trie_is_empty(const Trie *t);

// Iteration
typedef bool (*TrieIterator)(const char *word, void *userdata);
void trie_foreach(const Trie *t, TrieIterator fn, void *userdata);
```

### Internal Structure (recommended)

```c
// src/trie.c
typedef struct TrieNode {
    char *chars;              // Sorted array of child characters
    struct TrieNode **children;  // Corresponding child nodes
    uint8_t child_count;
    uint8_t child_capacity;
    bool is_end;
    const char *word;         // Interned string at word endings
} TrieNode;

struct Trie {
    TrieNode *root;
    size_t word_count;
};
```

### Behavior Specifications

**trie_insert:**
- Returns `true` if word was newly inserted
- Returns `false` if word already existed
- Stores the word pointer directly (caller owns the string - use interned strings!)

**trie_contains:**
- Returns `true` if exact word is in trie
- Prefix matches return `false` (use completion for that)

**trie_complete:**
- Finds all words starting with prefix
- Fills results array up to max_results
- Returns actual count (may be less than max)
- Returns 0 if no matches or empty prefix

**trie_remove:**
- Returns `true` if word was found and removed
- Cleans up empty nodes (optional but nice)

## Test Cases

Create `test/test_trie.c`:

```c
void test_basic_operations(void) {
    Trie *t = trie_create();
    
    assert(trie_is_empty(t));
    assert(trie_insert(t, "hello"));
    assert(!trie_is_empty(t));
    assert(trie_count(t) == 1);
    assert(trie_contains(t, "hello"));
    assert(!trie_contains(t, "hell"));  // Prefix, not word
    assert(!trie_contains(t, "hello!"));
    
    // Duplicate insert returns false
    assert(!trie_insert(t, "hello"));
    
    trie_destroy(t);
}

void test_completion(void) {
    Trie *t = trie_create();
    
    trie_insert(t, "hello");
    trie_insert(t, "help");
    trie_insert(t, "helicopter");
    trie_insert(t, "world");
    
    const char *results[10];
    size_t count;
    
    // Complete "hel"
    count = trie_complete(t, "hel", results, 10);
    assert(count == 3);
    // Results should be alphabetical: helicopter, hello, help
    
    // Complete "wor"
    count = trie_complete(t, "wor", results, 10);
    assert(count == 1);
    assert(strcmp(results[0], "world") == 0);
    
    // Complete "xyz" - no matches
    count = trie_complete(t, "xyz", results, 10);
    assert(count == 0);
    
    // Complete with limit
    count = trie_complete(t, "hel", results, 2);
    assert(count == 2);
    
    trie_destroy(t);
}

void test_prefix_vs_word(void) {
    Trie *t = trie_create();
    
    trie_insert(t, "help");
    trie_insert(t, "helper");
    
    assert(trie_contains(t, "help"));
    assert(trie_contains(t, "helper"));
    assert(!trie_contains(t, "hel"));  // Not a word
    assert(!trie_contains(t, "helpless"));  // Not inserted
    
    trie_destroy(t);
}

void test_large_completion(void) {
    Trie *t = trie_create();
    
    // Load dictionary
    FILE *f = fopen("/usr/share/dict/words", "r");
    if (!f) {
        printf("Skipping - dictionary not found\n");
        trie_destroy(t);
        return;
    }
    
    char line[256];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = 0;
        trie_insert(t, strdup(line));  // Leak for simplicity
    }
    fclose(f);
    
    printf("Loaded %zu words\n", trie_count(t));
    
    // Benchmark completion
    const char *results[100];
    clock_t start = clock();
    
    for (int i = 0; i < 10000; i++) {
        trie_complete(t, "th", results, 100);
    }
    
    clock_t end = clock();
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    printf("10k completions of 'th': %.2f ms (%.2f us each)\n", 
           ms, ms * 100);
    
    trie_destroy(t);
}

void test_remove(void) {
    Trie *t = trie_create();
    
    trie_insert(t, "hello");
    trie_insert(t, "help");
    assert(trie_count(t) == 2);
    
    assert(trie_remove(t, "hello"));
    assert(trie_count(t) == 1);
    assert(!trie_contains(t, "hello"));
    assert(trie_contains(t, "help"));
    
    // Remove non-existent
    assert(!trie_remove(t, "hello"));
    
    trie_destroy(t);
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_trie
Running trie tests...
test_basic_operations: PASS
test_completion: PASS
test_prefix_vs_word: PASS
test_large_completion: Loaded 235886 words
  10k completions of 'th': 45.23 ms (4.52 us each) - PASS
test_remove: PASS
All tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_trie
...
All heap blocks were freed -- no leaks are possible
```

### Performance Targets
- Single completion (100 results max): < 100 μs
- Memory for 50k words: < 10 MB

## Stretch Goals

1. **Case-insensitive completion** - "Hel" matches "hello"
2. **Fuzzy prefix** - "hllo" matches "hello" (edit distance 1 from prefix)
3. **Frequency-weighted results** - Return most common words first
4. **Memory stats** - Report node count, average fanout, memory used

## Implementation Hints

### Sorted Child Insertion
```c
static void add_child(TrieNode *node, char c, TrieNode *child) {
    // Find insertion point (binary search)
    int pos = 0;
    while (pos < node->child_count && node->chars[pos] < c) {
        pos++;
    }
    
    // Make room
    if (node->child_count >= node->child_capacity) {
        // Grow arrays
    }
    
    // Shift existing elements
    memmove(&node->chars[pos + 1], &node->chars[pos], 
            node->child_count - pos);
    memmove(&node->children[pos + 1], &node->children[pos],
            (node->child_count - pos) * sizeof(TrieNode *));
    
    // Insert new child
    node->chars[pos] = c;
    node->children[pos] = child;
    node->child_count++;
}
```

### Binary Search for Child
```c
static TrieNode *get_child(TrieNode *node, char c) {
    int lo = 0, hi = node->child_count - 1;
    while (lo <= hi) {
        int mid = (lo + hi) / 2;
        if (node->chars[mid] == c) return node->children[mid];
        if (node->chars[mid] < c) lo = mid + 1;
        else hi = mid - 1;
    }
    return NULL;
}
```

## What You'll Learn

By completing this project:
- Tree data structure implementation
- Recursive and iterative tree traversal
- Memory management for tree structures
- Performance measurement and optimization

## Next Section Preview

Section 4 adds fuzzy matching with BK-trees. While tries handle "find words starting with X", BK-trees handle "find words similar to X" - essential for spelling suggestions.
