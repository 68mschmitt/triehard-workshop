# Trie Variants

## Why You Need This

The basic 26-pointer-per-node trie wastes massive memory. For ASCII, that's 26 × 8 = 208 bytes per node, even if most children are NULL. Real tries use smarter representations.

## What to Learn

### The Memory Problem

Basic trie node for "hello":
```c
struct TrieNode {
    struct TrieNode *children[26];  // 208 bytes!
    bool is_end;
    const char *word;
};
```

Most nodes have 1-3 children. You're paying for 26 slots every time.

### Variant 1: Hash-Based Children

Replace fixed array with hash map:

```c
typedef struct TrieNode {
    HashMap *children;  // Only allocates for actual children
    bool is_end;
    const char *word;
} TrieNode;

TrieNode *get_child(TrieNode *node, char c) {
    return hashmap_get(node->children, c);
}

void set_child(TrieNode *node, char c, TrieNode *child) {
    hashmap_set(node->children, c, child);
}
```

**Pros:** Great for sparse nodes, supports full Unicode
**Cons:** Hash overhead for dense nodes, more allocations

### Variant 2: Sorted Array Children

Store children in a sorted array:

```c
typedef struct TrieNode {
    char *chars;           // "acr" - sorted
    struct TrieNode **children;  // Corresponding child pointers
    uint8_t child_count;
    bool is_end;
    const char *word;
} TrieNode;

TrieNode *get_child(TrieNode *node, char c) {
    // Binary search in chars array
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

**Pros:** Memory efficient, cache-friendly
**Cons:** O(log k) child lookup instead of O(1)

### Variant 3: Patricia Tries (Radix Tries)

Compress chains of single-child nodes:

```
Standard trie:      Patricia trie:
     c                   c
     |                   |
     a                  "ar"
     |                 /    \
     r               "d"   "eful"
    / \
   d   e
       |
       f
       |
       u
       |
       l
```

```c
typedef struct PatriciaNode {
    char *edge_label;  // Stores multiple characters
    HashMap *children;
    bool is_end;
} PatriciaNode;
```

**Pros:** Dramatically fewer nodes, faster traversal
**Cons:** More complex insertion/deletion

### Variant 4: Array-Mapped Trie (for Performance)

Use bitmap to indicate which children exist:

```c
typedef struct TrieNode {
    uint32_t bitmap;  // Bit i set = child for char i exists
    struct TrieNode **children;  // Compact array, only existing children
    bool is_end;
} TrieNode;

TrieNode *get_child(TrieNode *node, char c) {
    int idx = c - 'a';
    if (!(node->bitmap & (1u << idx))) return NULL;
    
    // Count bits below idx to find array position
    int pos = __builtin_popcount(node->bitmap & ((1u << idx) - 1));
    return node->children[pos];
}
```

**Pros:** Compact AND fast
**Cons:** Complex insertion (need to resize and shift)

### Which to Choose?

| Use Case | Best Variant |
|----------|--------------|
| Simple, lowercase only | Fixed array [26] or sorted array |
| Full Unicode support | Hash-based children |
| Memory critical | Patricia trie |
| Read-heavy, write-rare | Array-mapped |

**For your project:** Start with sorted array children. Good balance of memory and speed. Patricia trie is a nice optimization later.

### Memory Comparison (10,000 words)

| Variant | Approximate Memory |
|---------|-------------------|
| Fixed array [26] | ~50 MB |
| Hash children | ~5 MB |
| Sorted array | ~2 MB |
| Patricia trie | ~1 MB |

These are rough estimates - actual depends on word patterns.

## Where to Learn

1. **"Algorithms" by Sedgewick** - Ternary Search Tries
2. **Original Patricia paper** by Morrison (1968)
3. **Blog:** "Radix Trees" on algorithms-and-data-structures sites
4. **Code:** Redis uses radix tree (rax.c) - excellent implementation

## Practice Exercise

Implement sorted array children:

```c
typedef struct TrieNode {
    char *chars;
    struct TrieNode **children;
    uint8_t count;
    bool is_end;
    const char *word;
} TrieNode;

TrieNode *trienode_get_child(TrieNode *node, char c);
void trienode_set_child(TrieNode *node, char c, TrieNode *child);
```

Compare memory usage against fixed-array version with 10,000 words.

## Connection to Project

Your trie needs to handle UTF-8 words and potentially tens of thousands of entries. Fixed [26] arrays won't cut it. Hash-based or sorted array children give you flexibility for Unicode and reasonable memory usage.
