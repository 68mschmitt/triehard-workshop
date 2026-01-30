# Section 8 Project: Final Testing & Polish

## Objective

Ensure your word library is production-ready with comprehensive testing, performance validation, and memory safety verification.

## Requirements

1. **Test coverage** - All public API functions tested
2. **Performance validated** - Meet stated requirements
3. **Memory clean** - Zero Valgrind errors
4. **Fuzz tested** - Common crash patterns caught
5. **Documentation** - README with usage examples

## Tasks

### Task 1: Complete Test Suite

Create comprehensive tests for every module:

```
test/
├── test_hashset.c       # Hash set unit tests
├── test_trie.c          # Trie unit tests  
├── test_bktree.c        # BK-tree unit tests
├── test_tokenizer.c     # Tokenizer unit tests
├── test_persistence.c   # Save/load tests
├── test_api.c           # Public API tests
├── test_integration.c   # End-to-end tests
└── test_cli.sh          # CLI behavior tests
```

**Minimum test counts:**
- Hash set: 10 tests
- Trie: 10 tests
- BK-tree: 10 tests
- Tokenizer: 8 tests
- Persistence: 6 tests
- API: 15 tests
- Integration: 5 tests

### Task 2: Performance Benchmark Suite

Create `bench/benchmark.c`:

```c
#include <stdio.h>
#include <time.h>
#include "wordlib.h"

// Benchmark configuration
#define DICT_SIZE 200000
#define ITERATIONS 100000

typedef struct {
    const char *name;
    double target_us;  // Target microseconds per operation
    double actual_us;
    bool passed;
} BenchResult;

BenchResult results[20];
int result_count = 0;

void report(const char *name, double target_us, double actual_us) {
    BenchResult *r = &results[result_count++];
    r->name = name;
    r->target_us = target_us;
    r->actual_us = actual_us;
    r->passed = actual_us <= target_us;
    
    printf("%-35s %8.3f us  (target: %8.3f us)  %s\n",
           name, actual_us, target_us, r->passed ? "PASS" : "FAIL");
}

void bench_membership(WordLib *wl) {
    clock_t start = clock();
    
    for (int i = 0; i < ITERATIONS; i++) {
        wordlib_contains(wl, "hello");
        wordlib_contains(wl, "nonexistent");
    }
    
    clock_t end = clock();
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    double us_per_op = ms * 1000 / (ITERATIONS * 2);
    
    report("Membership check", 1.0, us_per_op);  // Target: < 1 us
}

void bench_completion(WordLib *wl) {
    const char *results[100];
    
    clock_t start = clock();
    for (int i = 0; i < 10000; i++) {
        wordlib_complete(wl, "th", results, 100);
    }
    clock_t end = clock();
    
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    double us_per_op = ms * 1000 / 10000;
    
    report("Completion (prefix 'th')", 100.0, us_per_op);  // Target: < 100 us
}

void bench_suggestion(WordLib *wl) {
    WLSuggestion results[20];
    
    clock_t start = clock();
    for (int i = 0; i < 1000; i++) {
        wordlib_suggest(wl, "helo", 2, results, 20);
    }
    clock_t end = clock();
    
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    double us_per_op = ms * 1000 / 1000;
    
    report("Suggestion (dist=2)", 50000.0, us_per_op);  // Target: < 50 ms
}

void bench_check_text(WordLib *wl) {
    const char *text = "The quick brown fox jumps over the lazy dog";
    WLUnknownWord results[100];
    
    clock_t start = clock();
    for (int i = 0; i < 10000; i++) {
        wordlib_check_text(wl, text, results, 100);
    }
    clock_t end = clock();
    
    double ms = (double)(end - start) / CLOCKS_PER_SEC * 1000;
    double us_per_op = ms * 1000 / 10000;
    
    report("Check text (short)", 500.0, us_per_op);  // Target: < 500 us
}

int main(void) {
    printf("=== WordLib Performance Benchmarks ===\n\n");
    
    // Load large dictionary
    WordLib *wl = wordlib_create();
    printf("Loading dictionary...\n");
    
    FILE *f = fopen("/usr/share/dict/words", "r");
    if (f) {
        char line[256];
        while (fgets(line, sizeof(line), f)) {
            line[strcspn(line, "\n")] = '\0';
            wordlib_add_word(wl, line);
        }
        fclose(f);
    } else {
        // Generate synthetic dictionary
        for (int i = 0; i < DICT_SIZE; i++) {
            char word[32];
            snprintf(word, sizeof(word), "word%d", i);
            wordlib_add_word(wl, word);
        }
    }
    
    printf("Dictionary size: %zu words\n\n", wordlib_count(wl));
    
    // Run benchmarks
    bench_membership(wl);
    bench_completion(wl);
    bench_suggestion(wl);
    bench_check_text(wl);
    
    wordlib_destroy(wl);
    
    // Summary
    printf("\n=== Summary ===\n");
    int passed = 0;
    for (int i = 0; i < result_count; i++) {
        if (results[i].passed) passed++;
    }
    printf("%d/%d benchmarks passed\n", passed, result_count);
    
    return passed == result_count ? 0 : 1;
}
```

### Task 3: Memory Verification

Create `Makefile` targets:

```makefile
# Run all tests under Valgrind
test-valgrind:
	valgrind --leak-check=full --error-exitcode=1 ./test_all

# Run with AddressSanitizer
test-asan:
	$(CC) -fsanitize=address,undefined -g -o test_asan \
		$(TEST_SRCS) $(SRC_OBJS)
	./test_asan

# Run with Memory Sanitizer (clang only)
test-msan:
	clang -fsanitize=memory -g -o test_msan \
		$(TEST_SRCS) $(SRC_OBJS)
	./test_msan

# Full memory check
memory-check: test-valgrind test-asan
	@echo "All memory checks passed!"
```

### Task 4: Fuzz Testing

Create fuzz harnesses:

```c
// fuzz/fuzz_tokenizer.c
#include <stdint.h>
#include <stddef.h>
#include <string.h>
#include <stdlib.h>
#include "tokenizer.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    char *text = malloc(size + 1);
    if (!text) return 0;
    memcpy(text, data, size);
    text[size] = '\0';
    
    Token tokens[100];
    TokenizerConfig cfg = tokenizer_default_config();
    
    // Test with various configs
    cfg.include_apostrophes = true;
    tokenize(text, &cfg, tokens, 100);
    
    cfg.include_apostrophes = false;
    cfg.include_hyphens = true;
    tokenize(text, &cfg, tokens, 100);
    
    free(text);
    return 0;
}
```

```c
// fuzz/fuzz_load.c
#include <stdint.h>
#include <stddef.h>
#include <stdio.h>
#include "wordlib.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // Write to temp file
    FILE *f = fopen("/tmp/fuzz.txt", "wb");
    if (!f) return 0;
    fwrite(data, 1, size, f);
    fclose(f);
    
    // Try to load
    WordLib *wl = wordlib_create();
    wordlib_load(wl, "/tmp/fuzz.txt");
    wordlib_destroy(wl);
    
    return 0;
}
```

### Task 5: Documentation

Create `README.md`:

```markdown
# WordLib - Personal Word Library Engine

Fast, extensible word library for note-taking workflows.

## Features

- O(1) word membership checks
- Prefix-based auto-completion
- Spelling suggestions using edit distance
- UTF-8 text tokenization
- Persistent dictionary storage
- Clean C API for easy integration

## Building

```bash
make
make test
make benchmark
```

## Usage

### CLI

```bash
# Add words
wordlib add hello world café

# Check text for unknown words
wordlib check "The quikc brown fox"
echo "some text" | wordlib check

# Get completions
wordlib complete hel

# Get spelling suggestions  
wordlib suggest helo

# List all words
wordlib list
```

### C API

```c
#include "wordlib.h"

// Create library
WordLib *wl = wordlib_create();

// Load dictionary
wordlib_load(wl, "~/.wordlib/dict.txt");

// Add words
wordlib_add_word(wl, "hello");

// Check membership
if (wordlib_contains(wl, "hello")) {
    printf("Found!\n");
}

// Get completions
const char *results[20];
size_t count = wordlib_complete(wl, "hel", results, 20);

// Check text
WLUnknownWord unknown[100];
count = wordlib_check_text(wl, "some text to check", unknown, 100);

// Save and cleanup
wordlib_save(wl, "~/.wordlib/dict.txt");
wordlib_destroy(wl);
```

## Performance

Tested with 200,000 word dictionary:

| Operation | Time |
|-----------|------|
| Membership check | < 1 μs |
| Completion (100 results) | < 100 μs |
| Suggestion (dist=2) | < 50 ms |
| Text check (sentence) | < 500 μs |

## Future: LSP Integration

The engine is designed for future LSP server implementation. 
Byte-level spans enable direct mapping to LSP positions.

## License

MIT
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test
Running hash set tests... PASS
Running trie tests... PASS
Running BK-tree tests... PASS
Running tokenizer tests... PASS
Running persistence tests... PASS
Running API tests... PASS
Running integration tests... PASS
Running CLI tests... PASS
All 64 tests passed!
```

### Memory Clean
```bash
$ make test-valgrind
...
All heap blocks were freed -- no leaks are possible
ERROR SUMMARY: 0 errors from 0 contexts
```

### Performance Targets Met
```bash
$ make benchmark
=== WordLib Performance Benchmarks ===

Dictionary size: 235886 words

Membership check                      0.045 us  (target:    1.000 us)  PASS
Completion (prefix 'th')             12.345 us  (target:  100.000 us)  PASS
Suggestion (dist=2)               34567.890 us  (target: 50000.000 us)  PASS
Check text (short)                  234.567 us  (target:  500.000 us)  PASS

=== Summary ===
4/4 benchmarks passed
```

### Fuzz Clean
```bash
$ make fuzz FUZZ_TIME=60
Running tokenizer fuzzer for 60s... No crashes
Running loader fuzzer for 60s... No crashes
Fuzz testing complete!
```

### CLI Functional
```bash
$ ./wordlib --help
# Shows usage

$ echo "hello world" | ./wordlib check
# Works

$ ./wordlib --json complete hel
# Returns valid JSON
```

## Final Checklist

- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] CLI tests pass
- [ ] Valgrind shows zero leaks
- [ ] ASan shows zero errors
- [ ] All performance targets met
- [ ] Fuzz testing completed (60s minimum each target)
- [ ] README is complete and accurate
- [ ] Code is formatted consistently
- [ ] No compiler warnings with `-Wall -Wextra -Wpedantic`

## Congratulations!

If you've completed all sections and this final checklist, you have built:

1. **A production-quality hash set** with O(1) operations
2. **A trie** for fast prefix completion
3. **A BK-tree** for fuzzy string matching
4. **A UTF-8 tokenizer** that handles international text
5. **Atomic file persistence** that won't corrupt data
6. **A clean C API** suitable for multiple adapters
7. **A functional CLI** that integrates with editors

Your word library engine is ready for the next step: building an LSP server!

🎉 **Well done!**
