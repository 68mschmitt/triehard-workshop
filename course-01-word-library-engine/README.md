# Course 1: Word Library Engine

**Build a production-quality word library in C from scratch**

---

## Overview

This course takes you from C fundamentals to a complete, LSP-ready word library engine. You'll implement three core data structures (hash set, trie, BK-tree), learn UTF-8 text processing, build crash-safe persistence, and create a polished CLI tool.

By the end, you'll have a working system that can:
- Track thousands of words with O(1) lookup
- Provide instant prefix-based auto-completion
- Suggest spelling corrections using edit distance
- Tokenize UTF-8 text and identify unknown words
- Persist data safely across sessions
- Integrate with editors via command-line interface

---

## Target Audience

Software engineers who want to:
- Deepen their C programming skills
- Understand data structures at the implementation level
- Build a practical tool they'll actually use
- Prepare for systems programming work

**Prerequisites:** Basic programming experience. Prior C exposure helpful but not required.

---

## Course Structure

| Section | Focus | You'll Build |
|---------|-------|--------------|
| 1 | C Foundations | Project skeleton with opaque types |
| 2 | Hash Tables | O(1) lookup dictionary |
| 3 | Tries | Prefix completion engine |
| 4 | BK-Trees | Fuzzy matching for suggestions |
| 5 | UTF-8 Processing | International text tokenizer |
| 6 | Persistence | Crash-safe file storage |
| 7 | API & CLI | Complete command-line tool |
| 8 | Testing & Performance | Production-ready validation |

---

## Section Details

### Section 1: C Foundations
*Get comfortable with modern C and project organization*

- **01** - C99 Standard Features (designated initializers, fixed-width types)
- **02** - Memory Management (ownership, stack vs heap, create/destroy)
- **03** - Opaque Types & API Design (information hiding, clean interfaces)
- **04** - Build Systems (Makefiles, separate compilation)
- **Project** - Library skeleton with proper structure

### Section 2: Hash Tables
*Implement O(1) membership checking*

- **01** - Hash Function Design (FNV-1a, distribution, collisions)
- **02** - Collision Resolution (chaining, open addressing, Robin Hood)
- **03** - Dynamic Resizing (load factors, amortized analysis)
- **04** - String Interning (canonical storage, pointer comparison)
- **Project** - Production hash set with all features

### Section 3: Tries
*Build fast prefix completion*

- **01** - Basic Trie Structure (nodes, insertion, lookup)
- **02** - Trie Variants (array vs hash children, Patricia tries)
- **03** - Prefix Enumeration (collecting completions, limiting results)
- **04** - Memory Layout (cache efficiency, compact representations)
- **Project** - Trie with completion API

### Section 4: BK-Trees
*Enable spelling suggestions*

- **01** - Levenshtein Distance (edit distance algorithm, optimization)
- **02** - BK-Tree Structure (metric trees, triangle inequality pruning)
- **03** - Alternative Approaches (SymSpell, trigrams, trade-offs)
- **04** - Result Ranking (distance, frequency, presentation)
- **Project** - BK-tree with ranked suggestions

### Section 5: UTF-8 Processing
*Handle international text correctly*

- **01** - UTF-8 Encoding (byte sequences, character boundaries)
- **02** - Tokenization Strategies (word detection, configurable rules)
- **03** - Byte Offsets vs Positions (LSP compatibility, span reporting)
- **04** - Unicode Categories (letters, marks, practical handling)
- **Project** - UTF-8 aware tokenizer

### Section 6: Persistence
*Save and load data safely*

- **01** - Simple Text Formats (human-readable, versioned)
- **02** - File I/O in C (error handling, robustness)
- **03** - Atomic Writes (crash safety, temp-and-rename)
- **04** - Load/Save Architecture (single source of truth)
- **Project** - Crash-safe persistence layer

### Section 7: API & CLI
*Create the user-facing interface*

- **01** - API Design Principles (consistency, ownership, documentation)
- **02** - Error Handling Patterns (return codes, structured results)
- **03** - Command-Line Argument Parsing (subcommands, getopt)
- **04** - Stdin/Stdout Integration (pipes, scripting, output formats)
- **Project** - Complete CLI with all commands

### Section 8: Testing & Performance
*Ensure production quality*

- **01** - Unit Testing in C (assertions, test organization)
- **02** - Performance Measurement (timing, benchmarks, scaling)
- **03** - Memory Debugging (Valgrind, AddressSanitizer)
- **04** - Fuzzing (AFL, libFuzzer, crash discovery)
- **Project** - Full test suite and validation

---

## Time Estimate

**Total: 8-12 weeks** (adjustable to your pace)

| Phase | Sections | Time |
|-------|----------|------|
| Foundation | 1-2 | 2-3 weeks |
| Data Structures | 3-4 | 3-4 weeks |
| Text & Storage | 5-6 | 2 weeks |
| Integration | 7-8 | 2-3 weeks |

Each section has ~4 learning topics plus a hands-on project. Plan for 3-5 hours per topic if learning fresh, less if reviewing.

---

## Learning Approach

Each topic file follows this structure:

1. **Why You Need This** - Connects to your end goal
2. **What to Learn** - Core concepts with code examples
3. **Where to Learn** - Books, articles, source code to study
4. **Practice Exercise** - Hands-on task (30-60 minutes)
5. **Connection to Project** - How it fits the bigger picture

**Philosophy:**
- Learn by building, not just reading
- Understand *why*, not just *how*
- Each section produces working code
- Fun > exhaustive (you can always go deeper later)

---

## What You'll Have Built

```
wordlib/
├── include/
│   └── wordlib.h          # Clean public API
├── src/
│   ├── hashset.c          # O(1) word storage
│   ├── trie.c             # Prefix completion
│   ├── bktree.c           # Fuzzy suggestions
│   ├── tokenizer.c        # UTF-8 text processing
│   ├── persistence.c      # Safe file I/O
│   └── wordlib.c          # Core engine
├── cli/
│   └── main.c             # Command-line interface
├── test/
│   └── *.c                # Comprehensive tests
├── bench/
│   └── benchmark.c        # Performance validation
└── Makefile
```

**CLI Usage:**
```bash
wordlib add hello world cafe
wordlib check "The quikc brown fox"
wordlib complete hel
wordlib suggest helo
```

---

## Performance Targets

By completion, your engine will handle:

| Operation | Target | Typical Result |
|-----------|--------|----------------|
| Membership check | < 1 μs | ~0.05 μs |
| Completion (100 results) | < 100 μs | ~10 μs |
| Suggestion (dist=2) | < 50 ms | ~35 ms |
| Text check (sentence) | < 500 μs | ~200 μs |

With a 200,000 word dictionary loaded in under 500ms.

---

## Next Steps After This Course

This engine is designed for future expansion:

- **Course 2** (planned): LSP Server Implementation
  - JSON-RPC protocol handling
  - Diagnostic reporting
  - Completion provider
  - Code actions (add word to dictionary)

- **Course 3** (potential): Editor Integration
  - Neovim plugin development
  - VS Code extension
  - Real-time checking

---

## Getting Started

1. Read `section-01-c-foundations/01-c99-standard-features.md`
2. Work through each topic in order
3. Complete the section project before moving on
4. Use the practice exercises to solidify understanding

**Remember:** The goal is a working tool you'll actually use. Have fun with it!

---

## File Listing

```
course-01-word-library-engine/
├── README.md (this file)
├── section-01-c-foundations/
│   ├── 01-c99-standard-features.md
│   ├── 02-memory-management.md
│   ├── 03-opaque-types-api-design.md
│   ├── 04-build-systems.md
│   └── 05-section-project.md
├── section-02-hash-tables/
│   ├── 01-hash-function-design.md
│   ├── 02-collision-resolution.md
│   ├── 03-dynamic-resizing.md
│   ├── 04-string-interning.md
│   └── 05-section-project.md
├── section-03-tries/
│   ├── 01-basic-trie-structure.md
│   ├── 02-trie-variants.md
│   ├── 03-prefix-enumeration.md
│   ├── 04-memory-layout.md
│   └── 05-section-project.md
├── section-04-bk-trees/
│   ├── 01-levenshtein-distance.md
│   ├── 02-bk-tree-structure.md
│   ├── 03-alternative-approaches.md
│   ├── 04-result-ranking.md
│   └── 05-section-project.md
├── section-05-utf8-processing/
│   ├── 01-utf8-encoding.md
│   ├── 02-tokenization-strategies.md
│   ├── 03-byte-offsets-vs-positions.md
│   ├── 04-unicode-categories.md
│   └── 05-section-project.md
├── section-06-persistence/
│   ├── 01-simple-text-formats.md
│   ├── 02-file-io.md
│   ├── 03-atomic-writes.md
│   ├── 04-load-save-architecture.md
│   └── 05-section-project.md
├── section-07-api-cli/
│   ├── 01-api-design-principles.md
│   ├── 02-error-handling-patterns.md
│   ├── 03-cli-argument-parsing.md
│   ├── 04-stdin-stdout-integration.md
│   └── 05-section-project.md
└── section-08-testing-performance/
    ├── 01-unit-testing.md
    ├── 02-performance-measurement.md
    ├── 03-memory-debugging.md
    ├── 04-fuzzing.md
    └── 05-section-project.md
```

---

*Happy coding!* 🚀
