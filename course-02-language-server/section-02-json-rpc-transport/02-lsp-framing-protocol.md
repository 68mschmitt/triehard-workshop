# LSP Framing Protocol

## Why You Need This

Your requirements specify: "Messages shall follow LSP framing (`Content-Length` headers)." Without proper framing, you can't tell where one message ends and the next begins. This is the foundation of reliable communication.

## What to Learn

### The Problem

TCP/stdio are byte streams, not message streams. If client sends:

```
{"jsonrpc":"2.0","id":1,"method":"foo"}{"jsonrpc":"2.0","id":2,"method":"bar"}
```

How do you know where the first message ends? JSON doesn't help—it allows `}{` between objects.

### LSP Solution: Content-Length Header

Every LSP message is prefixed with HTTP-style headers:

```
Content-Length: 52\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}
```

**Format:**
```
Header-Name: Header-Value\r\n
Header-Name: Header-Value\r\n
\r\n
<content of exactly Content-Length bytes>
```

### The Headers

**Required:**
- `Content-Length`: Number of bytes in the content (not including headers)

**Optional:**
- `Content-Type`: Always `application/vscode-jsonrpc; charset=utf-8` if present

In practice, only `Content-Length` is used.

### Reading a Message

```c
typedef struct {
    char *content;
    size_t length;
} Message;

// Read headers, extract Content-Length, read that many bytes
Message *read_message(FILE *in) {
    char line[256];
    size_t content_length = 0;
    
    // Read headers until empty line
    while (fgets(line, sizeof(line), in)) {
        // Empty line (just \r\n or \n) marks end of headers
        if (line[0] == '\r' || line[0] == '\n') {
            break;
        }
        
        // Parse Content-Length header
        if (strncasecmp(line, "Content-Length:", 15) == 0) {
            content_length = (size_t)atol(line + 15);
        }
        // Ignore other headers
    }
    
    if (content_length == 0) {
        return NULL;  // Invalid or EOF
    }
    
    // Allocate and read content
    Message *msg = malloc(sizeof(Message));
    msg->content = malloc(content_length + 1);
    msg->length = content_length;
    
    size_t read = fread(msg->content, 1, content_length, in);
    if (read != content_length) {
        free(msg->content);
        free(msg);
        return NULL;  // Incomplete message
    }
    
    msg->content[content_length] = '\0';
    return msg;
}
```

### Writing a Message

```c
void write_message(FILE *out, const char *content) {
    size_t len = strlen(content);
    
    fprintf(out, "Content-Length: %zu\r\n", len);
    fprintf(out, "\r\n");
    fwrite(content, 1, len, out);
    fflush(out);  // Important! Don't buffer LSP messages
}

// With JSON value
void send_json(FILE *out, const JsonValue *json) {
    char *content = json_stringify(json);
    write_message(out, content);
    free(content);
}
```

### Edge Cases

**Header case insensitivity:**
```c
// Both are valid:
// Content-Length: 42
// content-length: 42
if (strncasecmp(line, "Content-Length:", 15) == 0)
```

**\r\n vs \n:**
Some systems might send just `\n`. Be tolerant:
```c
// Check for \r\n or just \n
if (line[0] == '\r' || line[0] == '\n') {
    break;  // End of headers
}
```

**Partial reads:**
stdio might not return all bytes at once. Use a loop:
```c
size_t total_read = 0;
while (total_read < content_length) {
    size_t n = fread(msg->content + total_read, 1, 
                     content_length - total_read, in);
    if (n == 0) break;  // EOF or error
    total_read += n;
}
```

### Buffering Strategy

For better performance, buffer incoming data:

```c
typedef struct {
    char *buffer;
    size_t capacity;
    size_t length;
    size_t position;
} ReadBuffer;

// Fill buffer from stdin
void buffer_fill(ReadBuffer *buf, FILE *in) {
    // Move unconsumed data to front
    if (buf->position > 0) {
        memmove(buf->buffer, buf->buffer + buf->position, 
                buf->length - buf->position);
        buf->length -= buf->position;
        buf->position = 0;
    }
    
    // Read more data
    size_t space = buf->capacity - buf->length;
    size_t n = fread(buf->buffer + buf->length, 1, space, in);
    buf->length += n;
}

// Check if we have a complete message
bool buffer_has_message(ReadBuffer *buf, size_t *content_length) {
    // Look for \r\n\r\n marking end of headers
    char *header_end = strstr(buf->buffer + buf->position, "\r\n\r\n");
    if (!header_end) return false;
    
    // Parse Content-Length
    char *cl = strcasestr(buf->buffer + buf->position, "Content-Length:");
    if (!cl || cl > header_end) return false;
    
    *content_length = (size_t)atol(cl + 15);
    
    // Check if we have full content
    size_t header_size = (header_end + 4) - (buf->buffer + buf->position);
    return buf->length - buf->position >= header_size + *content_length;
}
```

### Complete Transport Module

```c
// transport.h
typedef struct Transport Transport;

Transport *transport_create(FILE *in, FILE *out);
void transport_destroy(Transport *t);

// Blocking read - returns NULL on EOF/error
char *transport_read(Transport *t);

// Write message
void transport_write(Transport *t, const char *json);

// transport.c
struct Transport {
    FILE *in;
    FILE *out;
    char *buffer;
    size_t buf_capacity;
    size_t buf_length;
    size_t buf_pos;
};

Transport *transport_create(FILE *in, FILE *out) {
    Transport *t = calloc(1, sizeof(Transport));
    t->in = in;
    t->out = out;
    t->buf_capacity = 65536;  // 64KB initial buffer
    t->buffer = malloc(t->buf_capacity);
    return t;
}

char *transport_read(Transport *t) {
    // Implementation using buffering strategy above
    // Returns malloc'd string that caller must free
}

void transport_write(Transport *t, const char *json) {
    size_t len = strlen(json);
    fprintf(t->out, "Content-Length: %zu\r\n\r\n", len);
    fwrite(json, 1, len, t->out);
    fflush(t->out);
}
```

## Where to Learn

1. **LSP Specification - Base Protocol:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#headerPart
   
2. **HTTP/1.1 Message Format (similar concept):**
   - RFC 2616 section 4

3. **Existing implementations:**
   - Look at how `clangd` or `rust-analyzer` handle transport

## Practice Exercise

Write a test that:
1. Creates a pipe
2. Writes a framed message to one end
3. Reads and parses from the other end
4. Verifies content matches

```c
void test_framing_roundtrip(void) {
    int pipefd[2];
    pipe(pipefd);
    
    FILE *write_end = fdopen(pipefd[1], "w");
    FILE *read_end = fdopen(pipefd[0], "r");
    
    // Write
    const char *msg = "{\"test\":\"hello\"}";
    fprintf(write_end, "Content-Length: %zu\r\n\r\n%s", strlen(msg), msg);
    fflush(write_end);
    fclose(write_end);
    
    // Read
    Transport *t = transport_create(read_end, NULL);
    char *received = transport_read(t);
    
    assert(strcmp(received, msg) == 0);
    printf("Framing roundtrip: PASS\n");
    
    free(received);
    transport_destroy(t);
    fclose(read_end);
}
```

## Connection to Project

The transport layer is the lowest level of your server:

```
stdin → Transport → JSON Parser → Dispatcher → Handlers
                                      ↓
stdout ← Transport ← JSON Builder ←───┘
```

Get this right and you never think about it again. Every message arrives as a clean JSON string, every response gets properly framed.
