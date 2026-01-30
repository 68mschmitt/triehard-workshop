# Alternative Approaches

## Why You Need This

BK-trees are the classic solution, but they're not the only option. Understanding alternatives helps you make informed tradeoffs and gives you fallback options if BK-trees don't perform well for your use case.

## What to Learn

### SymSpell Algorithm

**Key insight:** Pre-compute all "deletes" up to max edit distance.

For word "hello" with max_dist=2:
- Delete 1 char: "ello", "hllo", "helo", "hell"
- Delete 2 chars: "llo", "hlo", "heo", "hel", "elo", "ell", etc.

Store all these in a hash map pointing to original word.

**Lookup:** Generate deletes of query, look up in hash map.

```c
// Conceptual - not actual implementation
void symspell_add(SymSpell *ss, const char *word) {
    hashmap_add(ss->words, word, word);
    generate_deletes(word, ss->max_dist, deletes);
    for (each delete in deletes) {
        hashmap_add(ss->deletes, delete, word);
    }
}

void symspell_suggest(SymSpell *ss, const char *query) {
    // Check if query itself is valid
    if (hashmap_contains(ss->words, query)) {
        add_suggestion(query, 0);
    }
    
    // Check deletes of query
    generate_deletes(query, ss->max_dist, deletes);
    for (each delete in deletes) {
        candidates = hashmap_get_all(ss->deletes, delete);
        for (each candidate) {
            dist = levenshtein(query, candidate);
            if (dist <= max_dist) add_suggestion(candidate, dist);
        }
    }
}
```

**Pros:** Extremely fast lookup (O(1) hash lookups)
**Cons:** High memory (stores all delete variants), expensive preprocessing

**Best for:** Static dictionaries, when lookup speed is critical

### Trigram Index

**Key insight:** Similar words share character trigrams.

"hello" → trigrams: "#he", "hel", "ell", "llo", "lo#"

Store inverted index: trigram → words containing it.

```c
void trigram_add(TrigramIndex *idx, const char *word) {
    char trigrams[MAX_TRIGRAMS][4];
    int count = extract_trigrams(word, trigrams);
    
    for (int i = 0; i < count; i++) {
        add_to_inverted_index(idx, trigrams[i], word);
    }
}

void trigram_suggest(TrigramIndex *idx, const char *query) {
    char trigrams[MAX_TRIGRAMS][4];
    int count = extract_trigrams(query, trigrams);
    
    // Find words sharing many trigrams
    HashMap *candidates = hashmap_create();
    for (int i = 0; i < count; i++) {
        words = get_words_with_trigram(idx, trigrams[i]);
        for (each word) {
            increment_count(candidates, word);
        }
    }
    
    // Words with high overlap are likely similar
    for (each candidate with count > threshold) {
        dist = levenshtein(query, candidate);
        if (dist <= max_dist) add_suggestion(candidate, dist);
    }
}
```

**Pros:** Works well for longer words, good for partial matching
**Cons:** Less effective for short words, more false positives

**Best for:** Full-text search, document similarity

### Linear Scan with Bounded Levenshtein

Sometimes brute force wins:

```c
void linear_suggest(Dictionary *dict, const char *query, int max_dist) {
    for (int i = 0; i < dict->count; i++) {
        int dist = levenshtein_bounded(query, dict->words[i], max_dist);
        if (dist <= max_dist) {
            add_suggestion(dict->words[i], dist);
        }
    }
}
```

**Pros:** Simple, no preprocessing, constant memory
**Cons:** O(n) always

**Best for:** Small dictionaries (< 10k words), or when memory is constrained

### Levenshtein Automaton

Build a finite automaton that accepts all strings within distance k of query.

```
Query "cat", distance 1:

States represent (position, errors_so_far)
Transitions on each character
Accept states = reached end with ≤1 error
```

**Pros:** Very fast for searching sorted lists, can integrate with tries
**Cons:** Complex to implement, automaton size grows exponentially with distance

**Best for:** When you need to search a sorted word list or trie

### VP-Trees (Vantage Point Trees)

Like BK-trees but binary (two children instead of n):

```c
struct VPNode {
    const char *word;
    int threshold;         // Median distance to children
    struct VPNode *inside; // dist <= threshold
    struct VPNode *outside; // dist > threshold
};
```

**Pros:** More balanced than BK-trees, works for any metric
**Cons:** Slightly more complex, similar performance to BK-trees

### Comparison Matrix

| Approach | Build Time | Query Time | Memory | Best For |
|----------|-----------|------------|--------|----------|
| BK-Tree | O(n log n) | O(n^α) | O(n) | General use |
| SymSpell | O(n × d!) | O(1) | O(n × d!) | Speed critical |
| Trigrams | O(n × w) | O(candidates) | O(n × w) | Long words |
| Linear | O(1) | O(n) | O(n) | Small dicts |
| Automaton | O(3^k) | O(n) | O(3^k) | With tries |

Where n = words, w = word length, d = max distance, k = max edit distance.

## Where to Learn

1. **SymSpell GitHub:** Wolf Garbe's implementation with great documentation
2. **"Managing Gigabytes"** - Text search algorithms including trigrams
3. **"Levenshtein Automata" paper** by Schulz and Mihov
4. **Blog:** "Spell Checking in Lucene" - Real-world comparison

## Recommendation for Your Project

**Start with BK-tree:**
- Good balance of implementation complexity and performance
- Works well for typical dictionary sizes (10k-100k words)
- Easy to understand and debug

**Consider SymSpell if:**
- Your dictionary is static (loaded once at startup)
- You have memory to spare
- Lookup speed is critical (< 1ms for suggestions)

**Avoid for now:**
- Levenshtein automata (complexity not worth it for your use case)
- Trigrams (better for document search than spell checking)

## Connection to Project

You'll implement BK-tree in the section project. But knowing these alternatives means:
1. You can benchmark and compare
2. If BK-tree disappoints, you have options
3. You understand the tradeoffs in your design
