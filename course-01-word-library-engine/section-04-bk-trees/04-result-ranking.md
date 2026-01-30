# Result Ranking

## Why You Need This

Your requirements say: "Return ranked or bounded suggestion sets." Finding words within edit distance is step 1. Presenting them in a useful order is step 2.

## What to Learn

### The Ranking Problem

Query "helo" returns:
- "hello" (distance 1)
- "help" (distance 2)
- "hero" (distance 2)
- "helm" (distance 2)
- "held" (distance 2)

All distance-2 results are "equally good" by edit distance. But "help" is more common than "helm". Users expect common words first.

### Primary Sort: Edit Distance

Always sort by distance first:

```c
int compare_by_distance(const void *a, const void *b) {
    const Suggestion *sa = a;
    const Suggestion *sb = b;
    return sa->distance - sb->distance;
}
```

Distance 1 beats distance 2, always.

### Secondary Sort: Frequency

For equal distances, prefer common words:

```c
typedef struct {
    const char *word;
    int distance;
    int frequency;  // From word frequency list
} Suggestion;

int compare_suggestions(const void *a, const void *b) {
    const Suggestion *sa = a;
    const Suggestion *sb = b;
    
    // Primary: distance
    if (sa->distance != sb->distance) {
        return sa->distance - sb->distance;
    }
    
    // Secondary: frequency (higher = better, so reverse)
    return sb->frequency - sa->frequency;
}
```

### Getting Frequency Data

**Option 1: Word frequency lists**
- Google's n-gram data
- Wiktionary frequency lists
- Project Gutenberg word counts

**Option 2: Track usage**
```c
typedef struct WordEntry {
    const char *word;
    uint32_t add_count;      // How many times user added this word
    uint32_t complete_count; // How many times selected as completion
} WordEntry;
```

**Option 3: Length heuristic (simple)**
Shorter words are often more common:
```c
// Tie-breaker: prefer shorter words
if (sa->frequency == sb->frequency) {
    return strlen(sa->word) - strlen(sb->word);
}
```

### Tertiary Sort: Alphabetical

For equal distance and frequency, alphabetical gives predictable results:

```c
int compare_suggestions(const void *a, const void *b) {
    const Suggestion *sa = a;
    const Suggestion *sb = b;
    
    if (sa->distance != sb->distance)
        return sa->distance - sb->distance;
    
    if (sa->frequency != sb->frequency)
        return sb->frequency - sa->frequency;
    
    return strcmp(sa->word, sb->word);
}
```

### Limiting Results

Users don't want 100 suggestions. Typical limits:
- Completions: 10-20 (shown in dropdown)
- Spelling suggestions: 5-10 (shown inline)

```c
size_t get_suggestions(const char *query, Suggestion *results, size_t max) {
    // Collect all matches
    Suggestion all[1000];
    size_t count = bktree_search(...);
    
    // Sort
    qsort(all, count, sizeof(Suggestion), compare_suggestions);
    
    // Return top N
    size_t to_copy = count < max ? count : max;
    memcpy(results, all, to_copy * sizeof(Suggestion));
    return to_copy;
}
```

### Early Termination Optimization

If you only need top 5, stop searching after finding 5 distance-1 matches:

```c
void search_with_early_stop(BKTree *tree, const char *query, 
                            int max_dist, int max_results) {
    int found_at_distance[MAX_DIST + 1] = {0};
    
    // ... during search ...
    if (dist <= max_dist) {
        found_at_distance[dist]++;
        
        // If we have enough distance-1 matches, stop looking
        if (found_at_distance[1] >= max_results) {
            max_dist = 1;  // Only look for distance 1 now
        }
    }
}
```

### Deduplication

Same word might be found multiple times (through different paths). Use a set:

```c
HashSet *seen = hashset_create();

void on_match(const char *word, int dist) {
    if (hashset_contains(seen, word)) return;
    hashset_add(seen, word);
    add_to_results(word, dist);
}
```

Or check before adding:
```c
for (int i = 0; i < result_count; i++) {
    if (strcmp(results[i].word, word) == 0) return;
}
```

### Example Output

Query: "teh" (common typo for "the")

```
Suggestions:
1. "the"  - distance 1, frequency: 99999999
2. "tea"  - distance 2, frequency: 50000
3. "ten"  - distance 2, frequency: 45000
4. "ted"  - distance 2, frequency: 10000 (name)
5. "tex"  - distance 2, frequency: 500
```

User almost certainly wants "the", and it's first!

## Where to Learn

1. **"Introduction to Information Retrieval"** - Chapter on spelling correction
2. **Hunspell source code** - Real-world spell checker ranking
3. **Google's "Did you mean"** - Blog posts about their approach
4. **Word frequency lists** - Google n-gram viewer data

## Practice Exercise

Implement a suggestion ranker:

```c
typedef struct {
    const char *word;
    int distance;
    int frequency;
} Suggestion;

void rank_suggestions(Suggestion *results, size_t count);
size_t get_top_suggestions(BKTree *tree, const char *query,
                           int max_dist, Suggestion *results, size_t max);
```

Test with:
1. Add words with varying frequencies
2. Query with typo
3. Verify results are sorted: distance first, then frequency

## Connection to Project

Good ranking transforms raw matches into useful suggestions. When your engine returns ["the", "tea", "ten"] for "teh" with "the" first, users trust the tool. When it returns ["teal", "tech", "ted"], they don't.
