# File I/O in C

## Why You Need This

Your requirements say "persist known words to disk" and "tolerate missing or empty persistence files." That means robust file handling - not just the happy path.

## What to Learn

### Basic File Operations

```c
#include <stdio.h>

// Open file
FILE *f = fopen("words.txt", "r");   // Read
FILE *f = fopen("words.txt", "w");   // Write (truncates!)
FILE *f = fopen("words.txt", "a");   // Append
FILE *f = fopen("words.txt", "r+");  // Read and write

// Check for failure
if (!f) {
    perror("fopen");  // Prints error message
    return -1;
}

// Read line
char line[256];
while (fgets(line, sizeof(line), f)) {
    // Process line
}

// Write
fprintf(f, "%s\n", word);

// Always close
fclose(f);
```

### Error Handling

```c
#include <errno.h>

FILE *f = fopen(path, "r");
if (!f) {
    if (errno == ENOENT) {
        // File doesn't exist - might be OK
        return 0;
    }
    if (errno == EACCES) {
        // Permission denied
        return ERROR_PERMISSION;
    }
    // Other error
    return ERROR_IO;
}
```

Common errno values:
- `ENOENT` - File not found
- `EACCES` - Permission denied
- `ENOSPC` - No space left on device
- `EROFS` - Read-only file system
- `EISDIR` - Is a directory (not a file)

### Checking File Existence

```c
#include <sys/stat.h>

bool file_exists(const char *path) {
    struct stat st;
    return stat(path, &st) == 0;
}

bool is_regular_file(const char *path) {
    struct stat st;
    if (stat(path, &st) != 0) return false;
    return S_ISREG(st.st_mode);
}
```

### Reading Entire File

```c
char *read_file(const char *path, size_t *out_len) {
    FILE *f = fopen(path, "rb");
    if (!f) return NULL;
    
    // Get file size
    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    fseek(f, 0, SEEK_SET);
    
    if (size < 0) {
        fclose(f);
        return NULL;
    }
    
    char *buf = malloc(size + 1);
    if (!buf) {
        fclose(f);
        return NULL;
    }
    
    size_t read = fread(buf, 1, size, f);
    fclose(f);
    
    if (read != (size_t)size) {
        free(buf);
        return NULL;
    }
    
    buf[size] = '\0';
    if (out_len) *out_len = size;
    return buf;
}
```

### Writing with Error Checking

```c
int save_words(const char *path, const char **words, size_t count) {
    FILE *f = fopen(path, "w");
    if (!f) return -1;
    
    for (size_t i = 0; i < count; i++) {
        if (fprintf(f, "%s\n", words[i]) < 0) {
            fclose(f);
            return -1;  // Write error
        }
    }
    
    // Flush to ensure data is written
    if (fflush(f) != 0) {
        fclose(f);
        return -1;
    }
    
    if (fclose(f) != 0) {
        return -1;  // Close can fail!
    }
    
    return 0;
}
```

### The fclose() Trap

```c
// WRONG - ignores fclose errors
fclose(f);
return 0;

// RIGHT - check fclose return value
if (fclose(f) != 0) {
    return -1;  // Data might not be on disk!
}
return 0;
```

`fclose()` flushes buffers. On NFS or full disks, this can fail after all writes "succeeded."

### Path Handling

```c
#include <string.h>
#include <stdlib.h>

// Join paths safely
char *path_join(const char *dir, const char *file) {
    size_t dir_len = strlen(dir);
    size_t file_len = strlen(file);
    
    // Remove trailing slash from dir
    while (dir_len > 0 && dir[dir_len - 1] == '/') {
        dir_len--;
    }
    
    char *result = malloc(dir_len + 1 + file_len + 1);
    if (!result) return NULL;
    
    memcpy(result, dir, dir_len);
    result[dir_len] = '/';
    memcpy(result + dir_len + 1, file, file_len + 1);
    
    return result;
}

// Expand ~ to home directory
char *expand_home(const char *path) {
    if (path[0] != '~') return strdup(path);
    
    const char *home = getenv("HOME");
    if (!home) return strdup(path);
    
    size_t home_len = strlen(home);
    size_t path_len = strlen(path);
    
    char *result = malloc(home_len + path_len);  // -1 for ~, +1 for null
    if (!result) return NULL;
    
    memcpy(result, home, home_len);
    memcpy(result + home_len, path + 1, path_len);  // Skip ~
    
    return result;
}
```

### Directory Creation

```c
#include <sys/stat.h>
#include <errno.h>

int ensure_directory(const char *path) {
    struct stat st;
    
    if (stat(path, &st) == 0) {
        if (S_ISDIR(st.st_mode)) return 0;  // Already exists
        return -1;  // Exists but not a directory
    }
    
    if (errno != ENOENT) return -1;
    
    // Create with rwxr-xr-x permissions
    if (mkdir(path, 0755) != 0) {
        return -1;
    }
    
    return 0;
}
```

## Where to Learn

1. **"The C Programming Language"** - Chapter 7 (I/O)
2. **"Advanced Programming in the UNIX Environment"** - Comprehensive I/O
3. **Linux man pages** - `man 3 fopen`, `man 2 stat`
4. **POSIX standard** - Portable file operations

## Practice Exercise

Implement robust file utilities:

```c
// Read file, return NULL and set errno on error
char *read_file(const char *path, size_t *len);

// Write file atomically (write to temp, rename)
int write_file(const char *path, const char *data, size_t len);

// Check if path exists and is readable
bool is_readable(const char *path);
```

Test edge cases:
- Non-existent file
- Permission denied
- Empty file
- Very large file (>100MB)
- Directory instead of file

## Connection to Project

Your dictionary needs to:
- Load on startup (missing file = empty dictionary)
- Save on command (handle errors gracefully)
- Not lose data on crash (atomic writes)
- Work with user-specified paths

Solid file I/O prevents "where did my dictionary go?" disasters.
