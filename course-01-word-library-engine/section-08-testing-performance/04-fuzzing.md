# Fuzzing

## Why You Need This

Tests check what you thought of. Fuzzing finds what you didn't think of. For a tool processing arbitrary user text, fuzzing catches edge cases you'd never manually write.

## What to Learn

### What Is Fuzzing?

Feed random/mutated inputs to your program. See if it crashes. Repeat millions of times.

**Effective at finding:**
- Buffer overflows
- Integer overflows
- Null pointer dereferences
- Assertion failures
- Infinite loops
- Memory corruption

### AFL (American Fuzzy Lop)

**Install:**
```bash
# Ubuntu/Debian
sudo apt install afl

# macOS
brew install afl-fuzz
```

**Instrument your code:**
```bash
afl-gcc -o wordlib_fuzz cli/main.c src/*.c
```

**Create seed inputs:**
```bash
mkdir -p fuzz_input
echo "hello world" > fuzz_input/simple.txt
echo "café naïve" > fuzz_input/utf8.txt
echo "" > fuzz_input/empty.txt
```

**Run fuzzer:**
```bash
mkdir fuzz_output
afl-fuzz -i fuzz_input -o fuzz_output -- ./wordlib_fuzz check @@
```

The `@@` gets replaced with the fuzz input file.

### Fuzz Harness

Create a dedicated fuzzing entry point:

```c
// fuzz/fuzz_check.c
#include <stdio.h>
#include <stdlib.h>
#include "wordlib.h"

int main(int argc, char **argv) {
    if (argc != 2) return 1;
    
    // Read input file
    FILE *f = fopen(argv[1], "rb");
    if (!f) return 1;
    
    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    fseek(f, 0, SEEK_SET);
    
    char *data = malloc(size + 1);
    fread(data, 1, size, f);
    data[size] = '\0';
    fclose(f);
    
    // Create library with minimal dictionary
    WordLib *wl = wordlib_create();
    wordlib_add_word(wl, "test");
    wordlib_add_word(wl, "hello");
    
    // Fuzz target: check arbitrary text
    WLUnknownWord results[100];
    wordlib_check_text(wl, data, results, 100);
    
    // Also fuzz completion
    const char *comp[10];
    if (size > 0) {
        wordlib_complete(wl, data, comp, 10);
    }
    
    // Also fuzz suggestions
    WLSuggestion sugg[10];
    if (size > 0 && size < 100) {  // Avoid pathological cases
        wordlib_suggest(wl, data, 2, sugg, 10);
    }
    
    wordlib_destroy(wl);
    free(data);
    
    return 0;
}
```

### LibFuzzer (LLVM)

Modern fuzzer integrated with LLVM:

```c
// fuzz/fuzz_tokenizer.c
#include <stdint.h>
#include <stddef.h>
#include "tokenizer.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // Null-terminate the input
    char *text = malloc(size + 1);
    memcpy(text, data, size);
    text[size] = '\0';
    
    // Fuzz tokenization
    Token tokens[100];
    TokenizerConfig cfg = tokenizer_default_config();
    tokenize(text, &cfg, tokens, 100);
    
    free(text);
    return 0;
}
```

**Compile:**
```bash
clang -g -fsanitize=fuzzer,address -o fuzz_tokenizer \
    fuzz/fuzz_tokenizer.c src/tokenizer.c
```

**Run:**
```bash
mkdir corpus
./fuzz_tokenizer corpus/
```

### What to Fuzz

**High-value targets for your project:**

1. **Tokenizer** - Arbitrary text input
2. **UTF-8 validation** - Binary data
3. **Levenshtein** - Long strings, special characters
4. **File parser** - Malformed dictionary files

**Fuzz the tokenizer:**
```c
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    char *text = malloc(size + 1);
    memcpy(text, data, size);
    text[size] = '\0';
    
    Token tokens[1000];
    TokenizerConfig cfg = tokenizer_default_config();
    
    // Try different configurations
    cfg.include_apostrophes = (size % 2);
    cfg.include_hyphens = (size % 3);
    
    size_t count = tokenize(text, &cfg, tokens, 1000);
    
    // Verify spans are valid
    for (size_t i = 0; i < count; i++) {
        assert(tokens[i].span.start <= size);
        assert(tokens[i].span.end <= size);
        assert(tokens[i].span.start <= tokens[i].span.end);
    }
    
    free(text);
    return 0;
}
```

**Fuzz the file loader:**
```c
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // Write fuzz data to temp file
    FILE *f = fopen("/tmp/fuzz_dict.txt", "wb");
    fwrite(data, 1, size, f);
    fclose(f);
    
    // Try to load it
    WordLib *wl = wordlib_create();
    wordlib_load(wl, "/tmp/fuzz_dict.txt");  // Should not crash
    wordlib_destroy(wl);
    
    return 0;
}
```

### Analyzing Crashes

AFL saves crashing inputs in `fuzz_output/crashes/`:

```bash
# Reproduce crash
./wordlib_fuzz check fuzz_output/crashes/id:000000,sig:11,...

# Debug with GDB
gdb --args ./wordlib_fuzz check fuzz_output/crashes/id:000000
```

### Continuous Fuzzing

```yaml
# GitHub Actions example
fuzz:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v2
    
    - name: Build fuzz target
      run: |
        clang -g -fsanitize=fuzzer,address -o fuzz_check \
          fuzz/fuzz_check.c src/*.c
    
    - name: Run fuzzer (time-limited)
      run: |
        mkdir -p corpus
        timeout 300 ./fuzz_check corpus/ || true
    
    - name: Upload crashes
      if: failure()
      uses: actions/upload-artifact@v2
      with:
        name: crashes
        path: corpus/crash-*
```

## Where to Learn

1. **AFL documentation** - Excellent tutorials
2. **libFuzzer documentation** - LLVM fuzzer guide
3. **"Fuzzing Book"** - Online textbook
4. **OSS-Fuzz** - Google's continuous fuzzing service

## Practice Exercise

1. Create fuzz harnesses for:
   - Tokenizer
   - UTF-8 validation
   - Dictionary file parsing

2. Run each for at least 10 minutes

3. Fix any crashes found

4. Add crash inputs to regression tests:
```c
void test_fuzz_crash_001(void) {
    // This input crashed the fuzzer
    const char *evil = "\xff\xfe\x00\x00...";
    // Should not crash anymore
    tokenize(evil, &cfg, tokens, 100);
    PASS();
}
```

## Connection to Project

Fuzzing finds bugs you'd never think to test:
- What if someone pastes binary data?
- What if a dictionary file is corrupted?
- What if UTF-8 is malformed?

A tool that crashes on weird input loses user trust. Fuzzing builds robustness.
