# Stdin/Stdout Integration

## Why You Need This

Your requirements say: "Support reading text from stdin" and "Be suitable for scripting and editor integration." Unix tools compose via pipes. Your CLI should play nice with this ecosystem.

## What to Learn

### The Unix Philosophy

```bash
# Composition via pipes
cat document.txt | wordlib check | grep "error"
wordlib complete "com" | head -5
echo "hello world" | wordlib check --json | jq '.unknown'
```

Your tool should:
- Read from stdin when no file argument
- Write to stdout in parseable format
- Send errors to stderr
- Return meaningful exit codes

### Stdin Reading Patterns

**Line by line:**
```c
char line[4096];
while (fgets(line, sizeof(line), stdin)) {
    process_line(line);
}
```

**Entire input:**
```c
char *read_all_stdin(size_t *out_size) {
    size_t capacity = 4096;
    size_t size = 0;
    char *buffer = malloc(capacity);
    
    size_t n;
    while ((n = fread(buffer + size, 1, capacity - size - 1, stdin)) > 0) {
        size += n;
        if (size + 1 >= capacity) {
            capacity *= 2;
            buffer = realloc(buffer, capacity);
        }
    }
    
    buffer[size] = '\0';
    *out_size = size;
    return buffer;
}
```

**Detect if stdin has data:**
```c
#include <unistd.h>
#include <sys/select.h>

bool stdin_has_data(void) {
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(STDIN_FILENO, &fds);
    
    struct timeval tv = {0, 0};  // Don't wait
    return select(1, &fds, NULL, NULL, &tv) > 0;
}
```

### Stdout Output Patterns

**Human-readable (default):**
```c
void output_results_human(const char **words, size_t count) {
    for (size_t i = 0; i < count; i++) {
        printf("%s\n", words[i]);
    }
}
```

**JSON (machine-readable):**
```c
void output_results_json(const char **words, size_t count) {
    printf("[");
    for (size_t i = 0; i < count; i++) {
        if (i > 0) printf(",");
        printf("\"%s\"", words[i]);  // TODO: proper escaping
    }
    printf("]\n");
}
```

**Tab-separated (easy parsing):**
```c
void output_unknown_words(Token *tokens, size_t count) {
    for (size_t i = 0; i < count; i++) {
        printf("%zu\t%zu\t%.*s\n", 
               tokens[i].span.start,
               tokens[i].span.end,
               (int)tokens[i].length,
               tokens[i].text);
    }
}
```

### Stderr for Errors

```c
// Good: errors go to stderr, data to stdout
if (error) {
    fprintf(stderr, "Error: %s\n", wl_strerror(err));
}
printf("%s\n", result);  // Data to stdout

// Bad: mixing errors and data
printf("Error: something went wrong\n");  // Corrupts output!
printf("%s\n", result);
```

### Interactive vs Pipe Detection

```c
#include <unistd.h>

bool is_interactive(void) {
    return isatty(STDIN_FILENO);
}

int cmd_check(int argc, char **argv) {
    const char *text;
    char *allocated = NULL;
    
    if (argc > 0) {
        // Text provided as argument
        text = argv[0];
    } else if (!is_interactive()) {
        // Reading from pipe
        allocated = read_all_stdin(NULL);
        text = allocated;
    } else {
        // Interactive mode with no input
        fprintf(stderr, "Usage: wordlib check <text>\n");
        fprintf(stderr, "   or: echo <text> | wordlib check\n");
        return 1;
    }
    
    // Process text...
    
    free(allocated);
    return 0;
}
```

### Output Modes

```c
typedef enum {
    OUTPUT_HUMAN,   // Pretty-printed for humans
    OUTPUT_JSON,    // JSON for scripts
    OUTPUT_TSV,     // Tab-separated for awk/cut
    OUTPUT_QUIET,   // Only exit code matters
} OutputMode;

OutputMode g_output_mode = OUTPUT_HUMAN;

// Set via flags
// --json, --tsv, --quiet

void output_completion(const char *word) {
    switch (g_output_mode) {
        case OUTPUT_HUMAN:
            printf("  %s\n", word);
            break;
        case OUTPUT_JSON:
            printf("\"%s\"", word);  // Part of array
            break;
        case OUTPUT_TSV:
            printf("%s\n", word);  // Just the word
            break;
        case OUTPUT_QUIET:
            // No output
            break;
    }
}
```

### Exit Codes for Scripting

```c
// Exit codes
#define EXIT_SUCCESS 0      // Everything OK
#define EXIT_ERROR 1        // General error
#define EXIT_USAGE 2        // Bad arguments
#define EXIT_FOUND 0        // For check: no unknown words
#define EXIT_NOT_FOUND 3    // For check: has unknown words

int cmd_check(...) {
    // ... process text ...
    
    if (unknown_count > 0) {
        // Report unknown words
        return EXIT_NOT_FOUND;  // Non-zero for scripting
    }
    return EXIT_FOUND;
}
```

Usage in scripts:
```bash
if wordlib check document.txt; then
    echo "All words known"
else
    echo "Found unknown words"
fi
```

### Neovim Integration Example

```vim
" In Neovim, check current buffer
function! CheckSpelling()
    let text = join(getline(1, '$'), "\n")
    let result = system('wordlib check --json', text)
    let unknown = json_decode(result)
    " Highlight unknown words...
endfunction

" Get completions
function! GetCompletions(prefix)
    let result = system('wordlib complete ' . a:prefix)
    return split(result, "\n")
endfunction
```

### Performance for Pipes

```c
// Buffer output for large results
void cmd_complete_many(...) {
    // Without buffering: each printf() = syscall = slow
    // With buffering: batch writes
    
    char buffer[65536];
    setvbuf(stdout, buffer, _IOFBF, sizeof(buffer));
    
    for (size_t i = 0; i < count; i++) {
        printf("%s\n", words[i]);
    }
    
    fflush(stdout);
}
```

## Where to Learn

1. **"The Unix Programming Environment"** - Classic on pipes and filters
2. **"The Art of Unix Programming"** - Textuality principle
3. **POSIX documentation** - stdin/stdout/stderr conventions
4. **jq source code** - Excellent JSON CLI tool

## Practice Exercise

Implement stdin/stdout integration:

```bash
# These should all work:
wordlib check "inline text"
echo "piped text" | wordlib check
cat document.txt | wordlib check
wordlib check < document.txt

# Output modes:
wordlib check --json "text"
wordlib check --tsv "text"
wordlib check --quiet "text"  # Only exit code
```

Test in real pipes:
```bash
cat large_file.txt | wordlib check | wc -l
wordlib complete "com" | grep "plete"
```

## Connection to Project

Good stdin/stdout handling means:
- Neovim can shell out and parse results
- Users can build pipelines
- Your tool composes with others
- Testing is easy (feed input, check output)
