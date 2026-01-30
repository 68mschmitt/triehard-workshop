# BK-Tree Structure

## Why You Need This

Your requirements say: "Suggestions shall be implemented using a BK-tree or equivalent metric index." A naive approach checks every word in the dictionary - O(n) per query. BK-trees make this sub-linear.

## What to Learn

### The Problem with Brute Force

Dictionary of 100,000 words, find all within distance 2 of "helo":

```c
// Naive: Check every word
for (int i = 0; i < dict_size; i++) {
    if (levenshtein(query, dict[i]) <= 2) {
        add_to_results(dict[i]);
    }
}
// 100,000 distance calculations per query. Slow!
```

### The Key Insight: Triangle Inequality

For any metric distance d (including Levenshtein):
```
d(a, c) ≤ d(a, b) + d(b, c)
```

This means: if we know the distance from a query to one word, we can **eliminate** many other words without computing their distances!

### BK-Tree Structure

A tree where:
- Each node stores a word
- Children are organized by their distance to the parent
- Child at edge "k" is at distance k from parent

```
           "book" (root)
          /  |   \
        1    2    4
       /     |     \
    "look" "cook" "back"
      |
      2
      |
   "loop"
```

- d("book", "look") = 1 → "look" is child at edge 1
- d("book", "cook") = 2 → "cook" is child at edge 2
- d("look", "loop") = 2 → "loop" is child of "look" at edge 2

### Building the Tree

```c
typedef struct BKNode {
    const char *word;
    struct BKNode **children;  // Index = distance
    int max_children;          // Size of children array
} BKNode;

void bktree_insert(BKTree *tree, const char *word) {
    if (!tree->root) {
        tree->root = create_node(word);
        return;
    }
    
    BKNode *node = tree->root;
    while (true) {
        int dist = levenshtein(word, node->word);
        
        if (dist == 0) return;  // Duplicate word
        
        // Ensure children array is large enough
        if (dist >= node->max_children) {
            grow_children(node, dist + 1);
        }
        
        if (node->children[dist] == NULL) {
            node->children[dist] = create_node(word);
            return;
        }
        
        node = node->children[dist];  // Traverse to child
    }
}
```

### Searching the Tree

The magic: prune entire subtrees using triangle inequality!

```c
void bktree_search(BKTree *tree, const char *query, int max_dist,
                   void (*callback)(const char *word, int dist)) {
    if (!tree->root) return;
    search_node(tree->root, query, max_dist, callback);
}

static void search_node(BKNode *node, const char *query, int max_dist,
                        void (*callback)(const char *word, int dist)) {
    int dist = levenshtein(query, node->word);
    
    if (dist <= max_dist) {
        callback(node->word, dist);
    }
    
    // Only search children within [dist - max_dist, dist + max_dist]
    int low = (dist - max_dist > 0) ? dist - max_dist : 1;
    int high = dist + max_dist;
    
    for (int i = low; i <= high && i < node->max_children; i++) {
        if (node->children[i]) {
            search_node(node->children[i], query, max_dist, callback);
        }
    }
}
```

### Why Pruning Works

Query: "helo", max_dist: 2, at node "book":
- d("helo", "book") = 4
- For any word w reachable through "book" at edge k:
  - d("helo", w) ≥ |d("helo", "book") - d("book", w)| = |4 - k|
- Only edges where |4 - k| ≤ 2 might lead to matches: k ∈ [2, 6]
- Skip edges 1 (would need distance ≥ 3)

In practice, this eliminates 80-90% of the tree!

### Visual Example

```
Query: "cat", max_dist: 1

        "book"          d("cat","book")=4
       /  |   \         Only check edges 3,4,5
      1   2    4
          |     \
       "cook"  "back"   d("cat","back")=3
                  \     Only check edges 2,3,4
                   3
                    \
                  "bat"  d("cat","bat")=1 ✓ MATCH!
```

Without BK-tree: Check all words
With BK-tree: Check ~10-20% of words

### Complexity Analysis

- Build: O(n × k) where k = average word length
- Search: O(n^α) where α depends on max_dist and data
  - For max_dist=1: typically check 5-10% of tree
  - For max_dist=2: typically check 10-20% of tree
  - For max_dist=3: typically check 20-40% of tree

Not as good as hash table O(1), but WAY better than O(n) brute force.

## Where to Learn

1. **Original Paper:** "Some Approaches to Best-Match File Searching" by Burkhard & Keller (1973)
2. **Blog:** "BK-Trees" on algorithms sites (many good explanations)
3. **Wikipedia:** "BK-tree" article
4. **Code:** Look for BK-tree implementations in spell-checker projects

## Practice Exercise

Implement basic BK-tree:

```c
BKTree *bktree_create(void);
void bktree_destroy(BKTree *tree);
void bktree_insert(BKTree *tree, const char *word);
void bktree_search(BKTree *tree, const char *query, int max_dist,
                   void (*callback)(const char *word, int dist, void *ctx),
                   void *ctx);
```

Test:
1. Insert 10,000 words
2. Search for "helo" with max_dist 2
3. Count how many nodes were visited vs total nodes
4. Should visit < 25% of nodes for max_dist 2

## Connection to Project

This powers your spelling suggestions:
- User types unknown word "teh"
- BK-tree search finds words within distance 1-2
- Return: ["the", "tea", "ten", ...]

Combined with Levenshtein, this makes suggestions feel instant even with large dictionaries.
