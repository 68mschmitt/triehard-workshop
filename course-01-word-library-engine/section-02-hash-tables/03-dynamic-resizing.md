# Dynamic Resizing

## Why You Need This

Your requirements say "support large personal dictionaries (thousands to tens of thousands of words)." You can't pre-allocate for 50,000 words when users might only have 100. And you can't cap at 1,000 when power users want 50,000.

Dynamic resizing solves both problems.

## What to Learn

### Load Factor

```c
float load_factor = (float)count / (float)capacity;
```

- **Low load factor (< 0.5):** Wasting memory, but fast lookups
- **High load factor (> 0.9):** Memory efficient, but collisions kill performance
- **Sweet spot:** 0.6 - 0.75 for most applications

When load factor exceeds threshold → time to resize.

### The Resizing Process

1. Allocate new, larger table
2. Rehash ALL existing entries (hash % new_size is different!)
3. Free old table

```c
bool hashset_resize(HashSet *hs, size_t new_size) {
    Entry **new_buckets = calloc(new_size, sizeof(Entry *));
    if (!new_buckets) return false;
    
    // Rehash all existing entries
    for (size_t i = 0; i < hs->size; i++) {
        Entry *e = hs->buckets[i];
        while (e) {
            Entry *next = e->next;  // Save before we modify
            
            // Insert into new table
            size_t new_idx = fnv1a(e->key) % new_size;
            e->next = new_buckets[new_idx];
            new_buckets[new_idx] = e;
            
            e = next;
        }
    }
    
    free(hs->buckets);
    hs->buckets = new_buckets;
    hs->size = new_size;
    return true;
}
```

### Growth Strategies

**Double the size:**
```c
new_size = old_size * 2;
```
Pros: Simple, amortized O(1) inserts. Cons: Sudden memory jumps.

**Prime number growth:**
```c
// Array of primes: 17, 37, 79, 163, 331, 673, ...
new_size = next_prime_in_table(old_size);
```
Pros: Better distribution with modulo. Cons: More complex.

**Power of two (with good hash):**
```c
new_size = old_size * 2;  // Always power of 2
index = hash & (size - 1);  // Fast bitwise AND
```
Pros: Fastest indexing. Cons: Needs excellent hash function.

### When to Resize

**Grow when:**
```c
if ((float)hs->count / hs->size > 0.7) {
    hashset_resize(hs, hs->size * 2);
}
```

**Shrink when (optional):**
```c
if (hs->size > INITIAL_SIZE && 
    (float)hs->count / hs->size < 0.2) {
    hashset_resize(hs, hs->size / 2);
}
```

Most implementations don't shrink - memory is cheap, shrinking is rare.

### Amortized Analysis (Why This Is O(1))

Individual resize is O(n) - you touch every element. Scary!

But think about it:
- Size 16 → 32: Rehash 11 items (at 70% load)
- Size 32 → 64: Rehash 22 items
- Size 64 → 128: Rehash 44 items

To trigger resize at 128, you inserted 44 items since last resize.
Cost of resize: 44 operations.
Cost per insert: 44/44 = 1 extra operation on average.

That's **amortized O(1)** - occasional expensive operations, but cheap on average.

### Thread Safety Note

Resizing is NOT thread-safe:
```c
// Thread 1: Looking up in old table
// Thread 2: Resizing, frees old table
// Thread 1: 💥
```

For your single-threaded CLI, this is fine. LSP adapter may need locking later.

## Where to Learn

1. **"Introduction to Algorithms" (CLRS)** - Amortized analysis chapter
2. **"The Art of Computer Programming" Vol 3** - Detailed analysis
3. **CPython dict implementation** - Well-commented, handles growth elegantly
4. **Java HashMap source** - Clear resize logic

## Practice Exercise

Extend your hash set with dynamic resizing:

1. Start with size 16
2. Resize at 70% load factor
3. Test by inserting 100,000 words
4. Track number of resizes (should be ~13 for 100k items starting at 16)
5. Verify all words still findable after resizes

**Bonus:** Add timing - measure insert speed before and after your resize implementation. Should stay roughly constant even as size grows.

## Connection to Project

Your word library needs to:
- Start small for users with few words
- Scale to tens of thousands without performance degradation
- Not waste memory on small dictionaries

Dynamic resizing is how you achieve all three. It's the difference between a toy and a tool.
