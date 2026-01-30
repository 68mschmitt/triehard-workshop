# Performance Measurement

## Why You Need This

Your requirements have specific performance targets: "O(1) average case" for membership, "efficient enough for interactive usage" for suggestions. You need to prove you meet these.

## What to Learn

### Basic Timing

```c
#include <time.h>

double measure_time(void (*fn)(void)) {
    clock_t start = clock();
    fn();
    clock_t end = clock();
    return (double)(end - start) / CLOCKS_PER_SEC * 1000.0;  // milliseconds
}

// Usage
void do_work(void) {
    for (int i = 0; i < 1000000; i++) {
        hashset_contains(hs, "test");
    }
}

double ms = measure_time(do_work);
printf("1M lookups: %.2f ms\n", ms);
```

### Higher Resolution Timing

```c
#include <sys/time.h>

typedef struct {
    struct timeval start;
    struct timeval end;
} Timer;

void timer_start(Timer *t) {
    gettimeofday(&t->start, NULL);
}

void timer_stop(Timer *t) {
    gettimeofday(&t->end, NULL);
}

double timer_elapsed_ms(Timer *t) {
    return (t->end.tv_sec - t->start.tv_sec) * 1000.0 +
           (t->end.tv_usec - t->start.tv_usec) / 1000.0;
}

double timer_elapsed_us(Timer *t) {
    return (t->end.tv_sec - t->start.tv_sec) * 1000000.0 +
           (t->end.tv_usec - t->start.tv_usec);
}
```

### Micro-Benchmark Pitfalls

**Compiler optimization:**
```c
// Bad: compiler might optimize away
for (int i = 0; i < N; i++) {
    hashset_contains(hs, "test");  // Result unused!
}

// Better: use the result
volatile int count = 0;
for (int i = 0; i < N; i++) {
    if (hashset_contains(hs, "test")) count++;
}
```

**Cache effects:**
```c
// First run: cold cache
// Subsequent runs: warm cache
// Run multiple times and discard first

for (int run = 0; run < 10; run++) {
    timer_start(&t);
    // ... work ...
    timer_stop(&t);
    times[run] = timer_elapsed_us(&t);
}
// Report median or average of runs 2-10
```

**System noise:**
```c
// Don't benchmark on a loaded system
// Disable turbo boost for consistency
// Multiple runs, report median
```

### Benchmark Suite

```c
typedef struct {
    const char *name;
    void (*setup)(void *ctx);
    void (*run)(void *ctx);
    void (*teardown)(void *ctx);
    int iterations;
} Benchmark;

void run_benchmark(Benchmark *b, void *ctx) {
    if (b->setup) b->setup(ctx);
    
    Timer t;
    timer_start(&t);
    
    for (int i = 0; i < b->iterations; i++) {
        b->run(ctx);
    }
    
    timer_stop(&t);
    
    if (b->teardown) b->teardown(ctx);
    
    double total_ms = timer_elapsed_ms(&t);
    double per_op_us = (total_ms * 1000.0) / b->iterations;
    
    printf("%-30s %8d ops  %8.2f ms  %8.3f us/op\n",
           b->name, b->iterations, total_ms, per_op_us);
}
```

### What to Benchmark

**Hash set operations:**
```c
void bench_hashset_lookup(void *ctx) {
    HashSet *hs = ctx;
    hashset_contains(hs, "test");
}

void bench_hashset_add(void *ctx) {
    HashSet *hs = ctx;
    static int counter = 0;
    char buf[32];
    snprintf(buf, sizeof(buf), "word%d", counter++);
    hashset_add(hs, buf);
}
```

**Trie completion:**
```c
void bench_trie_complete(void *ctx) {
    Trie *t = ctx;
    const char *results[100];
    trie_complete(t, "th", results, 100);
}
```

**BK-tree suggestions:**
```c
void bench_bktree_search(void *ctx) {
    BKTree *tree = ctx;
    BKMatch results[20];
    bktree_search(tree, "helo", 2, results, 20);
}
```

**Full pipeline:**
```c
void bench_check_text(void *ctx) {
    WordLib *wl = ctx;
    WLUnknownWord results[100];
    wordlib_check_text(wl, 
        "The quick brown fox jumps over the lazy dog",
        results, 100);
}
```

### Performance Report

```c
void run_all_benchmarks(void) {
    printf("=== WordLib Performance Report ===\n\n");
    printf("Dictionary size: %zu words\n", wordlib_count(wl));
    printf("\n");
    
    // Hash set benchmarks
    printf("Hash Set Operations:\n");
    run_benchmark(&bench_contains, hs);
    run_benchmark(&bench_add, hs);
    
    // Trie benchmarks
    printf("\nTrie Operations:\n");
    run_benchmark(&bench_complete_short, trie);
    run_benchmark(&bench_complete_long, trie);
    
    // BK-tree benchmarks
    printf("\nBK-Tree Operations:\n");
    run_benchmark(&bench_suggest_d1, bktree);
    run_benchmark(&bench_suggest_d2, bktree);
    
    // Integration benchmarks
    printf("\nIntegration:\n");
    run_benchmark(&bench_check_short, wl);
    run_benchmark(&bench_check_long, wl);
    
    printf("\n=== Targets ===\n");
    printf("Membership check: < 1 us (%.2f us actual)\n", actual_membership);
    printf("Completion: < 100 us (%.2f us actual)\n", actual_completion);
    printf("Suggestion: < 10 ms (%.2f ms actual)\n", actual_suggestion);
}
```

### Scaling Analysis

```c
void analyze_scaling(void) {
    printf("=== Scaling Analysis ===\n");
    
    int sizes[] = {1000, 10000, 50000, 100000, 200000};
    
    for (int i = 0; i < sizeof(sizes)/sizeof(sizes[0]); i++) {
        WordLib *wl = wordlib_create();
        load_n_words(wl, sizes[i]);
        
        Timer t;
        timer_start(&t);
        for (int j = 0; j < 10000; j++) {
            wordlib_contains(wl, "test");
        }
        timer_stop(&t);
        
        printf("  %6d words: %.3f us/lookup\n", 
               sizes[i], timer_elapsed_us(&t) / 10000);
        
        wordlib_destroy(wl);
    }
    
    // Should show roughly constant time (O(1))
}
```

## Where to Learn

1. **"Systems Performance" by Brendan Gregg** - Performance methodology
2. **Google Benchmark** - C++ library but good patterns
3. **perf tool documentation** - Linux performance analysis
4. **Agner Fog's manuals** - CPU-level optimization

## Practice Exercise

Create a benchmark suite:

```bash
$ make benchmark
=== WordLib Performance Report ===

Dictionary size: 235886 words

Hash Set Operations:
  contains (hit)                 1000000 ops     45.23 ms     0.045 us/op
  contains (miss)                1000000 ops     43.12 ms     0.043 us/op
  add (new)                       100000 ops     12.34 ms     0.123 us/op

Trie Operations:
  complete "th" (max 100)          10000 ops     48.23 ms     4.823 us/op
  complete "a" (max 100)           10000 ops     52.11 ms     5.211 us/op

BK-Tree Operations:
  suggest dist=1                   10000 ops    123.45 ms    12.345 us/op
  suggest dist=2                   10000 ops    456.78 ms    45.678 us/op

All targets met!
```

## Connection to Project

Performance benchmarks prove:
- You meet the O(1) membership requirement
- Completion is "interactive" (< 100ms)
- Suggestions are usable (< 1s)

Without measurement, you're just hoping.
