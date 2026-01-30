# Memory Debugging

## Why You Need This

C has no garbage collector. Every allocation must be freed, exactly once. Memory bugs are silent killers - they might work fine in testing and crash in production.

## What to Learn

### Common Memory Bugs

**Memory leak:**
```c
void process() {
    char *data = malloc(1024);
    if (error) return;  // Leaked!
    free(data);
}
```

**Double free:**
```c
free(ptr);
// ... later ...
free(ptr);  // Undefined behavior!
```

**Use after free:**
```c
free(ptr);
printf("%s\n", ptr);  // Undefined behavior!
```

**Buffer overflow:**
```c
char buf[10];
strcpy(buf, "this string is too long");  // Overflow!
```

**Uninitialized memory:**
```c
int *arr = malloc(100 * sizeof(int));
printf("%d\n", arr[50]);  // Uninitialized value!
```

### Valgrind

**Basic usage:**
```bash
valgrind ./your_program
```

**Leak check:**
```bash
valgrind --leak-check=full ./your_program
```

**Track origins:**
```bash
valgrind --leak-check=full --track-origins=yes ./your_program
```

**Example output:**
```
==12345== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x4C2FB0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==12345==    by 0x108B62: hashset_create (hashset.c:25)
==12345==    by 0x108A12: test_leak (test.c:10)
==12345==    by 0x108943: main (test.c:50)
```

### AddressSanitizer (ASan)

**Compile with ASan:**
```bash
gcc -fsanitize=address -g -o program program.c
```

**Also useful:**
```bash
gcc -fsanitize=address,undefined -g -o program program.c
```

**Example output:**
```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60200000001a
READ of size 1 at 0x60200000001a thread T0
    #0 0x4c3b5f in test_overflow test.c:15
    #1 0x4c3d2a in main test.c:50
```

**Advantages over Valgrind:**
- Much faster (2-5x slowdown vs 20-50x)
- Can be used in CI/CD
- Catches different bugs

### Memory Sanitizer (MSan)

Finds uninitialized memory reads:

```bash
clang -fsanitize=memory -g -o program program.c
./program
```

Note: MSan requires clang and has some limitations.

### Defensive Patterns

**Set freed pointers to NULL:**
```c
void safe_free(void **ptr) {
    if (ptr && *ptr) {
        free(*ptr);
        *ptr = NULL;
    }
}

// Usage
safe_free((void **)&data);
// Now data == NULL, double-free is detectable
```

**Zero before free (security):**
```c
void secure_free(void *ptr, size_t size) {
    if (ptr) {
        memset(ptr, 0, size);
        free(ptr);
    }
}
```

**Canary values:**
```c
typedef struct {
    uint32_t canary_start;
    char data[256];
    uint32_t canary_end;
} GuardedBuffer;

#define CANARY_VALUE 0xDEADBEEF

void check_canaries(GuardedBuffer *buf) {
    assert(buf->canary_start == CANARY_VALUE);
    assert(buf->canary_end == CANARY_VALUE);
}
```

### Debug Allocator

```c
#ifdef DEBUG
#include <stdio.h>

static size_t g_alloc_count = 0;
static size_t g_free_count = 0;
static size_t g_bytes_allocated = 0;

void *debug_malloc(size_t size, const char *file, int line) {
    void *ptr = malloc(size);
    if (ptr) {
        g_alloc_count++;
        g_bytes_allocated += size;
        fprintf(stderr, "ALLOC: %zu bytes at %p (%s:%d)\n", 
                size, ptr, file, line);
    }
    return ptr;
}

void debug_free(void *ptr, const char *file, int line) {
    if (ptr) {
        g_free_count++;
        fprintf(stderr, "FREE: %p (%s:%d)\n", ptr, file, line);
    }
    free(ptr);
}

void debug_report(void) {
    fprintf(stderr, "Memory report: %zu allocs, %zu frees, %zu bytes\n",
            g_alloc_count, g_free_count, g_bytes_allocated);
    if (g_alloc_count != g_free_count) {
        fprintf(stderr, "WARNING: Potential leak!\n");
    }
}

#define malloc(size) debug_malloc(size, __FILE__, __LINE__)
#define free(ptr) debug_free(ptr, __FILE__, __LINE__)
#endif
```

### CI Integration

```makefile
# Makefile
test: test-normal test-asan test-valgrind

test-normal:
	$(CC) -o test_runner $(TEST_SRCS) $(LIB)
	./test_runner

test-asan:
	$(CC) -fsanitize=address,undefined -o test_asan $(TEST_SRCS) $(LIB)
	./test_asan

test-valgrind:
	$(CC) -g -o test_valgrind $(TEST_SRCS) $(LIB)
	valgrind --leak-check=full --error-exitcode=1 ./test_valgrind
```

### Hunting a Specific Leak

```bash
# Run with verbose leak info
valgrind --leak-check=full --show-reachable=yes \
         --num-callers=20 ./program

# Save report to file
valgrind --leak-check=full --log-file=valgrind.txt ./program
```

Look for:
- "definitely lost" - true leaks
- "indirectly lost" - leaked via pointer in leaked block
- "possibly lost" - might be leak, might be false positive
- "still reachable" - not freed but pointer exists (often OK)

## Where to Learn

1. **Valgrind documentation** - Complete guide
2. **LLVM Sanitizers documentation** - ASan, MSan, UBSan
3. **"The Art of Debugging"** - Memory debugging techniques
4. **"Secure Coding in C and C++"** - Memory safety

## Practice Exercise

1. Intentionally introduce each type of memory bug
2. Detect with Valgrind
3. Detect with ASan
4. Fix and verify clean

Run your entire test suite under Valgrind:
```bash
valgrind --leak-check=full --error-exitcode=1 ./test_all
```

Zero errors = passing grade.

## Connection to Project

Memory bugs are:
- Silent in development
- Catastrophic in production
- Hard to debug without tools

Valgrind-clean code is non-negotiable for a production tool.
