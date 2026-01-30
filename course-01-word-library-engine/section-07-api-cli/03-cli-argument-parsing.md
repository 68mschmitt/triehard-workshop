# Command-Line Argument Parsing

## Why You Need This

Your requirements say: "Expose engine functionality via commands (add, remove, check, complete, suggest)" and "Be suitable for scripting and editor integration." A good CLI is the first adapter and test harness.

## What to Learn

### CLI Structure Options

**Subcommand style (like git):**
```bash
wordlib add hello world
wordlib check "some text"
wordlib complete hel
wordlib suggest helo
```

**Flag style:**
```bash
wordlib --add hello
wordlib --check "some text"
wordlib --complete hel
```

**Subcommand style is better for your use case** - clearer, more extensible.

### Basic Argument Parsing

```c
int main(int argc, char **argv) {
    if (argc < 2) {
        print_usage();
        return 1;
    }
    
    const char *command = argv[1];
    
    if (strcmp(command, "add") == 0) {
        return cmd_add(argc - 2, argv + 2);
    } else if (strcmp(command, "check") == 0) {
        return cmd_check(argc - 2, argv + 2);
    } else if (strcmp(command, "complete") == 0) {
        return cmd_complete(argc - 2, argv + 2);
    } else if (strcmp(command, "suggest") == 0) {
        return cmd_suggest(argc - 2, argv + 2);
    } else {
        fprintf(stderr, "Unknown command: %s\n", command);
        return 1;
    }
}
```

### Using getopt

```c
#include <getopt.h>

int main(int argc, char **argv) {
    char *dict_path = NULL;
    int max_results = 10;
    int opt;
    
    while ((opt = getopt(argc, argv, "d:n:h")) != -1) {
        switch (opt) {
            case 'd':
                dict_path = optarg;
                break;
            case 'n':
                max_results = atoi(optarg);
                break;
            case 'h':
                print_usage();
                return 0;
            default:
                return 1;
        }
    }
    
    // Remaining arguments: argv[optind] ... argv[argc-1]
    if (optind >= argc) {
        fprintf(stderr, "Expected command\n");
        return 1;
    }
    
    const char *command = argv[optind];
    // ...
}
```

### Long Options

```c
static struct option long_options[] = {
    {"dict", required_argument, 0, 'd'},
    {"max", required_argument, 0, 'n'},
    {"help", no_argument, 0, 'h'},
    {"version", no_argument, 0, 'V'},
    {0, 0, 0, 0}
};

while ((opt = getopt_long(argc, argv, "d:n:hV", long_options, NULL)) != -1) {
    // ...
}
```

Now supports both `-d` and `--dict`.

### Command Handlers

```c
int cmd_add(int argc, char **argv) {
    if (argc == 0) {
        fprintf(stderr, "Usage: wordlib add <word> [word...]\n");
        return 1;
    }
    
    WordLib *wl = load_dict();
    
    for (int i = 0; i < argc; i++) {
        if (wordlib_add_word(wl, argv[i])) {
            printf("Added: %s\n", argv[i]);
        } else {
            printf("Already exists: %s\n", argv[i]);
        }
    }
    
    save_dict(wl);
    wordlib_destroy(wl);
    return 0;
}

int cmd_check(int argc, char **argv) {
    // Can read from argument or stdin
    const char *text;
    char *allocated = NULL;
    
    if (argc > 0) {
        text = argv[0];
    } else {
        allocated = read_stdin();
        if (!allocated) {
            fprintf(stderr, "Error reading stdin\n");
            return 1;
        }
        text = allocated;
    }
    
    WordLib *wl = load_dict();
    
    // Check text and report unknown words
    // ...
    
    free(allocated);
    wordlib_destroy(wl);
    return 0;
}
```

### Reading from stdin

```c
char *read_stdin(void) {
    size_t capacity = 4096;
    size_t size = 0;
    char *buffer = malloc(capacity);
    
    if (!buffer) return NULL;
    
    int c;
    while ((c = getchar()) != EOF) {
        if (size + 1 >= capacity) {
            capacity *= 2;
            char *new_buf = realloc(buffer, capacity);
            if (!new_buf) {
                free(buffer);
                return NULL;
            }
            buffer = new_buf;
        }
        buffer[size++] = c;
    }
    
    buffer[size] = '\0';
    return buffer;
}
```

### Output Formatting

**Human-readable (default):**
```
$ wordlib check "The quikc brown fox"
Unknown words:
  quikc (at position 4-9)
```

**Machine-readable (for scripting):**
```
$ wordlib check --json "The quikc brown fox"
{"unknown":[{"word":"quikc","start":4,"end":9}]}
```

```c
void output_unknown(const char *word, size_t start, size_t end, 
                    bool json_mode) {
    if (json_mode) {
        printf("{\"word\":\"%s\",\"start\":%zu,\"end\":%zu}", 
               word, start, end);
    } else {
        printf("  %s (at position %zu-%zu)\n", word, start, end);
    }
}
```

### Exit Codes

```c
#define EXIT_OK 0
#define EXIT_ERROR 1
#define EXIT_USAGE 2
#define EXIT_UNKNOWN_WORDS 3  // Check found unknown words

int cmd_check(...) {
    // ...
    if (unknown_count > 0) {
        return EXIT_UNKNOWN_WORDS;  // Useful for scripts
    }
    return EXIT_OK;
}
```

This lets scripts do:
```bash
if ! wordlib check document.txt; then
    echo "Spell check failed"
fi
```

### Help and Usage

```c
void print_usage(void) {
    fprintf(stderr,
        "Usage: wordlib <command> [options] [arguments]\n"
        "\n"
        "Commands:\n"
        "  add <word>...      Add words to dictionary\n"
        "  remove <word>...   Remove words from dictionary\n"
        "  check [text]       Check text for unknown words\n"
        "  complete <prefix>  Show completions for prefix\n"
        "  suggest <word>     Show suggestions for word\n"
        "\n"
        "Options:\n"
        "  -d, --dict PATH    Dictionary file (default: ~/.wordlib/dict.txt)\n"
        "  -n, --max NUM      Maximum results (default: 10)\n"
        "  -h, --help         Show this help\n"
        "  -V, --version      Show version\n"
    );
}
```

## Where to Learn

1. **`man 3 getopt`** - getopt manual page
2. **"The Art of Unix Programming"** - CLI conventions
3. **GNU Coding Standards** - Argument syntax guidelines
4. **Study well-designed CLIs:** git, docker, kubectl

## Practice Exercise

Implement basic CLI skeleton:

```c
int main(int argc, char **argv);
int cmd_add(int argc, char **argv);
int cmd_remove(int argc, char **argv);
int cmd_check(int argc, char **argv);
int cmd_complete(int argc, char **argv);
int cmd_suggest(int argc, char **argv);
void print_usage(void);
```

Test manually:
```bash
./wordlib add hello world
./wordlib complete hel
echo "helo world" | ./wordlib check
./wordlib suggest helo
```

## Connection to Project

Your CLI is:
1. **Testing tool** - Verify engine works
2. **Integration point** - Neovim can shell out to it
3. **Reference implementation** - Shows how to use the API
4. **Useful standalone** - People might use it directly
