# Section 6 Project: Persistence Layer

## Objective

Implement save/load functionality for your word dictionary with atomic writes and crash safety.

## Requirements

1. **Human-readable format** - One word per line with optional header
2. **Atomic writes** - No data loss on crash
3. **Tolerant loading** - Missing files, empty files handled gracefully
4. **Single source of truth** - All structures populated from loaded data
5. **Memory safe** - Valgrind clean

## Specification

### Public API (`include/wordlib.h` additions)

```c
// File operations
int wordlib_load(WordLib *wl, const char *path);
int wordlib_save(const WordLib *wl, const char *path);

// State tracking
bool wordlib_is_dirty(const WordLib *wl);
void wordlib_mark_clean(WordLib *wl);

// Convenience
WordLib *wordlib_create_from_file(const char *path);
int wordlib_save_if_dirty(WordLib *wl, const char *path);
```

### Return Codes

```c
typedef enum {
    WORDLIB_OK = 0,
    WORDLIB_ERR_IO = -1,
    WORDLIB_ERR_MEMORY = -2,
    WORDLIB_ERR_FORMAT = -3,
    WORDLIB_ERR_PERMISSION = -4,
} WordLibError;
```

### File Format (v1)

```
# wordlib dictionary v1
hello
world
café
...
```

- First line: optional header comment
- Subsequent lines: one word per line
- UTF-8 encoded
- Unix line endings (\n)

### Behavior Specifications

**wordlib_load:**
- Returns `WORDLIB_OK` (0) on success
- Returns `WORDLIB_OK` if file doesn't exist (empty dictionary is valid)
- Returns `WORDLIB_ERR_IO` on read error
- Returns `WORDLIB_ERR_FORMAT` on invalid format (future-proofing)
- Skips empty lines and lines starting with #
- Populates hash set, trie, and BK-tree

**wordlib_save:**
- Returns `WORDLIB_OK` on success
- Uses atomic write (temp file + rename)
- Creates parent directory if needed (optional)
- Writes header line
- Returns `WORDLIB_ERR_IO` on write error

**wordlib_is_dirty:**
- Returns true if words added/removed since last save/load
- Returns false for freshly created or just-saved library

## Test Cases

Create `test/test_persistence.c`:

```c
void test_save_load_cycle(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "hello");
    wordlib_add_word(wl, "world");
    wordlib_add_word(wl, "café");
    
    assert(wordlib_save(wl, "test_dict.txt") == WORDLIB_OK);
    wordlib_destroy(wl);
    
    // Load into new instance
    wl = wordlib_create();
    assert(wordlib_load(wl, "test_dict.txt") == WORDLIB_OK);
    
    assert(wordlib_count(wl) == 3);
    assert(wordlib_contains(wl, "hello"));
    assert(wordlib_contains(wl, "world"));
    assert(wordlib_contains(wl, "café"));
    
    wordlib_destroy(wl);
    unlink("test_dict.txt");
}

void test_missing_file(void) {
    WordLib *wl = wordlib_create();
    
    // Should succeed with empty dictionary
    assert(wordlib_load(wl, "nonexistent.txt") == WORDLIB_OK);
    assert(wordlib_count(wl) == 0);
    
    wordlib_destroy(wl);
}

void test_empty_file(void) {
    // Create empty file
    FILE *f = fopen("empty.txt", "w");
    fclose(f);
    
    WordLib *wl = wordlib_create();
    assert(wordlib_load(wl, "empty.txt") == WORDLIB_OK);
    assert(wordlib_count(wl) == 0);
    
    wordlib_destroy(wl);
    unlink("empty.txt");
}

void test_file_with_header(void) {
    // Create file with header
    FILE *f = fopen("headed.txt", "w");
    fprintf(f, "# wordlib dictionary v1\n");
    fprintf(f, "apple\n");
    fprintf(f, "# comment line\n");
    fprintf(f, "banana\n");
    fprintf(f, "\n");  // empty line
    fprintf(f, "cherry\n");
    fclose(f);
    
    WordLib *wl = wordlib_create();
    assert(wordlib_load(wl, "headed.txt") == WORDLIB_OK);
    assert(wordlib_count(wl) == 3);
    assert(wordlib_contains(wl, "apple"));
    assert(wordlib_contains(wl, "banana"));
    assert(wordlib_contains(wl, "cherry"));
    
    wordlib_destroy(wl);
    unlink("headed.txt");
}

void test_dirty_flag(void) {
    WordLib *wl = wordlib_create();
    
    assert(!wordlib_is_dirty(wl));  // Fresh
    
    wordlib_add_word(wl, "test");
    assert(wordlib_is_dirty(wl));
    
    wordlib_save(wl, "dirty_test.txt");
    assert(!wordlib_is_dirty(wl));  // Clean after save
    
    wordlib_add_word(wl, "another");
    assert(wordlib_is_dirty(wl));
    
    wordlib_destroy(wl);
    unlink("dirty_test.txt");
}

void test_atomic_write(void) {
    WordLib *wl = wordlib_create();
    for (int i = 0; i < 1000; i++) {
        char buf[32];
        snprintf(buf, sizeof(buf), "word%d", i);
        wordlib_add_word(wl, buf);
    }
    
    wordlib_save(wl, "atomic_test.txt");
    
    // Verify no .tmp file remains
    assert(!file_exists("atomic_test.txt.tmp"));
    
    // Verify file is complete
    WordLib *wl2 = wordlib_create();
    wordlib_load(wl2, "atomic_test.txt");
    assert(wordlib_count(wl2) == 1000);
    
    wordlib_destroy(wl);
    wordlib_destroy(wl2);
    unlink("atomic_test.txt");
}

void test_utf8_preservation(void) {
    WordLib *wl = wordlib_create();
    
    wordlib_add_word(wl, "naïve");
    wordlib_add_word(wl, "北京");
    wordlib_add_word(wl, "日本語");
    
    wordlib_save(wl, "utf8_test.txt");
    wordlib_destroy(wl);
    
    // Load and verify
    wl = wordlib_create();
    wordlib_load(wl, "utf8_test.txt");
    
    assert(wordlib_contains(wl, "naïve"));
    assert(wordlib_contains(wl, "北京"));
    assert(wordlib_contains(wl, "日本語"));
    
    wordlib_destroy(wl);
    unlink("utf8_test.txt");
}

void test_large_dictionary(void) {
    WordLib *wl = wordlib_create();
    
    // Load system dictionary if available
    FILE *f = fopen("/usr/share/dict/words", "r");
    if (!f) {
        printf("Skipping large dictionary test\n");
        wordlib_destroy(wl);
        return;
    }
    
    char line[256];
    while (fgets(line, sizeof(line), f)) {
        line[strcspn(line, "\n")] = '\0';
        wordlib_add_word(wl, line);
    }
    fclose(f);
    
    size_t count = wordlib_count(wl);
    printf("Loaded %zu words\n", count);
    
    // Time the save
    clock_t start = clock();
    assert(wordlib_save(wl, "large_test.txt") == WORDLIB_OK);
    clock_t save_time = clock() - start;
    
    // Time the load
    WordLib *wl2 = wordlib_create();
    start = clock();
    assert(wordlib_load(wl2, "large_test.txt") == WORDLIB_OK);
    clock_t load_time = clock() - start;
    
    printf("Save: %.2f ms, Load: %.2f ms\n",
           (double)save_time / CLOCKS_PER_SEC * 1000,
           (double)load_time / CLOCKS_PER_SEC * 1000);
    
    assert(wordlib_count(wl2) == count);
    
    wordlib_destroy(wl);
    wordlib_destroy(wl2);
    unlink("large_test.txt");
}

void test_concurrent_access_simulation(void) {
    // Simulate what happens if we try to save while another
    // process might be reading
    
    WordLib *wl = wordlib_create();
    wordlib_add_word(wl, "test");
    
    // First save
    wordlib_save(wl, "concurrent.txt");
    
    // Simulate reader has file open (we can still write)
    FILE *reader = fopen("concurrent.txt", "r");
    
    // Second save (should work - writes to .tmp then renames)
    wordlib_add_word(wl, "another");
    assert(wordlib_save(wl, "concurrent.txt") == WORDLIB_OK);
    
    // Reader still works (might see old or new data, but not corrupted)
    char line[256];
    while (fgets(line, sizeof(line), reader)) {
        // Not empty/corrupted
        assert(strlen(line) > 0);
    }
    fclose(reader);
    
    wordlib_destroy(wl);
    unlink("concurrent.txt");
}
```

## Acceptance Criteria

### All Tests Pass
```bash
$ make test_persistence
Running persistence tests...
test_save_load_cycle: PASS
test_missing_file: PASS
test_empty_file: PASS
test_file_with_header: PASS
test_dirty_flag: PASS
test_atomic_write: PASS
test_utf8_preservation: PASS
test_large_dictionary: Loaded 235886 words
  Save: 45.23 ms, Load: 123.45 ms
  PASS
test_concurrent_access_simulation: PASS
All tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_persistence
...
All heap blocks were freed -- no leaks are possible
```

### Performance
- Save 200k words: < 200ms
- Load 200k words: < 500ms

## Implementation Hints

### Atomic Save Pattern
```c
int wordlib_save(const WordLib *wl, const char *path) {
    char tmp_path[PATH_MAX];
    snprintf(tmp_path, sizeof(tmp_path), "%s.tmp.%d", path, getpid());
    
    FILE *f = fopen(tmp_path, "w");
    if (!f) return WORDLIB_ERR_IO;
    
    // Write header
    fprintf(f, "# wordlib dictionary v1\n");
    
    // Write words
    // ... (iterate hash set or use foreach)
    
    if (fflush(f) != 0 || fsync(fileno(f)) != 0) {
        fclose(f);
        unlink(tmp_path);
        return WORDLIB_ERR_IO;
    }
    
    if (fclose(f) != 0) {
        unlink(tmp_path);
        return WORDLIB_ERR_IO;
    }
    
    if (rename(tmp_path, path) != 0) {
        unlink(tmp_path);
        return WORDLIB_ERR_IO;
    }
    
    return WORDLIB_OK;
}
```

### Load with Line Processing
```c
int wordlib_load(WordLib *wl, const char *path) {
    FILE *f = fopen(path, "r");
    if (!f) {
        return (errno == ENOENT) ? WORDLIB_OK : WORDLIB_ERR_IO;
    }
    
    char line[4096];  // Long enough for any reasonable word
    while (fgets(line, sizeof(line), f)) {
        // Strip newline
        line[strcspn(line, "\r\n")] = '\0';
        
        // Skip empty and comment lines
        if (line[0] == '\0' || line[0] == '#') continue;
        
        // Add word
        wordlib_add_word(wl, line);
    }
    
    fclose(f);
    wl->dirty = false;  // Just loaded, not dirty
    return WORDLIB_OK;
}
```

## Stretch Goals

1. **Version migration** - Handle loading v0 (no header) and v1 formats
2. **Backup rotation** - Keep last N backups
3. **Import/export** - Support other formats (JSON, CSV)
4. **Compression** - Optional gzip compression for large dictionaries

## What You'll Learn

By completing this project:
- Robust file I/O
- Atomic write patterns
- Error handling and recovery
- Integration testing

## Next Section Preview

Section 7 brings it all together with API design and CLI implementation. You'll create a clean interface for all the engine functionality and a command-line tool to use it.
