# C99 Standard Features

## Why You Need This

Your requirements specify ISO C99. This isn't arbitrary - C99 introduced features that make writing clean, portable code significantly easier. You'll use these daily.

## What to Learn

### Designated Initializers
```c
// Instead of remembering positional order:
struct Config config = { .max_distance = 3, .case_sensitive = false };
```
Makes your initialization code self-documenting.

### Compound Literals
```c
// Create anonymous structs/arrays inline:
return (Result){ .success = true, .value = 42 };
```
Essential for clean return values without temp variables.

### The `restrict` Keyword
```c
void copy(char *restrict dest, const char *restrict src);
```
Tells the compiler pointers don't alias - enables optimizations. Useful in hot paths.

### Fixed-Width Types (`stdint.h`)
```c
#include <stdint.h>
uint32_t hash;    // Exactly 32 bits, always
size_t length;    // Platform's natural size type
```
Portable across platforms. No more "is int 32 bits here?" questions.

### Boolean Type (`stdbool.h`)
```c
#include <stdbool.h>
bool found = true;  // Instead of int found = 1;
```
Self-documenting intent.

### Inline Functions
```c
static inline int max(int a, int b) { return a > b ? a : b; }
```
Replace macros for simple functions - type-safe with no overhead.

### Variable-Length Arrays (VLAs)
```c
void process(int n) {
    char buffer[n];  // Size determined at runtime
}
```
**Caution:** Many projects ban VLAs (stack overflow risk). Know they exist, but prefer `malloc` for safety.

## Where to Learn

1. **Primary:** "Modern C" by Jens Gustedt - Chapters 1-5
   - Free PDF: https://gustedt.gitlabpages.inria.fr/modern-c/
   
2. **Reference:** cppreference.com C language section
   - Great for looking up specific features

3. **Quick Read:** "C99 Features" Wikipedia page
   - Good overview, links to details

## Practice Exercise

Write a small program that uses:
- At least 3 designated initializers
- One compound literal return
- `uint32_t` and `size_t` appropriately
- `bool` for flags

The program can be simple - the goal is muscle memory for the syntax.

## Connection to Project

Every file you write will use these features. Your `WordLib` context struct will use designated initializers. Your API functions will use fixed-width types for byte offsets. Getting comfortable now pays dividends forever.
