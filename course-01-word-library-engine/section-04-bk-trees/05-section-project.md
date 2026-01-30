# Section 4 Project: BK-Tree Implementation

## Objective

Build a BK-tree for fuzzy string matching that powers your spelling suggestions feature.

## Requirements

1. **Sub-linear search** - Don't check every word
2. **Configurable max distance** - Support distance 1, 2, or 3
3. **Ranked results** - Sort by distance, then alphabetically
4. **Bounded results** - Limit output count
5. **Memory safe** - Valgrind clean

## Specification

### Public API (`include/bktree.h`)

```c
typedef struct BKTree BKTree;

typedef struct {
    const char *word;
    int distance;
} BKMatch;

// Lifecycle
BKTree *bktree_create(void);
void bktree_destroy(BKTree *tree);

// Core operations
bool bktree_insert(BKTree *tree, const char *word);
size_t bktree_count(const BKTree *tree);

// Search
size_t bktree_search(const BKTree *tree, const char *query, int max_dist,
                     BKMatch *results, size_t max_results);

// Iteration (for testing)
typedef void (*BKTreeIterator)(const char *word, void *userdata);
void bktree_foreach(const BKTree *tree, BKTreeIterator fn, void *userdata);

// Stats (for debugging)
typedef struct {
    size_t total_nodes;
    size_t nodes_visited;
    size_t comparisons;
} BKTreeStats;

size_t bktree_search_with_stats(const BKTree *tree, const char *query, 
                                 int max_dist, BKMatch *results, 
                                 size_t max_results, BKTreeStats *stats);
```

### Internal Structure (recommended)

```c
// src/bktree.c
typedef struct BKNode {
    const char *word;
    struct BKNode **children;  // Index = distance from this node's word
    int child_capacity;        // Size of children array
} BKNode;

struct BKTree {
    BKNode *root;
    size_t node_count;
};

// Levenshtein implementation
static int levenshtein(const char *s1, const char *s2);
static int levenshtein_bounded(const char *s1, const char *s2, int max);
```

### Behavior Specifications

**bktree_insert:**
- Returns `true` if word was newly inserted
- Returns `false` if word already existed (distance 0 from existing node)
- Builds tree structure based on edit distances

**bktree_search:**
- Finds all words within `max_dist` of query
- Returns matches sorted by distance, then alphabetically
- Returns at most `max_results` matches
- Returns actual count found (may be less than max)

**levenshtein_bounded:**
- Returns actual distance if ≤ max
- Returns max+1 if distance exceeds max (early termination)

## Test Cases

Create `test/test_bktree.c`:

```c
void test_basic_operations(void) {
    BKTree *tree = bktree_create();
    
    assert(bktree_insert(tree, "hello"));
    assert(bktree_insert(tree, "help"));
    assert(bktree_insert(tree, "world"));
    assert(bktree_count(tree) == 3);
    
    // Duplicate
    assert(!bktree_insert(tree, "hello"));
    assert(bktree_count(tree) == 3);
    
    bktree_destroy(tree);
}

void test_exact_match(void) {
    BKTree *tree = bktree_create();
    bktree_insert(tree, "hello");
    bktree_insert(tree, "world");
    
    BKMatch results[10];
    size_t count = bktree_search(tree, "hello", 0, results, 10);
    
    assert(count == 1);
    assert(strcmp(results[0].word, "hello") == 0);
    assert(results[0].distance == 0);
    
    bktree_destroy(tree);
}

void test_distance_one(void) {
    BKTree *tree = bktree_create();
    bktree_insert(tree, "hello");
    bktree_insert(tree, "hallo");  // 1 sub
    bktree_insert(tree, "hell");   // 1 del
    bktree_insert(tree, "hellos"); // 1 ins
    bktree_insert(tree, "world");  // far
    
    BKMatch results[10];
    size_t count = bktree_search(tree, "hello", 1, results, 10);
    
    assert(count == 4);  // hello, hallo, hell, hellos
    assert(results[0].distance == 0);  // hello first
    
    bktree_destroy(tree);
}

void test_distance_two(void) {
    BKTree *tree = bktree_create();
    bktree_insert(tree, "cat");
    bktree_insert(tree, "car");   // d=1
    bktree_insert(tree, "cart");  // d=2
    bktree_insert(tree, "bat");   // d=1
    bktree_insert(tree, "dog");   // d=3
    
    BKMatch results[10];
    size_t count = bktree_search(tree, "cat", 2, results, 10);
    
    assert(count == 4);  // cat, car, bat, cart (not dog)
    
    bktree_destroy(tree);
}

void test_sorting(void) {
    BKTree *tree = bktree_create();
    bktree_insert(tree, "apple");
    bktree_insert(tree, "apply");   // d=1
    bktree_insert(tree, "ape");     // d=2
    bktree_insert(tree, "apples");  // d=1
    
    BKMatch results[10];
    size_t count = bktree_search(tree, "apple", 2, results, 10);
    
    assert(count == 4);
    // Distance 0 first
    assert(results[0].distance == 0);
    // Then distance 1, alphabetically
    assert(results[1].distance == 1);
    assert(results[2].distance == 1);
    assert(strcmp(results[1].word, "apples") < 0 || 
           strcmp(results[2].word, "apples") < 0);
    // Then distance 2
    assert(results[3].distance == 2);
    
    bktree_destroy(tree);
}

void test_limited_results(void) {
    BKTree *tree = bktree_create();
    
    // Insert many similar words
    char buf[32];
    for (int i = 0; i < 100; i++) {
        snprintf(buf, sizeof(buf), "test%d", i);
        bktree_insert(tree, strdup(buf));
    }
    
    BKMatch results[5];
    size_t count = bktree_search(tree, "test", 2, results, 5);
    
    assert(count == 5);  // Limited to 5
    
    bktree_destroy(tree);
}

void test_pruning_efficiency(void) {
    BKTree *tree = bktree_create();
    
    // Load dictionary
    FILE *f = fopen("/usr/share/dict/words", "r");
    if (!f) {
        printf("Skipping - dictionary not found\n");
        bktree_destroy(tree);
        return;
    }
    
    char line[256];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = 0;
        if (strlen(line) > 0) {
            bktree_insert(tree, strdup(line));
        }
    }
    fclose(f);
    
    printf("Loaded %zu words\n", bktree_count(tree));
    
    BKMatch results[20];
    BKTreeStats stats = {0};
    
    // Search with stats
    size_t count = bktree_search_with_stats(tree, "helo", 2, 
                                            results, 20, &stats);
    
    printf("Found %zu matches\n", count);
    printf("Nodes visited: %zu / %zu (%.1f%%)\n", 
           stats.nodes_visited, stats.total_nodes,
           100.0 * stats.nodes_visited / stats.total_nodes);
    printf("Distance comparisons: %zu\n", stats.comparisons);
    
    // Should visit much less than 100% of nodes
    assert(stats.nodes_visited < stats.total_nodes * 0.5);
    
    bktree_destroy(tree);
}

void test_levenshtein(void) {
    // Basic tests
    assert(levenshtein("", "") == 0);
    assert(levenshtein("hello", "hello") == 0);
    assert(levenshtein("hello", "helo") == 1);
    assert(levenshtein("kitten", "sitting") == 3);
    assert(levenshtein("", "abc") == 3);
    assert(levenshtein("abc", "") == 3);
    
    // Bounded version
    assert(levenshtein_bounded("kitten", "sitting", 2) == 3);  // > max
    assert(levenshtein_bounded("kitten", "sitting", 3) == 3);
    assert(levenshtein_bounded("hello", "helo", 1) == 1);
    
    printf("Levenshtein tests passed\n");
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_bktree
Running BK-tree tests...
test_levenshtein: PASS
test_basic_operations: PASS
test_exact_match: PASS
test_distance_one: PASS
test_distance_two: PASS
test_sorting: PASS
test_limited_results: PASS
test_pruning_efficiency: Loaded 235886 words
  Found 15 matches
  Nodes visited: 28451 / 235886 (12.1%)
  Distance comparisons: 28451
  PASS
All tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_bktree
...
All heap blocks were freed -- no leaks are possible
```

### Performance
- Search (max_dist=2) on 200k words: < 50ms
- Should visit < 30% of nodes for max_dist=2

## Stretch Goals

1. **Frequency-based ranking** - Add frequency field, sort by distance then frequency
2. **Deletion support** - Mark nodes as deleted (lazy deletion)
3. **Persistence** - Save/load tree structure to file
4. **Alternative metric** - Support Damerau-Levenshtein (transpositions)

## Implementation Hints

### Growing Children Array
```c
static void grow_children(BKNode *node, int needed) {
    if (needed <= node->child_capacity) return;
    
    int new_cap = node->child_capacity ? node->child_capacity * 2 : 8;
    while (new_cap <= needed) new_cap *= 2;
    
    node->children = realloc(node->children, new_cap * sizeof(BKNode *));
    for (int i = node->child_capacity; i < new_cap; i++) {
        node->children[i] = NULL;
    }
    node->child_capacity = new_cap;
}
```

### Search Range Calculation
```c
// Only search children at edges [dist - max_dist, dist + max_dist]
int low = dist - max_dist;
if (low < 1) low = 1;  // Distance 0 would be duplicate
int high = dist + max_dist;
```

### Sorting Results
```c
static int compare_matches(const void *a, const void *b) {
    const BKMatch *ma = a;
    const BKMatch *mb = b;
    if (ma->distance != mb->distance)
        return ma->distance - mb->distance;
    return strcmp(ma->word, mb->word);
}

qsort(results, count, sizeof(BKMatch), compare_matches);
```

## What You'll Learn

By completing this project:
- BK-tree data structure implementation
- Edit distance algorithms with optimization
- Tree pruning and search space reduction
- Performance measurement and analysis

## Next Section Preview

Section 5 tackles UTF-8 text processing. Your engine needs to tokenize "I went to café" correctly, handling multi-byte characters without corrupting them. This is where the rubber meets the road for real-world text.
