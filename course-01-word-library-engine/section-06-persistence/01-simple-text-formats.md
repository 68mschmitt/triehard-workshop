# Simple Text Formats

## Why You Need This

Your requirements say: "Known words shall be persisted in a simple, human-readable format." You need to save the dictionary to disk and load it back. The format matters more than you might think.

## What to Learn

### Why Human-Readable?

1. **Debuggability:** Open in any text editor to inspect
2. **Portability:** No binary compatibility issues
3. **Git-friendly:** Can diff, merge, version control
4. **Recovery:** Manual fixes when things go wrong
5. **Trust:** Users can see exactly what's stored

### The Simplest Format: One Word Per Line

```
hello
world
café
```

That's it. One UTF-8 word per line.

**Pros:**
- Trivially simple to read and write
- Universal tool support (cat, grep, sort, wc)
- Editable by users

**Cons:**
- No metadata (frequency, source, timestamp)
- Newline handling edge cases
- Large file for many words

### Reading the Format

```c
int load_words(const char *path, WordCallback cb, void *ctx) {
    FILE *f = fopen(path, "r");
    if (!f) {
        if (errno == ENOENT) return 0;  // Missing file is OK
        return -1;
    }
    
    char line[1024];
    int count = 0;
    
    while (fgets(line, sizeof(line), f)) {
        // Remove trailing newline
        size_t len = strlen(line);
        while (len > 0 && (line[len-1] == '\n' || line[len-1] == '\r')) {
            line[--len] = '\0';
        }
        
        // Skip empty lines
        if (len == 0) continue;
        
        cb(line, ctx);
        count++;
    }
    
    fclose(f);
    return count;
}
```

### Writing the Format

```c
int save_words(const char *path, const char **words, size_t count) {
    FILE *f = fopen(path, "w");
    if (!f) return -1;
    
    for (size_t i = 0; i < count; i++) {
        fprintf(f, "%s\n", words[i]);
    }
    
    fclose(f);
    return 0;
}
```

### Handling Edge Cases

**Empty lines:**
```c
// Skip on load
if (len == 0) continue;
```

**Whitespace:**
```c
// Trim leading/trailing whitespace (optional)
char *trimmed = trim(line);
if (*trimmed == '\0') continue;
```

**BOM (Byte Order Mark):**
```c
// Skip UTF-8 BOM at start of file
if (count == 0 && len >= 3 && 
    (unsigned char)line[0] == 0xEF &&
    (unsigned char)line[1] == 0xBB &&
    (unsigned char)line[2] == 0xBF) {
    memmove(line, line + 3, len - 3 + 1);
}
```

**Very long lines:**
```c
// Reject lines that don't fit
if (strlen(line) == sizeof(line) - 1 && line[sizeof(line)-2] != '\n') {
    // Line too long, skip rest
    int c;
    while ((c = fgetc(f)) != '\n' && c != EOF);
    continue;
}
```

### Format Versioning

Add a header comment for future-proofing:

```
# wordlib dictionary v1
hello
world
café
```

```c
// Check header on load
if (fgets(line, sizeof(line), f)) {
    if (strncmp(line, "# wordlib dictionary v", 22) == 0) {
        int version = atoi(line + 22);
        if (version > SUPPORTED_VERSION) {
            fclose(f);
            return ERROR_VERSION;
        }
    } else {
        // No header - treat as v0 (raw word list)
        rewind(f);
    }
}
```

### Extended Format (Optional)

If you need metadata later:

```
# wordlib dictionary v2
# word<TAB>frequency<TAB>added_date
hello	100	2024-01-15
world	95	2024-01-15
café	12	2024-01-20
```

Tab-separated, still human-readable.

```c
// Parsing v2 format
char *word = strtok(line, "\t");
char *freq_str = strtok(NULL, "\t");
char *date_str = strtok(NULL, "\t");
int frequency = freq_str ? atoi(freq_str) : 0;
```

### What About Binary Formats?

**MessagePack, Protocol Buffers, etc.:**
- Faster to parse
- Smaller files
- NOT human-readable
- Versioning complexity

For a personal dictionary of 10k-50k words, text is plenty fast. Stick with simple.

## Where to Learn

1. **Unix Philosophy** - "Text streams are a universal interface"
2. **"The Art of Unix Programming"** - Chapter on data formats
3. **INI/TOML/YAML specs** - Inspiration for simple formats
4. **SQLite file format docs** - When you DO need binary (overkill here)

## Practice Exercise

Implement basic persistence:

```c
int wordlib_save(WordLib *wl, const char *path);
int wordlib_load(WordLib *wl, const char *path);
```

Test:
1. Create WordLib, add words
2. Save to file
3. Create new WordLib
4. Load from file
5. Verify all words present
6. Inspect file in text editor

## Connection to Project

Your dictionary needs to persist across sessions. When the user adds "kubernetes" to their dictionary, it should still be there tomorrow. Simple text format makes this reliable and debuggable.
