# Stdio Communication

## Why You Need This

Your requirements specify: "Communication shall occur over stdio." This is simple in concept but has pitfalls. Blocking, buffering, and signal handling all need attention.

## What to Learn

### Why Stdio?

**Pros:**
- Universal—works on every OS
- Simple—no networking code
- Editor spawns server as child process
- Clean process lifecycle

**Cons:**
- Can't share server between editors (one per process)
- Debugging is trickier (can't just print)

### The Basic Loop

```c
int main(void) {
    Transport *transport = transport_create(stdin, stdout);
    Server *server = server_create();
    
    while (server->state != SERVER_STOPPED) {
        char *message = transport_read(transport);
        if (!message) {
            break;  // EOF or error
        }
        
        JsonValue *json = json_parse(message);
        free(message);
        
        if (!json) {
            // Parse error - send error response
            continue;
        }
        
        JsonValue *response = server_handle(server, json);
        json_free(json);
        
        if (response) {
            char *response_str = json_stringify(response);
            transport_write(transport, response_str);
            free(response_str);
            json_free(response);
        }
    }
    
    transport_destroy(transport);
    server_destroy(server);
    return 0;
}
```

### Buffering Gotchas

**stdout buffering can delay messages:**
```c
// BAD: Message might not send immediately
printf("Content-Length: %zu\r\n\r\n%s", len, json);

// GOOD: Force immediate send
fprintf(stdout, "Content-Length: %zu\r\n\r\n%s", len, json);
fflush(stdout);

// Or disable buffering entirely
setvbuf(stdout, NULL, _IONBF, 0);
```

**stdin is usually line-buffered, but that doesn't matter:**
LSP messages aren't line-based. Read exactly the bytes you need.

### Reading Without Blocking Forever

```c
// Option 1: Just let it block (simplest)
// Server main loop blocks on stdin, which is fine

// Option 2: Use select/poll for timeout
#include <poll.h>

bool wait_for_input(FILE *in, int timeout_ms) {
    struct pollfd pfd = {
        .fd = fileno(in),
        .events = POLLIN
    };
    
    int ret = poll(&pfd, 1, timeout_ms);
    return ret > 0 && (pfd.revents & POLLIN);
}

// Usage: Periodic health checks
while (server_is_running(server)) {
    if (wait_for_input(stdin, 1000)) {  // 1 second timeout
        char *msg = transport_read(transport);
        handle(msg);
    } else {
        // No input, do housekeeping
        check_parent_alive(server);
    }
}
```

### EOF Handling

```c
char *transport_read(Transport *t) {
    // ... read headers ...
    
    size_t n = fread(content, 1, content_length, t->in);
    if (n == 0) {
        if (feof(t->in)) {
            // Client closed connection - clean shutdown
            return NULL;
        }
        if (ferror(t->in)) {
            // Read error
            log_error("stdin read error: %s", strerror(errno));
            return NULL;
        }
    }
    
    // ... rest of read ...
}
```

### Signal Handling

```c
#include <signal.h>

static volatile sig_atomic_t should_exit = 0;

void signal_handler(int sig) {
    (void)sig;
    should_exit = 1;
}

int main(void) {
    // Set up signal handlers
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);
    signal(SIGPIPE, SIG_IGN);  // Ignore broken pipe
    
    // Main loop checks flag
    while (!should_exit && server->state != SERVER_STOPPED) {
        // ... process messages ...
    }
    
    // Clean shutdown
    server_shutdown(server);
    return 0;
}
```

### Debugging with stderr

**stdout is for LSP messages only.** Use stderr for everything else:

```c
// Good: Logs go to stderr
void log_info(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    fprintf(stderr, "[INFO] ");
    vfprintf(stderr, fmt, args);
    fprintf(stderr, "\n");
    va_end(args);
}

// Bad: This corrupts the LSP stream!
printf("Debug: processing message\n");
```

### Testing Stdio Communication

```c
// test_stdio.c
void test_stdio_roundtrip(void) {
    // Create pipes to simulate stdin/stdout
    int stdin_pipe[2];
    int stdout_pipe[2];
    pipe(stdin_pipe);
    pipe(stdout_pipe);
    
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child: run server with pipes
        close(stdin_pipe[1]);   // Close write end of stdin
        close(stdout_pipe[0]);  // Close read end of stdout
        
        dup2(stdin_pipe[0], STDIN_FILENO);
        dup2(stdout_pipe[1], STDOUT_FILENO);
        
        execl("./wordlib-lsp", "wordlib-lsp", NULL);
        exit(1);
    }
    
    // Parent: send/receive messages
    close(stdin_pipe[0]);   // Close read end of stdin
    close(stdout_pipe[1]);  // Close write end of stdout
    
    FILE *to_server = fdopen(stdin_pipe[1], "w");
    FILE *from_server = fdopen(stdout_pipe[0], "r");
    
    // Send initialize request
    send_framed(to_server, "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{}}");
    
    // Read response
    char *response = read_framed(from_server);
    assert(strstr(response, "\"result\"") != NULL);
    
    // Cleanup
    fclose(to_server);
    fclose(from_server);
    waitpid(pid, NULL, 0);
}
```

### Non-Blocking Considerations

For a simple LSP server, blocking I/O is fine. The protocol is request-response, so you naturally alternate between reading and writing.

If you need non-blocking:
```c
#include <fcntl.h>

// Make stdin non-blocking
int flags = fcntl(STDIN_FILENO, F_GETFL, 0);
fcntl(STDIN_FILENO, F_SETFL, flags | O_NONBLOCK);

// Then read returns EAGAIN if no data available
ssize_t n = read(STDIN_FILENO, buf, size);
if (n == -1 && errno == EAGAIN) {
    // No data yet, try later
}
```

But this adds complexity. Start with blocking.

## Where to Learn

1. **APUE (Advanced Programming in the Unix Environment):**
   - Chapter 14: Advanced I/O
   - Chapter 15: Interprocess Communication

2. **man pages:**
   - `man stdio`, `man poll`, `man pipe`

3. **Language server implementations:**
   - See how others handle the main loop

## Practice Exercise

Write a simple echo server over stdio:

```c
// echo_server.c
int main(void) {
    setvbuf(stdout, NULL, _IONBF, 0);
    
    char *msg;
    while ((msg = read_framed(stdin)) != NULL) {
        // Echo back
        write_framed(stdout, msg);
        
        // Log to stderr
        fprintf(stderr, "Echoed: %zu bytes\n", strlen(msg));
        
        // Check for exit command
        if (strstr(msg, "\"method\":\"exit\"")) {
            free(msg);
            break;
        }
        free(msg);
    }
    
    return 0;
}
```

Test it:
```bash
echo 'Content-Length: 35

{"jsonrpc":"2.0","method":"test"}' | ./echo_server
```

## Connection to Project

Stdio is your server's connection to the outside world:

```
┌────────────────────────────────────────────────────────┐
│                        Editor                           │
│                                                         │
│  Spawns your server:  wordlib-lsp                      │
│  Connects stdin/stdout to the process                  │
└────────────────────────────────────────────────────────┘
         │                              ▲
         │ stdin                        │ stdout
         ▼                              │
┌────────────────────────────────────────────────────────┐
│                     Your Server                         │
│                                                         │
│  transport_read(stdin)  ────────▶  handle_message()    │
│  transport_write(stdout) ◀────────  send_response()    │
│                                                         │
│  fprintf(stderr, ...)  ────────▶  [logs, ignored]      │
└────────────────────────────────────────────────────────┘
```

Keep it simple: block on read, flush on write, log to stderr.
