# Prefix Enumeration

## Why You Need This

Your requirements say: "Return zero or more candidate words for a given prefix." Finding the prefix node is step 1. Collecting all words under that node is step 2 - and that's where prefix enumeration comes in.

## What to Learn

### The Problem

Given prefix "hel" in this trie:
```
       h
       |
       e
       |
       l
      /|\
     l i p
     |   |
     o   (end)
     |
    (end)
```

We need to collect: ["hello", "help"]

### Approach 1: Recursive Collection

```c
void collect_words(TrieNode *node, char **results, size_t *count, size_t max) {
    if (*count >= max) return;
    
    if (node->is_end) {
        results[(*count)++] = (char *)node->word;
    }
    
    for (int i = 0; i < node->child_count; i++) {
        collect_words(node->children[i], results, count, max);
    }
}

size_t trie_complete(Trie *t, const char *prefix, 
                     const char **results, size_t max_results) {
    TrieNode *node = find_prefix_node(t, prefix);
    if (!node) return 0;
    
    size_t count = 0;
    collect_words(node, results, &count, max_results);
    return count;
}
```

**Pros:** Simple, natural
**Cons:** Deep recursion on pathological input, allocates all results at once

### Approach 2: Iterative with Stack

```c
size_t trie_complete(Trie *t, const char *prefix,
                     const char **results, size_t max_results) {
    TrieNode *start = find_prefix_node(t, prefix);
    if (!start) return 0;
    
    // Stack for DFS
    TrieNode **stack = malloc(1024 * sizeof(TrieNode *));
    size_t stack_size = 0;
    size_t count = 0;
    
    stack[stack_size++] = start;
    
    while (stack_size > 0 && count < max_results) {
        TrieNode *node = stack[--stack_size];
        
        if (node->is_end) {
            results[count++] = node->word;
        }
        
        // Push children in reverse order for alphabetical traversal
        for (int i = node->child_count - 1; i >= 0; i--) {
            stack[stack_size++] = node->children[i];
        }
    }
    
    free(stack);
    return count;
}
```

**Pros:** No recursion limit issues, explicit stack
**Cons:** Manual stack management, slightly more code

### Approach 3: Callback/Iterator Pattern

```c
typedef bool (*CompletionCallback)(const char *word, void *userdata);

void trie_complete_foreach(Trie *t, const char *prefix,
                           CompletionCallback cb, void *userdata) {
    TrieNode *node = find_prefix_node(t, prefix);
    if (!node) return;
    
    enumerate_subtree(node, cb, userdata);
}

static void enumerate_subtree(TrieNode *node, 
                              CompletionCallback cb, void *userdata) {
    if (node->is_end) {
        if (!cb(node->word, userdata)) return;  // Callback can stop iteration
    }
    
    for (int i = 0; i < node->child_count; i++) {
        enumerate_subtree(node->children[i], cb, userdata);
    }
}
```

**Pros:** Caller controls memory, can stop early
**Cons:** Callback indirection, slightly awkward API

### Limiting Results

Users don't want 10,000 completions. Limit intelligently:

```c
typedef struct {
    const char **results;
    size_t count;
    size_t max;
} CollectContext;

static bool collect_callback(const char *word, void *userdata) {
    CollectContext *ctx = userdata;
    if (ctx->count >= ctx->max) return false;  // Stop
    ctx->results[ctx->count++] = word;
    return true;  // Continue
}
```

### Ordering Results

DFS gives you results in... some order. For alphabetical:

**Option 1:** Sort after collection
```c
qsort(results, count, sizeof(char *), string_compare);
```

**Option 2:** Traverse children in order (requires sorted children)

**Option 3:** Use a priority queue during collection (overkill)

For your project, traversing sorted children gives natural alphabetical order.

### Performance Considerations

**Prefix "a" on 200k word dictionary:**
- Could match 20,000+ words
- Don't collect all of them!
- Use max_results parameter (100 is plenty for UI)

**Empty prefix "":**
- Matches everything - guard against this
- Either reject or apply strict limit

```c
if (strlen(prefix) < MIN_PREFIX_LENGTH) {
    return 0;  // Require at least 2-3 characters
}
```

## Where to Learn

1. **Any algorithms course** - Tree traversal (DFS/BFS)
2. **LeetCode 208** - Implement Trie with search
3. **LeetCode 211** - Design Add and Search Words (regex completion)

## Practice Exercise

Implement completion with these features:

```c
// Returns count, fills results array
size_t trie_complete(Trie *t, const char *prefix,
                     const char **results, size_t max_results);

// Callback version
void trie_complete_foreach(Trie *t, const char *prefix,
                           bool (*callback)(const char *word, void *ctx),
                           void *ctx);
```

Test:
1. Complete "hel" → should get "hello", "help", etc.
2. Complete "" → should return 0 or limited set
3. Complete "xyz" → should return 0
4. Benchmark: complete("th") on 200k words, limit 100, should be < 1ms

## Connection to Project

This IS your completion feature. User types in editor, your CLI gets the prefix, trie_complete returns candidates. The editor (or LSP adapter) displays them. Fast, bounded, alphabetically ordered.
