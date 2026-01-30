# Hash Function Design

## Why You Need This

Your requirements state "membership checks shall be O(1) average case." Hash tables deliver this, but ONLY if you have a good hash function. Bad hash = collisions = O(n) performance = sad.

## What to Learn

### What Makes a Good Hash Function?

1. **Deterministic:** Same input → same output, always
2. **Uniform distribution:** Outputs spread evenly across range
3. **Avalanche effect:** Small input changes → big output changes
4. **Fast:** Constant time per byte of input

### Hash Functions for Strings

**djb2 (Simple, decent):**
```c
uint32_t djb2(const char *str) {
    uint32_t hash = 5381;
    int c;
    while ((c = *str++)) {
        hash = ((hash << 5) + hash) + c;  // hash * 33 + c
    }
    return hash;
}
```
Pros: Dead simple, fast. Cons: Not the best distribution.

**FNV-1a (Better distribution):**
```c
uint32_t fnv1a(const char *str) {
    uint32_t hash = 2166136261u;  // FNV offset basis
    while (*str) {
        hash ^= (uint8_t)*str++;
        hash *= 16777619u;  // FNV prime
    }
    return hash;
}
```
Pros: Good distribution, still simple. This is a solid default choice.

**xxHash (Performance beast):**
More complex, but blazing fast for longer strings. Overkill for typical word lengths, but worth knowing exists.

### Mapping Hash to Index

```c
size_t index = hash % table_size;
```

**Critical:** Use prime table sizes OR power-of-two with good hash.

Power-of-two trick (faster, but needs good hash):
```c
size_t index = hash & (table_size - 1);  // Only if table_size is 2^n
```

### Testing Your Hash Function

```c
// Distribution test: hash 100k words, check bucket distribution
int buckets[1024] = {0};
for (int i = 0; i < word_count; i++) {
    uint32_t h = fnv1a(words[i]);
    buckets[h % 1024]++;
}
// Ideal: each bucket has ~word_count/1024 entries
// Bad: some buckets have 1000x more than others
```

### Common Pitfalls

**Ignoring case:**
```c
// "Hello" and "hello" should probably hash the same
uint32_t hash_lower(const char *str) {
    uint32_t hash = 2166136261u;
    while (*str) {
        hash ^= (uint8_t)tolower(*str++);
        hash *= 16777619u;
    }
    return hash;
}
```

**Integer overflow:** Not a bug in C for unsigned types! Wrapping is defined behavior and expected.

**Using `strlen` in the hash:** Just iterate until null terminator - no need to know length first.

## Where to Learn

1. **"Algorithms" by Sedgewick** - Chapter 3.4 on hashing
2. **FNV Hash Page:** http://www.isthe.com/chongo/tech/comp/fnv/
3. **SMHasher:** Test suite for hash functions (shows quality metrics)
4. **"The Art of Computer Programming" Vol 3** - Knuth's deep dive (optional, dense)

## Practice Exercise

1. Implement both `djb2` and `fnv1a`
2. Load `/usr/share/dict/words` (or similar word list)
3. Hash all words into 1024 buckets
4. Count how many words land in each bucket
5. Calculate standard deviation - lower is better

**Expected result:** FNV-1a should have slightly better distribution than djb2.

## Connection to Project

Your hash set will use one of these functions to:
- Check if a word exists (O(1) average)
- Find where to insert a new word
- Support your "known words" dictionary

Pick FNV-1a unless you have a reason not to. It's the sweet spot of simple + good.
