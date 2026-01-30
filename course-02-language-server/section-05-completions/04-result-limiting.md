# Result Limiting

## Why You Need This

Your requirements say: "Completion requests shall return results within typical LSP latency expectations" and "Limit completion results for performance."

A dictionary might have thousands of matches for a short prefix. Returning all of them would be slow and overwhelming. Smart limiting keeps completions fast and useful.

## What to Learn

### The Performance Problem

Prefix "a" might match:
- "a", "ability", "able", "about", "above", "accept", "according", ...
- Potentially 10,000+ words in a full dictionary

Sending all of them:
- Slow to serialize as JSON
- Slow to transmit
- Slow for editor to render
- Overwhelming for user

### Strategy 1: Hard Limit

Simple cap on result count:

```c
#define MAX_COMPLETION_RESULTS 50

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // ...
    
    const char *results[MAX_COMPLETION_RESULTS];
    size_t count = wordlib_complete(server->engine, prefix, 
                                   results, MAX_COMPLETION_RESULTS);
    
    // ...
}
```

### Strategy 2: Incomplete Flag

Tell editor there are more results:

```c
JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // Try to get one more than we'll return
    const char *results[MAX_COMPLETION_RESULTS + 1];
    size_t count = wordlib_complete(server->engine, prefix, 
                                   results, MAX_COMPLETION_RESULTS + 1);
    
    // Check if there are more
    bool is_incomplete = (count > MAX_COMPLETION_RESULTS);
    if (is_incomplete) {
        count = MAX_COMPLETION_RESULTS;
    }
    
    // Build CompletionList format (not just array)
    JsonValue *result = json_object();
    json_set_bool(result, "isIncomplete", is_incomplete);
    
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        json_array_push(items, create_completion_item(results[i], i));
    }
    json_set(result, "items", items);
    
    return result;
}
```

When `isIncomplete` is true, the editor may:
- Show a "..." indicator
- Re-request when user types more characters

### Strategy 3: Minimum Prefix Length

Don't complete very short prefixes:

```c
#define MIN_PREFIX_LENGTH 2

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // ... extract prefix ...
    
    if (strlen(prefix) < MIN_PREFIX_LENGTH) {
        free(prefix);
        return json_array();  // No completions for short prefixes
    }
    
    // ... continue with completion
}
```

### Strategy 4: Relevance Ranking

Engine might already return results in relevance order. If not, rank them:

```c
typedef struct {
    const char *word;
    int score;
} RankedWord;

int compare_ranked(const void *a, const void *b) {
    const RankedWord *wa = a;
    const RankedWord *wb = b;
    return wb->score - wa->score;  // Higher scores first
}

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    const char *results[1000];
    size_t count = wordlib_complete(server->engine, prefix, results, 1000);
    
    // Rank results
    RankedWord *ranked = malloc(count * sizeof(RankedWord));
    for (size_t i = 0; i < count; i++) {
        ranked[i].word = results[i];
        ranked[i].score = compute_score(results[i], prefix);
    }
    
    qsort(ranked, count, sizeof(RankedWord), compare_ranked);
    
    // Return top N
    size_t return_count = (count > MAX_COMPLETION_RESULTS) 
                          ? MAX_COMPLETION_RESULTS : count;
    
    JsonValue *items = json_array();
    for (size_t i = 0; i < return_count; i++) {
        json_array_push(items, 
            create_completion_item(ranked[i].word, i));
    }
    
    free(ranked);
    return items;
}

int compute_score(const char *word, const char *prefix) {
    int score = 0;
    
    // Exact prefix match bonus
    if (strncmp(word, prefix, strlen(prefix)) == 0) {
        score += 100;
    }
    
    // Shorter words often more relevant
    score -= strlen(word);
    
    // Common words could get bonus (if you track frequency)
    
    return score;
}
```

### Strategy 5: Progressive Refinement

Return few results for short prefixes, more for longer:

```c
size_t get_max_results(size_t prefix_len) {
    if (prefix_len <= 1) return 10;
    if (prefix_len <= 2) return 25;
    if (prefix_len <= 3) return 50;
    return 100;  // Longer prefix = more specific, can return more
}
```

### Recommended Approach

For your spell checker, keep it simple:

```c
#define MAX_COMPLETION_RESULTS 50
#define MIN_PREFIX_LENGTH 1  // Or 2 for less noise

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    // ... extract and validate ...
    
    if (strlen(prefix) < MIN_PREFIX_LENGTH) {
        return json_array();
    }
    
    const char *results[MAX_COMPLETION_RESULTS + 1];
    size_t count = wordlib_complete(server->engine, prefix, 
                                   results, MAX_COMPLETION_RESULTS + 1);
    
    bool is_incomplete = count > MAX_COMPLETION_RESULTS;
    if (is_incomplete) count = MAX_COMPLETION_RESULTS;
    
    JsonValue *list = json_object();
    json_set_bool(list, "isIncomplete", is_incomplete);
    
    JsonValue *items = json_array();
    for (size_t i = 0; i < count; i++) {
        json_array_push(items, create_completion_item(results[i], i));
    }
    json_set(list, "items", items);
    
    return list;
}
```

### Performance Measurement

```c
#include <time.h>

JsonValue *handle_completion(Server *server, const JsonValue *params) {
    clock_t start = clock();
    
    // ... completion logic ...
    
    clock_t end = clock();
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    
    if (ms > 50) {
        log_warn("Completion took %.1f ms (prefix: %s)", ms, prefix);
    }
    
    return result;
}
```

## Where to Learn

1. **LSP Specification - CompletionList:**
   - isIncomplete flag documentation

2. **Performance profiling:**
   - Measure actual latency in your setup

3. **User studies:**
   - Research on optimal completion list sizes

## Practice Exercise

Test completion performance:

```c
void benchmark_completion(void) {
    Server *server = test_server_create();
    
    // Load a large dictionary
    load_dictionary(server->engine, "/usr/share/dict/words");
    
    // Test various prefix lengths
    const char *prefixes[] = { "a", "ab", "abc", "abcd" };
    
    for (int i = 0; i < 4; i++) {
        clock_t start = clock();
        
        const char *results[100];
        size_t count = wordlib_complete(server->engine, 
                                       prefixes[i], results, 100);
        
        clock_t end = clock();
        double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
        
        printf("Prefix '%s': %zu results in %.2f ms\n",
               prefixes[i], count, ms);
    }
    
    // All should be under 50ms
    test_server_destroy(server);
}
```

## Connection to Project

Smart limiting ensures good UX:

| Prefix | Without Limit | With Limit |
|--------|--------------|------------|
| "a" | 10,000 results, 500ms | 50 results, 5ms |
| "ab" | 500 results, 100ms | 50 results, 5ms |
| "abc" | 50 results, 10ms | 50 results, 5ms |

Users get fast, relevant completions regardless of dictionary size.
