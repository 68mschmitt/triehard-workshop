# Load/Save Architecture

## Why You Need This

Your requirements say: "Internal data structures (hash set, trie, BK-tree) must derive from a single canonical word store." This means one source of truth that populates all structures. The load/save architecture makes this clean.

## What to Learn

### The Canonical Store Pattern

```
                    ┌─────────────────┐
                    │   words.txt     │
                    │   (on disk)     │
                    └────────┬────────┘
                             │
                           Load
                             │
                             ▼
                    ┌─────────────────┐
                    │  String Pool    │
                    │  (canonical)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Hash Set │  │   Trie   │  │ BK-Tree  │
        │ (lookup) │  │(complete)│  │(suggest) │
        └──────────┘  └──────────┘  └──────────┘
```

All structures share interned pointers from the string pool.

### Load Flow

```c
int wordlib_load(WordLib *wl, const char *path) {
    // 1. Read file line by line
    FILE *f = fopen(path, "r");
    if (!f) {
        if (errno == ENOENT) return 0;  // Empty is OK
        return -1;
    }
    
    char line[1024];
    while (fgets(line, sizeof(line), f)) {
        // 2. Clean up line
        line[strcspn(line, "\r\n")] = '\0';
        if (line[0] == '\0') continue;
        
        // 3. Intern string (canonical copy)
        const char *word = stringpool_intern(wl->pool, line);
        
        // 4. Add to all structures
        hashset_add(wl->words, word);
        trie_insert(wl->trie, word);
        bktree_insert(wl->bktree, word);
    }
    
    fclose(f);
    return 0;
}
```

### Save Flow

```c
typedef struct {
    FILE *file;
    int error;
} SaveContext;

static void save_word(const char *word, void *ctx) {
    SaveContext *sc = ctx;
    if (sc->error) return;
    
    if (fprintf(sc->file, "%s\n", word) < 0) {
        sc->error = 1;
    }
}

int wordlib_save(WordLib *wl, const char *path) {
    char tmp_path[PATH_MAX];
    snprintf(tmp_path, sizeof(tmp_path), "%s.tmp", path);
    
    FILE *f = fopen(tmp_path, "w");
    if (!f) return -1;
    
    // Write header
    fprintf(f, "# wordlib dictionary v1\n");
    
    // Iterate canonical store (hash set has all words)
    SaveContext ctx = { .file = f, .error = 0 };
    hashset_foreach(wl->words, save_word, &ctx);
    
    if (ctx.error) {
        fclose(f);
        unlink(tmp_path);
        return -1;
    }
    
    // Ensure written
    fflush(f);
    fsync(fileno(f));
    fclose(f);
    
    // Atomic rename
    return rename(tmp_path, path);
}
```

### Adding Words at Runtime

```c
int wordlib_add_word(WordLib *wl, const char *word) {
    // 1. Intern (get canonical pointer)
    const char *interned = stringpool_intern(wl->pool, word);
    
    // 2. Check if already known
    if (hashset_contains(wl->words, interned)) {
        return 0;  // Already exists
    }
    
    // 3. Add to all structures (same pointer!)
    hashset_add(wl->words, interned);
    trie_insert(wl->trie, interned);
    bktree_insert(wl->bktree, interned);
    
    // 4. Mark dirty (needs save)
    wl->dirty = true;
    
    return 1;  // Added
}
```

### Removing Words at Runtime

```c
int wordlib_remove_word(WordLib *wl, const char *word) {
    // 1. Find canonical pointer
    const char *interned = hashset_get(wl->words, word);
    if (!interned) return 0;  // Not found
    
    // 2. Remove from structures
    hashset_remove(wl->words, interned);
    trie_remove(wl->trie, interned);
    // BK-tree: mark as deleted (or rebuild later)
    bktree_mark_deleted(wl->bktree, interned);
    
    // 3. Note: Don't remove from string pool
    //    (Other structures might still reference it briefly)
    //    Pool cleanup happens on destroy or explicit compact
    
    wl->dirty = true;
    return 1;
}
```

### Lazy vs Eager Loading

**Eager loading (simpler, recommended):**
```c
WordLib *wordlib_create_from_file(const char *path) {
    WordLib *wl = wordlib_create();
    wordlib_load(wl, path);  // Load everything now
    return wl;
}
```

**Lazy loading (complex, premature optimization):**
```c
// Don't do this unless you have a proven performance problem
WordLib *wordlib_create_lazy(const char *path) {
    WordLib *wl = wordlib_create();
    wl->pending_path = strdup(path);
    wl->loaded = false;
    return wl;
}

void ensure_loaded(WordLib *wl) {
    if (!wl->loaded) {
        wordlib_load(wl, wl->pending_path);
        wl->loaded = true;
    }
}
```

For 50k words, eager loading takes < 100ms. Not worth the complexity of lazy loading.

### Auto-Save Pattern

```c
int wordlib_maybe_save(WordLib *wl, const char *path) {
    if (!wl->dirty) return 0;  // Nothing to save
    
    int result = wordlib_save(wl, path);
    if (result == 0) {
        wl->dirty = false;
    }
    return result;
}
```

CLI adapter can call this before exit:
```c
// cli/main.c
int main(int argc, char **argv) {
    WordLib *wl = wordlib_create();
    wordlib_load(wl, dict_path);
    
    // ... process commands ...
    
    wordlib_maybe_save(wl, dict_path);  // Save if changed
    wordlib_destroy(wl);
    return 0;
}
```

### Error Recovery

```c
int wordlib_load_safe(WordLib *wl, const char *path) {
    // Check for stale temp file (crash recovery)
    char tmp_path[PATH_MAX];
    snprintf(tmp_path, sizeof(tmp_path), "%s.tmp", path);
    
    if (file_exists(tmp_path)) {
        // Previous save was interrupted
        // Option 1: Delete temp (was incomplete)
        unlink(tmp_path);
        // Option 2: Could try to recover, but risky
    }
    
    return wordlib_load(wl, path);
}
```

## Where to Learn

1. **Repository pattern** - Software architecture concept
2. **"Clean Architecture"** - Separation of concerns
3. **SQLite internals** - How they manage multiple indices
4. **ORM design patterns** - Similar single-source-of-truth concepts

## Connection to Project

This architecture ensures:
- One file = one source of truth
- All lookups consistent (same data)
- Add/remove operations stay in sync
- Save captures complete state

When the LSP adapter calls `wordlib_add_word`, it Just Works - hash set, trie, and BK-tree all get the word.
