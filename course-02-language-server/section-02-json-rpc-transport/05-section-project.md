# Section 2 Project: Working Transport Layer

## Objective

Build a complete transport layer that handles LSP message framing, JSON parsing, and stdio communication. This is the foundation everything else builds on.

## Requirements

1. **Read LSP-framed messages from stdin**
2. **Write LSP-framed messages to stdout**
3. **Parse JSON into usable structures**
4. **Build JSON responses**
5. **Handle edge cases gracefully**

## Specification

### Transport API (`include/transport.h`)

```c
#ifndef TRANSPORT_H
#define TRANSPORT_H

#include <stdio.h>
#include <stdbool.h>

typedef struct Transport Transport;

// Lifecycle
Transport *transport_create(FILE *in, FILE *out);
void transport_destroy(Transport *t);

// Read next message (blocking)
// Returns NULL on EOF or error
// Caller must free returned string
char *transport_read(Transport *t);

// Write message with framing
void transport_write(Transport *t, const char *json);

// Check if transport is still valid
bool transport_ok(Transport *t);

#endif
```

### JSON API (`include/json.h`)

```c
#ifndef JSON_H
#define JSON_H

#include <stdbool.h>
#include <stdint.h>
#include <stddef.h>

typedef struct cJSON JsonValue;

// Parsing and cleanup
JsonValue *json_parse(const char *text);
void json_free(JsonValue *v);
char *json_stringify(const JsonValue *v);

// Type checking
bool json_is_object(const JsonValue *v);
bool json_is_array(const JsonValue *v);
bool json_is_string(const JsonValue *v);
bool json_is_number(const JsonValue *v);
bool json_is_bool(const JsonValue *v);
bool json_is_null(const JsonValue *v);

// Object access
JsonValue *json_get(const JsonValue *obj, const char *key);
const char *json_get_string(const JsonValue *obj, const char *key);
int64_t json_get_int(const JsonValue *obj, const char *key);
double json_get_number(const JsonValue *obj, const char *key);
bool json_get_bool(const JsonValue *obj, const char *key);
bool json_has_key(const JsonValue *obj, const char *key);

// Array access
size_t json_array_length(const JsonValue *arr);
JsonValue *json_array_get(const JsonValue *arr, size_t index);

// Creating values
JsonValue *json_object(void);
JsonValue *json_array(void);
JsonValue *json_string(const char *s);
JsonValue *json_int(int64_t n);
JsonValue *json_number(double n);
JsonValue *json_bool(bool b);
JsonValue *json_null(void);

// Building objects
void json_set(JsonValue *obj, const char *key, JsonValue *value);
void json_set_string(JsonValue *obj, const char *key, const char *s);
void json_set_int(JsonValue *obj, const char *key, int64_t n);
void json_set_bool(JsonValue *obj, const char *key, bool b);
void json_array_push(JsonValue *arr, JsonValue *item);

#endif
```

### Directory Structure

```
wordlib-lsp/
├── include/
│   ├── transport.h
│   └── json.h
├── src/
│   ├── transport.c
│   ├── json.c
│   └── cJSON.c       # Third-party
├── test/
│   ├── test_transport.c
│   └── test_json.c
├── cJSON.h           # Third-party header
└── Makefile
```

## Implementation Guide

### Step 1: Get cJSON

```bash
cd wordlib-lsp
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.c
curl -O https://raw.githubusercontent.com/DaveGamble/cJSON/master/cJSON.h
mv cJSON.c src/
```

### Step 2: Implement json.c

Wrap cJSON with your API. Start simple:

```c
// src/json.c
#include "json.h"
#include "cJSON.h"
#include <stdlib.h>
#include <string.h>

JsonValue *json_parse(const char *text) {
    return cJSON_Parse(text);
}

void json_free(JsonValue *v) {
    cJSON_Delete(v);
}

char *json_stringify(const JsonValue *v) {
    return cJSON_PrintUnformatted((cJSON *)v);
}

// ... implement all other functions
```

### Step 3: Implement transport.c

```c
// src/transport.c
#include "transport.h"
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

struct Transport {
    FILE *in;
    FILE *out;
    bool error;
    char *buffer;
    size_t buf_size;
    size_t buf_len;
    size_t buf_pos;
};

Transport *transport_create(FILE *in, FILE *out) {
    Transport *t = calloc(1, sizeof(Transport));
    if (!t) return NULL;
    
    t->in = in;
    t->out = out;
    t->buf_size = 65536;
    t->buffer = malloc(t->buf_size);
    if (!t->buffer) {
        free(t);
        return NULL;
    }
    
    // Disable output buffering
    if (out) {
        setvbuf(out, NULL, _IONBF, 0);
    }
    
    return t;
}

void transport_destroy(Transport *t) {
    if (t) {
        free(t->buffer);
        free(t);
    }
}

// Read until we find the header-body separator
static bool read_headers(Transport *t, size_t *content_length) {
    char line[256];
    *content_length = 0;
    
    while (fgets(line, sizeof(line), t->in)) {
        // Strip trailing whitespace
        size_t len = strlen(line);
        while (len > 0 && isspace((unsigned char)line[len - 1])) {
            line[--len] = '\0';
        }
        
        // Empty line = end of headers
        if (len == 0) {
            return *content_length > 0;
        }
        
        // Parse Content-Length
        if (strncasecmp(line, "Content-Length:", 15) == 0) {
            *content_length = (size_t)atol(line + 15);
        }
        // Ignore other headers
    }
    
    return false;  // EOF before finding separator
}

char *transport_read(Transport *t) {
    if (t->error) return NULL;
    
    size_t content_length;
    if (!read_headers(t, &content_length)) {
        t->error = feof(t->in) ? false : true;
        return NULL;
    }
    
    // Allocate buffer for content
    char *content = malloc(content_length + 1);
    if (!content) {
        t->error = true;
        return NULL;
    }
    
    // Read exact number of bytes
    size_t total = 0;
    while (total < content_length) {
        size_t n = fread(content + total, 1, content_length - total, t->in);
        if (n == 0) {
            free(content);
            t->error = true;
            return NULL;
        }
        total += n;
    }
    
    content[content_length] = '\0';
    return content;
}

void transport_write(Transport *t, const char *json) {
    if (t->error || !t->out) return;
    
    size_t len = strlen(json);
    fprintf(t->out, "Content-Length: %zu\r\n\r\n", len);
    fwrite(json, 1, len, t->out);
    fflush(t->out);
}

bool transport_ok(Transport *t) {
    return !t->error;
}
```

### Step 4: Write Tests

```c
// test/test_json.c
#include "json.h"
#include <assert.h>
#include <stdio.h>
#include <string.h>

void test_parse_object(void) {
    const char *text = "{\"name\":\"test\",\"value\":42}";
    JsonValue *json = json_parse(text);
    
    assert(json != NULL);
    assert(json_is_object(json));
    assert(strcmp(json_get_string(json, "name"), "test") == 0);
    assert(json_get_int(json, "value") == 42);
    
    json_free(json);
    printf("test_parse_object: PASS\n");
}

void test_parse_array(void) {
    const char *text = "[1, 2, 3]";
    JsonValue *json = json_parse(text);
    
    assert(json != NULL);
    assert(json_is_array(json));
    assert(json_array_length(json) == 3);
    
    json_free(json);
    printf("test_parse_array: PASS\n");
}

void test_build_response(void) {
    JsonValue *response = json_object();
    json_set_string(response, "jsonrpc", "2.0");
    json_set_int(response, "id", 1);
    
    JsonValue *result = json_object();
    json_set_string(result, "name", "wordlib-lsp");
    json_set(response, "result", result);
    
    char *str = json_stringify(response);
    assert(strstr(str, "\"jsonrpc\":\"2.0\"") != NULL);
    assert(strstr(str, "\"id\":1") != NULL);
    assert(strstr(str, "\"name\":\"wordlib-lsp\"") != NULL);
    
    free(str);
    json_free(response);
    printf("test_build_response: PASS\n");
}

void test_nested_access(void) {
    const char *text = "{\"params\":{\"textDocument\":{\"uri\":\"file:///test\"}}}";
    JsonValue *json = json_parse(text);
    
    JsonValue *params = json_get(json, "params");
    JsonValue *td = json_get(params, "textDocument");
    const char *uri = json_get_string(td, "uri");
    
    assert(strcmp(uri, "file:///test") == 0);
    
    json_free(json);
    printf("test_nested_access: PASS\n");
}

int main(void) {
    test_parse_object();
    test_parse_array();
    test_build_response();
    test_nested_access();
    printf("\nAll JSON tests passed!\n");
    return 0;
}
```

```c
// test/test_transport.c
#include "transport.h"
#include "json.h"
#include <assert.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void test_roundtrip(void) {
    int pipefd[2];
    assert(pipe(pipefd) == 0);
    
    FILE *write_end = fdopen(pipefd[1], "w");
    FILE *read_end = fdopen(pipefd[0], "r");
    
    // Write a message
    const char *msg = "{\"test\":\"hello\"}";
    fprintf(write_end, "Content-Length: %zu\r\n\r\n%s", strlen(msg), msg);
    fclose(write_end);
    
    // Read it back
    Transport *t = transport_create(read_end, NULL);
    char *received = transport_read(t);
    
    assert(received != NULL);
    assert(strcmp(received, msg) == 0);
    
    free(received);
    transport_destroy(t);
    fclose(read_end);
    
    printf("test_roundtrip: PASS\n");
}

void test_multiple_messages(void) {
    int pipefd[2];
    pipe(pipefd);
    
    FILE *write_end = fdopen(pipefd[1], "w");
    FILE *read_end = fdopen(pipefd[0], "r");
    
    // Write two messages
    const char *msg1 = "{\"id\":1}";
    const char *msg2 = "{\"id\":2}";
    fprintf(write_end, "Content-Length: %zu\r\n\r\n%s", strlen(msg1), msg1);
    fprintf(write_end, "Content-Length: %zu\r\n\r\n%s", strlen(msg2), msg2);
    fclose(write_end);
    
    // Read both
    Transport *t = transport_create(read_end, NULL);
    
    char *r1 = transport_read(t);
    assert(r1 && strcmp(r1, msg1) == 0);
    free(r1);
    
    char *r2 = transport_read(t);
    assert(r2 && strcmp(r2, msg2) == 0);
    free(r2);
    
    // Third read should return NULL (EOF)
    assert(transport_read(t) == NULL);
    
    transport_destroy(t);
    fclose(read_end);
    
    printf("test_multiple_messages: PASS\n");
}

void test_write_message(void) {
    int pipefd[2];
    pipe(pipefd);
    
    FILE *write_end = fdopen(pipefd[1], "w");
    FILE *read_end = fdopen(pipefd[0], "r");
    
    Transport *t = transport_create(NULL, write_end);
    transport_write(t, "{\"result\":null}");
    transport_destroy(t);
    fclose(write_end);
    
    // Verify framing
    char header[100];
    fgets(header, sizeof(header), read_end);
    assert(strncmp(header, "Content-Length: 15", 18) == 0);
    
    fclose(read_end);
    printf("test_write_message: PASS\n");
}

int main(void) {
    test_roundtrip();
    test_multiple_messages();
    test_write_message();
    printf("\nAll transport tests passed!\n");
    return 0;
}
```

### Step 5: Create Makefile

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -Wpedantic -std=c99 -Iinclude -I.
LDFLAGS =

# Debug build
ifdef DEBUG
CFLAGS += -g -O0 -DDEBUG
else
CFLAGS += -O2
endif

# Source files
SRCS = src/transport.c src/json.c src/cJSON.c

# Test programs
test_json: test/test_json.c $(SRCS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

test_transport: test/test_transport.c $(SRCS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

.PHONY: test clean

test: test_json test_transport
	./test_json
	./test_transport

clean:
	rm -f test_json test_transport
```

## Acceptance Criteria

### JSON Tests Pass
```bash
$ make test_json && ./test_json
test_parse_object: PASS
test_parse_array: PASS
test_build_response: PASS
test_nested_access: PASS
All JSON tests passed!
```

### Transport Tests Pass
```bash
$ make test_transport && ./test_transport
test_roundtrip: PASS
test_multiple_messages: PASS
test_write_message: PASS
All transport tests passed!
```

### No Memory Leaks
```bash
$ valgrind --leak-check=full ./test_json
...
All heap blocks were freed -- no leaks are possible

$ valgrind --leak-check=full ./test_transport
...
All heap blocks were freed -- no leaks are possible
```

### Integration Test
```bash
# test_integration.sh
(
  echo 'Content-Length: 52'
  echo ''
  echo -n '{"jsonrpc":"2.0","id":1,"method":"test","params":{}}'
) | ./test_read_write
```

Where `test_read_write` is:
```c
// Simple program that reads one message and echoes it
int main(void) {
    Transport *t = transport_create(stdin, stdout);
    char *msg = transport_read(t);
    if (msg) {
        transport_write(t, msg);
        free(msg);
    }
    transport_destroy(t);
    return 0;
}
```

## Stretch Goals

1. **Handle partial reads** - Test with slow/limited input
2. **Buffer reuse** - Avoid malloc per message
3. **Large message handling** - Test with 1MB+ content
4. **Unicode validation** - Verify UTF-8 in JSON strings

## Hints

### Case-Insensitive Header Matching
```c
#include <strings.h>  // For strncasecmp on POSIX
// Or implement your own for portability
```

### Testing with Here Documents
```bash
./test_read_write << 'EOF'
Content-Length: 20

{"jsonrpc":"2.0"}
EOF
```

### Debugging Transport Issues
```c
// Add verbose logging to stderr
#define LOG(fmt, ...) fprintf(stderr, "[TRANSPORT] " fmt "\n", ##__VA_ARGS__)
```

## What You'll Learn

By completing this project:
- Low-level stdio handling in C
- JSON parsing with cJSON
- Building clean APIs over third-party code
- Memory management patterns
- Test-driven development in C

## Next Section Preview

Section 3 builds document management on top of this transport. You'll track open files, their contents, and versions—the foundation for diagnostics and completions.
