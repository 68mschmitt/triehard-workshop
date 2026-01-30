# Unit Testing in C

## Why You Need This

Your requirements mention "testability" and you've been writing tests throughout. Now let's formalize testing practices for your complete project.

## What to Learn

### Simple Assertion-Based Tests

```c
#include <assert.h>
#include <stdio.h>

void test_addition(void) {
    assert(1 + 1 == 2);
    assert(0 + 0 == 0);
    assert(-1 + 1 == 0);
    printf("test_addition: PASS\n");
}

int main(void) {
    test_addition();
    printf("All tests passed!\n");
    return 0;
}
```

### Better Assertions with Messages

```c
#define ASSERT_MSG(cond, msg) do { \
    if (!(cond)) { \
        fprintf(stderr, "FAIL: %s\n  %s:%d\n  %s\n", \
                msg, __FILE__, __LINE__, #cond); \
        exit(1); \
    } \
} while(0)

#define ASSERT_EQ(a, b) ASSERT_MSG((a) == (b), "Expected equal")
#define ASSERT_NE(a, b) ASSERT_MSG((a) != (b), "Expected not equal")
#define ASSERT_TRUE(x) ASSERT_MSG((x), "Expected true")
#define ASSERT_FALSE(x) ASSERT_MSG(!(x), "Expected false")
#define ASSERT_NULL(x) ASSERT_MSG((x) == NULL, "Expected NULL")
#define ASSERT_NOT_NULL(x) ASSERT_MSG((x) != NULL, "Expected not NULL")
#define ASSERT_STR_EQ(a, b) ASSERT_MSG(strcmp(a, b) == 0, "Expected strings equal")
```

### Test Organization

```c
// test/test_hashset.c

#include "test_common.h"
#include "hashset.h"

static void test_create_destroy(void) {
    HashSet *hs = hashset_create();
    ASSERT_NOT_NULL(hs);
    ASSERT_EQ(hashset_count(hs), 0);
    hashset_destroy(hs);
    PASS();
}

static void test_add_contains(void) {
    HashSet *hs = hashset_create();
    
    ASSERT_TRUE(hashset_add(hs, "hello"));
    ASSERT_TRUE(hashset_contains(hs, "hello"));
    ASSERT_FALSE(hashset_contains(hs, "world"));
    
    hashset_destroy(hs);
    PASS();
}

// ... more tests ...

int main(void) {
    RUN_TEST(test_create_destroy);
    RUN_TEST(test_add_contains);
    // ...
    
    PRINT_SUMMARY();
    return g_test_failures > 0 ? 1 : 0;
}
```

### Simple Test Framework

```c
// test/test_common.h

#ifndef TEST_COMMON_H
#define TEST_COMMON_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int g_tests_run = 0;
static int g_test_failures = 0;
static const char *g_current_test = NULL;

#define PASS() do { \
    printf("  %s: PASS\n", g_current_test); \
} while(0)

#define FAIL(msg) do { \
    printf("  %s: FAIL - %s\n", g_current_test, msg); \
    g_test_failures++; \
} while(0)

#define RUN_TEST(fn) do { \
    g_current_test = #fn; \
    g_tests_run++; \
    fn(); \
} while(0)

#define PRINT_SUMMARY() do { \
    printf("\n%d tests run, %d failures\n", \
           g_tests_run, g_test_failures); \
} while(0)

// Assertions
#define ASSERT(cond) do { \
    if (!(cond)) { \
        printf("  ASSERT FAILED: %s\n    at %s:%d\n", \
               #cond, __FILE__, __LINE__); \
        g_test_failures++; \
        return; \
    } \
} while(0)

#define ASSERT_EQ(a, b) ASSERT((a) == (b))
#define ASSERT_TRUE(x) ASSERT(x)
#define ASSERT_FALSE(x) ASSERT(!(x))
#define ASSERT_STR_EQ(a, b) ASSERT(strcmp(a, b) == 0)

#endif // TEST_COMMON_H
```

### Test Categories

**Unit tests:** Test individual functions in isolation
```c
void test_fnv1a_hash(void) {
    ASSERT_EQ(fnv1a(""), 2166136261u);
    ASSERT_EQ(fnv1a("a"), 3826002220u);
    // Different inputs should have different hashes
    ASSERT_NE(fnv1a("hello"), fnv1a("world"));
    PASS();
}
```

**Integration tests:** Test components working together
```c
void test_add_then_complete(void) {
    WordLib *wl = wordlib_create();
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "help");
    
    const char *results[10];
    size_t count = wordlib_complete(wl, "hel", results, 10);
    
    ASSERT_EQ(count, 2);
    wordlib_destroy(wl);
    PASS();
}
```

**Edge case tests:** Test boundary conditions
```c
void test_empty_inputs(void) {
    WordLib *wl = wordlib_create();
    
    ASSERT_EQ(wordlib_add_word(wl, ""), 0);  // Empty word rejected
    ASSERT_EQ(wordlib_complete(wl, "", NULL, 0), 0);  // Empty prefix
    
    wordlib_destroy(wl);
    PASS();
}

void test_null_safety(void) {
    // Should not crash
    ASSERT_EQ(wordlib_add_word(NULL, "test"), -1);
    ASSERT_FALSE(wordlib_contains(NULL, "test"));
    PASS();
}
```

### Makefile Integration

```makefile
TEST_SRCS = test/test_hashset.c test/test_trie.c test/test_bktree.c \
            test/test_tokenizer.c test/test_persistence.c test/test_api.c
TEST_BINS = $(TEST_SRCS:.c=)

test: $(TEST_BINS)
	@for t in $(TEST_BINS); do \
		echo "Running $$t..."; \
		./$$t || exit 1; \
	done
	@echo "All tests passed!"

test/%: test/%.c $(LIB)
	$(CC) $(CFLAGS) -o $@ $< $(LIB) $(LDFLAGS)

.PHONY: test
```

### Test Data Files

```c
// Create test data directory
void setup_test_data(void) {
    mkdir("test_data", 0755);
    
    // Create a test dictionary file
    FILE *f = fopen("test_data/small_dict.txt", "w");
    fprintf(f, "hello\nworld\ntest\n");
    fclose(f);
}

void teardown_test_data(void) {
    unlink("test_data/small_dict.txt");
    rmdir("test_data");
}
```

### Continuous Integration Check

```c
// Run all tests, return non-zero on failure
int main(void) {
    test_hashset();
    test_trie();
    test_bktree();
    test_tokenizer();
    test_persistence();
    test_api();
    
    if (g_test_failures > 0) {
        printf("\n%d FAILURES\n", g_test_failures);
        return 1;
    }
    
    printf("\nAll tests passed!\n");
    return 0;
}
```

## Where to Learn

1. **Unity Test Framework** - Popular C testing framework
2. **Check** - Another C testing framework
3. **"Test-Driven Development" by Kent Beck** - TDD concepts
4. **Google Test (gtest)** - C++ but instructive patterns

## Practice Exercise

Create a test suite for your project:

1. One test file per module
2. At least 5 tests per module
3. Cover happy path, edge cases, and error conditions
4. Run with `make test`

Aim for tests that:
- Run fast (< 1 second total)
- Are independent (order doesn't matter)
- Clean up after themselves

## Connection to Project

Tests are your safety net:
- Refactor with confidence
- Catch regressions immediately
- Document expected behavior
- Enable CI/CD
