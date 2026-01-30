# Atomic Writes

## Why You Need This

Your requirements don't explicitly mention crash safety, but imagine: user adds 1000 words over months, power goes out mid-save, dictionary is corrupted. User rage quits. Don't let this happen.

## What to Learn

### The Problem with Direct Writes

```c
// DANGEROUS
FILE *f = fopen("dict.txt", "w");  // Truncates file!
for (int i = 0; i < count; i++) {
    fprintf(f, "%s\n", words[i]);
    // CRASH HERE = corrupted file
}
fclose(f);
```

If crash happens:
- After `fopen`: File is empty (truncated)
- During writes: Partial data
- Before `fclose`: Data may be buffered, not on disk

### The Solution: Write-Then-Rename

```c
int atomic_save(const char *path, const char **words, size_t count) {
    // 1. Write to temporary file
    char tmp_path[PATH_MAX];
    snprintf(tmp_path, sizeof(tmp_path), "%s.tmp", path);
    
    FILE *f = fopen(tmp_path, "w");
    if (!f) return -1;
    
    for (size_t i = 0; i < count; i++) {
        if (fprintf(f, "%s\n", words[i]) < 0) {
            fclose(f);
            unlink(tmp_path);
            return -1;
        }
    }
    
    // 2. Flush to disk
    if (fflush(f) != 0) {
        fclose(f);
        unlink(tmp_path);
        return -1;
    }
    
    // 3. fsync for durability (optional but recommended)
    if (fsync(fileno(f)) != 0) {
        fclose(f);
        unlink(tmp_path);
        return -1;
    }
    
    if (fclose(f) != 0) {
        unlink(tmp_path);
        return -1;
    }
    
    // 4. Atomic rename
    if (rename(tmp_path, path) != 0) {
        unlink(tmp_path);
        return -1;
    }
    
    return 0;
}
```

### Why This Works

`rename()` is atomic on POSIX systems:
- If it succeeds: new file is in place
- If it fails: old file is unchanged
- No in-between state

At any point:
- Before rename: Old file intact, temp file has new data
- After rename: New file is complete

Crash at any point = recoverable state.

### The fsync() Decision

```c
fsync(fileno(f));  // Force data to disk
```

Without `fsync`:
- Data may be in OS cache
- Crash = data loss even after "successful" write
- Faster

With `fsync`:
- Data guaranteed on disk
- Slower (can be 10-100x)
- Safer

**Recommendation:** Use `fsync` for important data. A dictionary save every few minutes can afford the overhead.

### Handling Existing Temp Files

```c
int atomic_save(const char *path, ...) {
    char tmp_path[PATH_MAX];
    snprintf(tmp_path, sizeof(tmp_path), "%s.tmp", path);
    
    // Remove stale temp file from previous crash
    unlink(tmp_path);  // Ignore error - might not exist
    
    FILE *f = fopen(tmp_path, "wx");  // "x" = fail if exists
    if (!f) {
        if (errno == EEXIST) {
            // Another process is writing? Wait and retry?
            return -1;
        }
        return -1;
    }
    // ...
}
```

### Backup on Save (Belt and Suspenders)

```c
int save_with_backup(const char *path, ...) {
    char backup_path[PATH_MAX];
    snprintf(backup_path, sizeof(backup_path), "%s.bak", path);
    
    // 1. Backup existing file
    if (file_exists(path)) {
        rename(path, backup_path);  // Old file is now .bak
    }
    
    // 2. Write new file (atomic)
    int result = atomic_save(path, ...);
    
    // 3. On success, optionally remove backup
    if (result == 0) {
        // unlink(backup_path);  // Or keep for safety
    } else {
        // Restore backup on failure
        rename(backup_path, path);
    }
    
    return result;
}
```

### Cross-Platform Considerations

**Linux/macOS:** `rename()` is atomic within same filesystem.

**Windows:** Use `MoveFileEx` with `MOVEFILE_REPLACE_EXISTING`:
```c
#ifdef _WIN32
if (!MoveFileExA(tmp_path, path, MOVEFILE_REPLACE_EXISTING)) {
    return -1;
}
#else
if (rename(tmp_path, path) != 0) {
    return -1;
}
#endif
```

**Network filesystems (NFS):** Atomicity not guaranteed. When possible, save to local disk.

### Testing Atomic Saves

```c
void test_atomic_save_crash_simulation(void) {
    // 1. Save valid data
    save_words("test.txt", words1, count1);
    
    // 2. Start save of new data
    write_temp_file("test.txt.tmp", words2, count2);
    
    // 3. Simulate crash (don't rename)
    // ...
    
    // 4. Recovery check: original file should be intact
    verify_file("test.txt", words1, count1);
    
    // 5. Cleanup
    unlink("test.txt.tmp");
}
```

## Where to Learn

1. **"Crash-only Software" paper** - Design philosophy
2. **SQLite documentation** - How they handle durability
3. **"POSIX Programmer's Manual"** - `rename()`, `fsync()` semantics
4. **LevelDB design docs** - Write-ahead logging (more advanced)

## Practice Exercise

Implement atomic file save:

```c
// Save data atomically. Returns 0 on success.
int atomic_write_file(const char *path, const char *data, size_t len);

// Load with recovery from .tmp file
int load_with_recovery(const char *path, char **data, size_t *len);
```

Test scenarios:
1. Normal save/load cycle
2. Simulate crash after temp write (leave .tmp file)
3. Verify load_with_recovery handles stale .tmp
4. Kill process mid-write (use a large file and kill -9)

## Connection to Project

Users trust your tool with their personal dictionary built over months or years. One corrupted save and you've lost that trust forever. Atomic writes cost a few milliseconds and buy you sleep at night.
