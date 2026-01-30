# Collision Resolution

## Why You Need This

Two different words can hash to the same index. That's a collision. How you handle collisions determines your hash table's performance under load.

## What to Learn

### The Collision Problem

```c
hash("hello") % 8 = 3
hash("world") % 8 = 3  // Uh oh, same bucket!
```

This WILL happen. A good hash reduces collisions but can't eliminate them.

### Strategy 1: Separate Chaining

Each bucket is a linked list of entries:

```
Bucket 0: → NULL
Bucket 1: → "apple" → "banana" → NULL  
Bucket 2: → NULL
Bucket 3: → "hello" → "world" → NULL
...
```

```c
typedef struct Entry {
    char *key;
    struct Entry *next;
} Entry;

typedef struct {
    Entry **buckets;
    size_t size;
    size_t count;
} HashSet;

bool hashset_contains(HashSet *hs, const char *key) {
    size_t idx = fnv1a(key) % hs->size;
    for (Entry *e = hs->buckets[idx]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) return true;
    }
    return false;
}
```

**Pros:** Simple, never "full", deletions are easy
**Cons:** Cache-unfriendly (pointer chasing), extra memory for nodes

### Strategy 2: Open Addressing (Linear Probing)

Collisions probe to the next slot:

```
Index 0: "grape"
Index 1: [empty]
Index 2: [empty]
Index 3: "hello"   ← hash("hello") = 3
Index 4: "world"   ← hash("world") = 3, but 3 taken, so 4
Index 5: [empty]
```

```c
typedef struct {
    char **keys;      // Array of string pointers
    size_t size;
    size_t count;
} HashSet;

bool hashset_contains(HashSet *hs, const char *key) {
    size_t idx = fnv1a(key) % hs->size;
    while (hs->keys[idx] != NULL) {
        if (strcmp(hs->keys[idx], key) == 0) return true;
        idx = (idx + 1) % hs->size;  // Wrap around
    }
    return false;
}
```

**Pros:** Cache-friendly (sequential memory), simple
**Cons:** Clustering problems, deletions are tricky (tombstones)

### Strategy 3: Robin Hood Hashing (Modern Favorite)

Like linear probing, but "steal from the rich":
- Track how far each entry is from its ideal position
- If a new entry has traveled farther, swap them
- Keeps probe distances more even

```c
// Simplified concept
if (new_entry.probe_distance > existing.probe_distance) {
    swap(new_entry, existing);
}
```

**Pros:** Best of open addressing, bounded worst case
**Cons:** Slightly more complex

### Tombstones for Deletion (Open Addressing)

Can't just set slot to NULL - breaks probe chains!

```c
#define TOMBSTONE ((char *)-1)

void hashset_remove(HashSet *hs, const char *key) {
    size_t idx = find_index(hs, key);
    if (idx != NOT_FOUND) {
        free(hs->keys[idx]);
        hs->keys[idx] = TOMBSTONE;  // Not NULL!
        hs->count--;
    }
}
```

During lookup, tombstones count as "occupied but not matching."

### Which to Choose?

| Use Case | Best Choice |
|----------|-------------|
| Simple, memory not tight | Separate chaining |
| Performance critical | Linear probing or Robin Hood |
| Many deletions | Separate chaining |
| Mostly lookups | Open addressing |

**Recommendation for your project:** Start with separate chaining (easier), optimize to Robin Hood later if needed.

## Where to Learn

1. **"Introduction to Algorithms" (CLRS)** - Chapter 11
2. **"Robin Hood Hashing" blog posts** - Sebastian Sylvan's original, many follow-ups
3. **Google's Swiss Tables** - Modern state-of-the-art (complex but fascinating)
4. **Visualizations** - Many online hash table visualizers show collision handling

## Practice Exercise

Implement a hash set with separate chaining:

```c
HashSet *hashset_create(size_t initial_size);
void hashset_destroy(HashSet *hs);
bool hashset_add(HashSet *hs, const char *key);
bool hashset_contains(HashSet *hs, const char *key);
bool hashset_remove(HashSet *hs, const char *key);
size_t hashset_count(HashSet *hs);
```

Test with:
1. Adding 10,000 words
2. Checking membership for words that exist and don't exist
3. Removing words and verifying they're gone

## Connection to Project

Your "known words" dictionary is fundamentally a hash set. The choice of collision resolution affects:
- Memory usage
- Lookup speed
- Code complexity

Get this right and your membership checks will be genuinely O(1).
