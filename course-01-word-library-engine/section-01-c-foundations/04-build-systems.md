# Build Systems

## Why You Need This

Your requirements call for "clear module boundaries." In C, that means separate compilation - each `.c` file compiles independently, then links together. A build system manages this sanity.

## What to Learn

### Separate Compilation Model

```
src/hashset.c  → hashset.o  ─┐
src/trie.c     → trie.o     ─┼──→ libwordlib.a
src/bktree.c   → bktree.o   ─┤
src/wordlib.c  → wordlib.o  ─┘
                                    ↓
cli/main.c     → main.o     ──────→ wordlib (executable)
```

Each module compiles alone, knows nothing about others except through headers.

### Makefile Fundamentals

```makefile
# Variables
CC = gcc
CFLAGS = -Wall -Wextra -std=c99 -O2
LDFLAGS = 

# Source files
SRCS = src/hashset.c src/trie.c src/bktree.c src/wordlib.c
OBJS = $(SRCS:.c=.o)

# Default target
all: wordlib

# Link executable
wordlib: $(OBJS) cli/main.o
	$(CC) $(LDFLAGS) -o $@ $^

# Compile .c to .o
%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

# Header dependencies (simplified)
src/hashset.o: src/hashset.c include/hashset.h
src/trie.o: src/trie.c include/trie.h
src/wordlib.o: src/wordlib.c include/wordlib.h include/hashset.h include/trie.h

# Clean build artifacts
clean:
	rm -f $(OBJS) cli/main.o wordlib

.PHONY: all clean
```

### Key Make Concepts

**Targets and Prerequisites:**
```makefile
target: prerequisite1 prerequisite2
	command to build target
```

**Automatic Variables:**
- `$@` - target name
- `$<` - first prerequisite
- `$^` - all prerequisites

**Pattern Rules:**
```makefile
%.o: %.c
	$(CC) -c -o $@ $<
```

### Project Directory Structure

```
triehard-workshop/
├── include/           # Public headers
│   ├── wordlib.h
│   └── wordlib_types.h
├── src/              # Implementation
│   ├── internal.h    # Private headers
│   ├── hashset.c
│   ├── trie.c
│   ├── bktree.c
│   └── wordlib.c
├── cli/              # CLI adapter
│   └── main.c
├── test/             # Tests
│   └── test_hashset.c
├── Makefile
└── README.md
```

### Compiler Flags You'll Use

```makefile
# Warnings (catch bugs early)
CFLAGS += -Wall -Wextra -Wpedantic

# Standard
CFLAGS += -std=c99

# Debug vs Release
DEBUG_FLAGS = -g -O0 -DDEBUG
RELEASE_FLAGS = -O2 -DNDEBUG

# Sanitizers (development)
CFLAGS += -fsanitize=address,undefined
```

### Header Include Paths

```makefile
CFLAGS += -Iinclude  # Add include/ to search path
```

Now `#include "wordlib.h"` finds `include/wordlib.h`.

### CMake Alternative (Brief Overview)

```cmake
cmake_minimum_required(VERSION 3.10)
project(wordlib C)

set(CMAKE_C_STANDARD 99)

add_library(wordlib_core
    src/hashset.c
    src/trie.c
    src/bktree.c
    src/wordlib.c
)
target_include_directories(wordlib_core PUBLIC include)

add_executable(wordlib cli/main.c)
target_link_libraries(wordlib wordlib_core)
```

CMake generates Makefiles (or Ninja, etc.) - more portable, more complex.

**Recommendation:** Start with Make. It's simpler and teaches you what's actually happening.

## Where to Learn

1. **"GNU Make Manual"** - Chapters 1-4 cover everything you need
2. **"Managing Projects with GNU Make" by Mecklenburg** - Great for deeper understanding
3. **Practical:** Read Makefiles from small C projects on GitHub

## Practice Exercise

Create a Makefile that:

1. Compiles `src/main.c` and `src/util.c` into `myprogram`
2. Has a `clean` target
3. Tracks header dependencies properly
4. Has separate `debug` and `release` targets with different flags

Test it:
```bash
make clean
make debug
./myprogram
make clean
make release
./myprogram
```

## Connection to Project

Your word library will have multiple modules (hashset, trie, bktree, core) that need to:
- Compile independently
- Link together
- Support debug/release builds
- Have proper header dependencies

Getting the build system right early saves endless frustration later.
