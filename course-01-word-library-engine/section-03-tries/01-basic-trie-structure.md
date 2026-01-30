# Basic Trie Structure

## Why You Need This

Your requirements state: "prefix-based word completion" that is "optimized for interactive usage." Tries are THE data structure for this. Typing "hel" and getting ["hello", "help", "helicopter"] in microseconds? That's tries.

## What to Learn

### The Core Idea

A trie (prefix tree) stores strings by their common prefixes:

```
           (root)
          /   |   \
         h    w    c
         |    |    |
         e    o    a
        /|    |    |
       l a    r    t
      /| |    |
     l p d    l
     |        |
     o        d
```

Words: hello, help, head, world, cat

Each path from root to a marked node spells a word.

### Node Structure

```c
typedef struct TrieNode {
    struct TrieNode *children[26];  // a-z (simplified)
    bool is_end_of_word;
    const char *word;  // Store interned string at word endings
} TrieNode;

typedef struct Trie {
    TrieNode *root;
    size_t word_count;
} Trie;
```

### Basic Operations

**Insertion:**
```c
void trie_insert(Trie *t, const char *word) {
    TrieNode *node = t->root;
    
    for (const char *c = word; *c; c++) {
        int idx = *c - 'a';  // Simplified: lowercase only
        if (!node->children[idx]) {
            node->children[idx] = calloc(1, sizeof(TrieNode));
        }
        node = node->children[idx];
    }
    
    if (!node->is_end_of_word) {
        node->is_end_of_word = true;
        node->word = word;  // Store interned pointer
        t->word_count++;
    }
}
```

**Lookup:**
```c
bool trie_contains(Trie *t, const char *word) {
    TrieNode *node = t->root;
    
    for (const char *c = word; *c; c++) {
        int idx = *c - 'a';
        if (!node->children[idx]) return false;
        node = node->children[idx];
    }
    
    return node->is_end_of_word;
}
```

**Prefix Search (the magic):**
```c
TrieNode *trie_find_prefix(Trie *t, const char *prefix) {
    TrieNode *node = t->root;
    
    for (const char *c = prefix; *c; c++) {
        int idx = *c - 'a';
        if (!node->children[idx]) return NULL;
        node = node->children[idx];
    }
    
    return node;  // All words under this node share the prefix
}
```

### Why Tries Excel at Completion

Hash table: "Find all words starting with 'hel'"
- Scan ALL words, check each prefix: O(n)

Trie: "Find all words starting with 'hel'"
- Walk h→e→l, then enumerate subtree: O(k + m)
  - k = prefix length (3)
  - m = number of matching words

For a dictionary of 100,000 words with 50 matches for "hel", trie wins massively.

### Time Complexity

| Operation | Hash Set | Trie |
|-----------|----------|------|
| Insert | O(k) avg | O(k) |
| Contains | O(k) avg | O(k) |
| Prefix search | O(n) | O(k + m) |

Where k = word length, n = total words, m = matching words.

### Visualization Exercise

Draw the trie for: ["cat", "car", "card", "care", "careful", "bat"]

```
       (root)
       /    \
      c      b
      |      |
      a      a
     / \     |
    t   r    t
        |
        (d, e)
         |   |
         .  (f)
             |
             u
             |
             l
```

Being able to trace through tries mentally is crucial for debugging.

## Where to Learn

1. **"Algorithms" by Sedgewick** - Chapter 5.2 on Tries
2. **Visualgo.net** - Interactive trie visualization
3. **LeetCode trie problems** - 208, 211, 212 for practice
4. **"The Art of Computer Programming" Vol 3** - Knuth's analysis

## Practice Exercise

Implement a basic trie (lowercase letters only):

```c
Trie *trie_create(void);
void trie_destroy(Trie *t);
void trie_insert(Trie *t, const char *word);
bool trie_contains(Trie *t, const char *word);
bool trie_has_prefix(Trie *t, const char *prefix);
```

Test with:
1. Insert 100 words
2. Verify contains works
3. Verify has_prefix works for partial matches
4. Check memory usage vs hash set (tries use more memory!)

## Connection to Project

This is the heart of your completion feature. When a user types "compl", your trie instantly narrows to the subtree containing "complete", "completion", "complex", etc. The next topic covers how to enumerate that subtree efficiently.
