# Memory Layout

## Why You Need This

Your requirements say "optimized for interactive usage." That means latency matters. Modern CPUs are fast, but memory is slow. Cache-friendly data layouts can 10x your performance.

## What to Learn

### The Cache Hierarchy

```
Register:   < 1 ns    (handful of values)
L1 Cache:   ~ 1 ns    (32 KB)
L2 Cache:   ~ 4 ns    (256 KB)  
L3 Cache:   ~ 12 ns   (8 MB)
RAM:        ~ 100 ns  (gigabytes)
```

Cache fetches in 64-byte "lines." If you touch one byte, you load 64.

### Pointer-Heavy Structures Are Slow

Classic trie:
```c
struct Node {
    struct Node *children[26];  // 208 bytes, scattered in memory
    bool is_end;
};
```

Each child pointer → potential cache miss → 100ns penalty.

Traversing "hello" (5 characters) = up to 5 cache misses = 500ns just waiting for memory!

### Cache-Friendly Alternative: Arrays

```c
// All nodes in contiguous array
typedef struct {
    uint8_t children[26];  // Indices, not pointers (0 = no child)
    bool is_end;
} CompactNode;

typedef struct {
    CompactNode *nodes;    // Single allocation
    size_t count;
    size_t capacity;
} CompactTrie;
```

Traversing "hello" → sequential array access → likely in cache → fast!

**Limitation:** Only ~255 children per node with uint8_t indices. Use uint16_t or uint32_t for larger tries.

### Arena Allocation for Tries

Instead of malloc per node, allocate in chunks:

```c
typedef struct Arena {
    char *memory;
    size_t used;
    size_t capacity;
} Arena;

void *arena_alloc(Arena *a, size_t size) {
    if (a->used + size > a->capacity) {
        // Grow or fail
    }
    void *ptr = a->memory + a->used;
    a->used += size;
    return ptr;
}
```

All nodes in same region → better locality.

### Packed Node Structures

```c
// Minimize padding with careful ordering
typedef struct __attribute__((packed)) {
    uint32_t child_index;  // 4 bytes
    uint16_t sibling;      // 2 bytes
    uint8_t character;     // 1 byte
    uint8_t flags;         // 1 byte (is_end, etc.)
} PackedNode;  // Exactly 8 bytes
```

Smaller nodes = more fit in cache = faster.

### Left-Child Right-Sibling Representation

```
Standard:          LC-RS:
    A                A
   /|\              /
  B C D            B → C → D
 /|   |           /     |
E F   G          E → F  G
```

```c
typedef struct Node {
    struct Node *child;    // First child
    struct Node *sibling;  // Next sibling at same level
    char character;
    bool is_end;
} Node;
```

Fewer pointers per node, more sequential traversal.

### Measuring Cache Performance

```bash
# Linux perf tool
perf stat -e cache-misses,cache-references ./your_program

# Valgrind cachegrind
valgrind --tool=cachegrind ./your_program
```

Look for:
- Cache miss rate (lower is better)
- Cache references (shows memory access patterns)

### Practical Recommendations

**For your project, prioritize:**

1. **Use interned strings** - One allocation per unique word
2. **Sorted array children** - Better than hash map for small fanout
3. **Consider arena allocation** - All nodes in one region
4. **Benchmark with real data** - Theory doesn't always match practice

**Don't over-optimize early!** Get it working first, then profile.

## Where to Learn

1. **"What Every Programmer Should Know About Memory" by Ulrich Drepper** - THE deep dive
2. **"Computer Systems: A Programmer's Perspective"** - Chapter 6 on memory hierarchy
3. **Agner Fog's optimization manuals** - Hardcore CPU details
4. **Blog:** "Data-Oriented Design" articles (game dev community)

## Practice Exercise

Compare two implementations:

1. **Pointer-based:** Standard trie with malloc per node
2. **Array-based:** Nodes in contiguous array, indices instead of pointers

Load 200k words, complete "th" 10,000 times, measure time.

Expected result: Array version 2-5x faster.

## Connection to Project

For a personal dictionary (10k-50k words), a well-written pointer-based trie is fine. But if you want it to feel instant, these techniques help. More importantly, understanding WHY things are fast helps you make good design decisions throughout.
