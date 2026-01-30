# Section 7 Project: Complete API & CLI

## Objective

Finalize the WordLib API and implement a fully functional CLI that serves as the reference adapter for your engine.

## Requirements

1. **Complete API** - All engine features exposed cleanly
2. **Consistent conventions** - Naming, parameters, errors
3. **Functional CLI** - All commands working with proper I/O
4. **Scriptable** - Machine-readable output, meaningful exit codes
5. **Documented** - Help text, error messages

## Specification

### Final Public API (`include/wordlib.h`)

```c
#ifndef WORDLIB_H
#define WORDLIB_H

#include <stddef.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

//=============================================================================
// Types
//=============================================================================

typedef struct WordLib WordLib;

typedef struct {
    size_t start;  // Byte offset, inclusive
    size_t end;    // Byte offset, exclusive
} WLSpan;

typedef struct {
    WLSpan span;
    const char *word;
} WLUnknownWord;

typedef struct {
    const char *word;
    int distance;
} WLSuggestion;

typedef enum {
    WL_OK = 0,
    WL_ERR_NULL = -1,
    WL_ERR_INVALID = -2,
    WL_ERR_MEMORY = -10,
    WL_ERR_IO = -11,
    WL_ERR_PERMISSION = -12,
    WL_ERR_FORMAT = -30,
} WLError;

//=============================================================================
// Lifecycle
//=============================================================================

WordLib *wordlib_create(void);
void wordlib_destroy(WordLib *wl);

//=============================================================================
// Dictionary Operations
//=============================================================================

// Add word. Returns: 1=added, 0=existed, <0=error
int wordlib_add_word(WordLib *wl, const char *word);

// Remove word. Returns: 1=removed, 0=not found, <0=error
int wordlib_remove_word(WordLib *wl, const char *word);

// Check if word exists
bool wordlib_contains(const WordLib *wl, const char *word);

// Get word count
size_t wordlib_count(const WordLib *wl);

//=============================================================================
// Completion
//=============================================================================

// Get completions for prefix
// Returns number of results (0 to max_results)
// Results array filled with pointers valid until next modification
size_t wordlib_complete(const WordLib *wl, const char *prefix,
                        const char **results, size_t max_results);

//=============================================================================
// Suggestions
//=============================================================================

// Get spelling suggestions for word
// Returns number of results (0 to max_results)
// Results sorted by distance, then alphabetically
size_t wordlib_suggest(const WordLib *wl, const char *word,
                       int max_distance, WLSuggestion *results,
                       size_t max_results);

//=============================================================================
// Text Checking
//=============================================================================

// Check text for unknown words
// Returns number of unknown words found (0 to max_results)
// Results contain byte spans into original text
size_t wordlib_check_text(const WordLib *wl, const char *text,
                          WLUnknownWord *results, size_t max_results);

//=============================================================================
// Persistence
//=============================================================================

// Load dictionary from file. Missing file = empty dict, returns WL_OK
WLError wordlib_load(WordLib *wl, const char *path);

// Save dictionary to file (atomic write)
WLError wordlib_save(const WordLib *wl, const char *path);

// Check if dictionary has unsaved changes
bool wordlib_is_dirty(const WordLib *wl);

//=============================================================================
// Utilities
//=============================================================================

// Get error message for error code
const char *wordlib_strerror(WLError err);

// Get version string
const char *wordlib_version(void);

#ifdef __cplusplus
}
#endif

#endif // WORDLIB_H
```

### CLI Commands

```
wordlib <command> [options] [arguments]

Commands:
  add <word>...          Add words to dictionary
  remove <word>...       Remove words from dictionary  
  list                   List all words in dictionary
  check [text]           Check text for unknown words (stdin if no text)
  complete <prefix>      Show completions for prefix
  suggest <word>         Show spelling suggestions

Global Options:
  -d, --dict PATH        Dictionary file path
  -n, --max NUM          Maximum results (default: 20)
  -j, --json             Output in JSON format
  -q, --quiet            Suppress output, exit code only
  -h, --help             Show help
  -V, --version          Show version
```

### Exit Codes

```c
#define EXIT_OK 0              // Success
#define EXIT_ERROR 1           // General error
#define EXIT_USAGE 2           // Invalid usage
#define EXIT_UNKNOWN_FOUND 3   // check: found unknown words
```

## Test Cases

### API Tests (`test/test_api.c`)

```c
void test_lifecycle(void) {
    WordLib *wl = wordlib_create();
    assert(wl != NULL);
    assert(wordlib_count(wl) == 0);
    wordlib_destroy(wl);
}

void test_add_contains(void) {
    WordLib *wl = wordlib_create();
    
    assert(wordlib_add_word(wl, "hello") == 1);
    assert(wordlib_contains(wl, "hello"));
    assert(!wordlib_contains(wl, "world"));
    assert(wordlib_count(wl) == 1);
    
    // Duplicate
    assert(wordlib_add_word(wl, "hello") == 0);
    assert(wordlib_count(wl) == 1);
    
    wordlib_destroy(wl);
}

void test_remove(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "world");
    
    assert(wordlib_remove_word(wl, "hello") == 1);
    assert(!wordlib_contains(wl, "hello"));
    assert(wordlib_contains(wl, "world"));
    
    // Remove non-existent
    assert(wordlib_remove_word(wl, "hello") == 0);
    
    wordlib_destroy(wl);
}

void test_complete(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "help");
    wordlib_add_word(wl, "world");
    
    const char *results[10];
    size_t count = wordlib_complete(wl, "hel", results, 10);
    
    assert(count == 2);
    // Should be alphabetical
    assert(strcmp(results[0], "hello") == 0 || 
           strcmp(results[0], "help") == 0);
    
    wordlib_destroy(wl);
}

void test_suggest(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "help");
    wordlib_add_word(wl, "world");
    
    WLSuggestion results[10];
    size_t count = wordlib_suggest(wl, "helo", 2, results, 10);
    
    assert(count >= 1);
    assert(strcmp(results[0].word, "hello") == 0);
    assert(results[0].distance == 1);
    
    wordlib_destroy(wl);
}

void test_check_text(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "the");
    wordlib_add_word(wl, "quick");
    wordlib_add_word(wl, "fox");
    
    WLUnknownWord results[10];
    size_t count = wordlib_check_text(wl, "the quikc brown fox", 
                                       results, 10);
    
    assert(count == 2);  // "quikc" and "brown"
    
    // Find "quikc"
    bool found_quikc = false;
    for (size_t i = 0; i < count; i++) {
        if (strcmp(results[i].word, "quikc") == 0) {
            assert(results[i].span.start == 4);
            assert(results[i].span.end == 9);
            found_quikc = true;
        }
    }
    assert(found_quikc);
    
    wordlib_destroy(wl);
}

void test_null_safety(void) {
    // These should not crash
    assert(wordlib_add_word(NULL, "test") < 0);
    assert(wordlib_add_word(wordlib_create(), NULL) < 0);
    assert(!wordlib_contains(NULL, "test"));
    
    WordLib *wl = wordlib_create();
    wordlib_destroy(wl);
    // Don't double-destroy or use after destroy
}
```

### CLI Tests (shell script)

Create `test/test_cli.sh`:

```bash
#!/bin/bash
set -e

WORDLIB="./wordlib"
DICT="/tmp/test_dict_$$.txt"

cleanup() {
    rm -f "$DICT" "$DICT.tmp" "$DICT.bak"
}
trap cleanup EXIT

echo "Testing CLI..."

# Test add
$WORDLIB -d "$DICT" add hello world
test $($WORDLIB -d "$DICT" list | wc -l) -eq 2

# Test duplicate add
$WORDLIB -d "$DICT" add hello
test $($WORDLIB -d "$DICT" list | wc -l) -eq 2

# Test remove
$WORDLIB -d "$DICT" remove hello
test $($WORDLIB -d "$DICT" list | wc -l) -eq 1

# Test complete
$WORDLIB -d "$DICT" add hello help helicopter
result=$($WORDLIB -d "$DICT" complete hel)
echo "$result" | grep -q "hello"
echo "$result" | grep -q "help"

# Test suggest
$WORDLIB -d "$DICT" add hello
result=$($WORDLIB -d "$DICT" suggest helo)
echo "$result" | grep -q "hello"

# Test check with unknown words
$WORDLIB -d "$DICT" add the quick fox
if $WORDLIB -d "$DICT" check "the quikc brown fox"; then
    echo "FAIL: check should have returned non-zero"
    exit 1
fi

# Test check with all known words
if ! $WORDLIB -d "$DICT" check "the quick fox"; then
    echo "FAIL: check should have returned zero"
    exit 1
fi

# Test stdin
echo "the quikc fox" | $WORDLIB -d "$DICT" check && exit 1 || true

# Test JSON output
result=$($WORDLIB -d "$DICT" --json complete hel)
echo "$result" | grep -q '^\['

# Test quiet mode
$WORDLIB -d "$DICT" --quiet check "the quick fox"

# Test missing dict (should work, empty)
result=$($WORDLIB -d "/nonexistent/path/dict.txt" list 2>/dev/null || true)
test -z "$result"

echo "All CLI tests passed!"
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test
Running API tests... PASS
Running CLI tests... PASS
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_api
# All heap blocks were freed

$ valgrind --leak-check=full ./wordlib add test
# All heap blocks were freed
```

### CLI Usability
```bash
# Help is helpful
$ ./wordlib --help
# Shows clear usage

# Errors are clear
$ ./wordlib invalid_command
# "Unknown command: invalid_command"

# Works in pipes
$ echo "hello world" | ./wordlib check
$ ./wordlib complete hel | head -5
```

## Project Structure

```
wordlib/
├── include/
│   └── wordlib.h
├── src/
│   ├── wordlib.c      # Main API implementation
│   ├── hashset.c      # From section 2
│   ├── trie.c         # From section 3
│   ├── bktree.c       # From section 4
│   ├── tokenizer.c    # From section 5
│   └── persistence.c  # From section 6
├── cli/
│   ├── main.c         # CLI entry point
│   ├── commands.c     # Command handlers
│   └── output.c       # Output formatting
├── test/
│   ├── test_api.c
│   └── test_cli.sh
├── Makefile
└── README.md
```

## Stretch Goals

1. **Config file** - ~/.wordlibrc for default options
2. **Colored output** - Highlight unknown words in red
3. **Progress indicator** - For large file checks
4. **Batch mode** - `wordlib check file1.txt file2.txt`
5. **Watch mode** - `wordlib watch file.txt` (re-check on change)

## What You'll Learn

By completing this project:
- Complete API design
- CLI implementation patterns
- Integration of all previous components
- End-to-end testing

## Next Section Preview

Section 8 is the final polish: comprehensive testing, performance benchmarking, and memory debugging. You'll ensure your tool is production-ready.
