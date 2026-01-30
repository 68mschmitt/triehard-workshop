# Testing LSP Servers

## Why You Need This

Your requirements say you need a server that "behaves predictably across document edits" and handles various scenarios correctly. Testing LSP servers is different from testing regular libraries—you need to simulate the protocol.

## What to Learn

### Testing Layers

**1. Unit Tests:** Individual functions
```c
void test_byte_to_position(void);
void test_prefix_extraction(void);
void test_diagnostic_creation(void);
```

**2. Integration Tests:** Components working together
```c
void test_document_open_triggers_diagnostics(void);
void test_completion_uses_dictionary(void);
```

**3. Protocol Tests:** Full LSP message flow
```c
void test_initialize_handshake(void);
void test_lifecycle_sequence(void);
```

### Unit Testing Pattern

```c
// test/test_unit.c
#include "diagnostics.h"
#include <assert.h>

void test_position_conversion(void) {
    const char *text = "Hello\nWorld";
    
    // Test various positions
    Position p = byte_to_position(text, 0);
    assert(p.line == 0 && p.character == 0);
    
    p = byte_to_position(text, 6);
    assert(p.line == 1 && p.character == 0);
    
    printf("test_position_conversion: PASS\n");
}
```

### Integration Testing Pattern

```c
// test/test_integration.c
void test_completion_flow(void) {
    Server *server = test_server_create();
    
    // Setup: add words and open document
    wordlib_add_word(server->engine, "hello");
    wordlib_add_word(server->engine, "help");
    document_store_open(server->documents, "file:///test.txt",
                       "plaintext", 1, "hel");
    
    // Act: simulate completion request
    JsonValue *params = json_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":3}}");
    
    JsonValue *result = handle_completion(server, params);
    
    // Assert
    assert(json_array_length(json_get(result, "items")) >= 2);
    
    json_free(params);
    json_free(result);
    test_server_destroy(server);
}
```

### Protocol Testing with Replay

Record and replay LSP sessions:

```c
// test/test_protocol.c
void test_session_replay(void) {
    const char *messages[] = {
        // Initialize
        "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\","
        "\"params\":{\"rootUri\":null,\"capabilities\":{}}}",
        
        // Initialized
        "{\"jsonrpc\":\"2.0\",\"method\":\"initialized\",\"params\":{}}",
        
        // Open document
        "{\"jsonrpc\":\"2.0\",\"method\":\"textDocument/didOpen\","
        "\"params\":{\"textDocument\":{"
        "\"uri\":\"file:///test.txt\",\"languageId\":\"plaintext\","
        "\"version\":1,\"text\":\"Hello wrold\"}}}",
        
        // Completion
        "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"textDocument/completion\","
        "\"params\":{\"textDocument\":{\"uri\":\"file:///test.txt\"},"
        "\"position\":{\"line\":0,\"character\":5}}}",
        
        // Shutdown
        "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"shutdown\",\"params\":null}",
        
        // Exit
        "{\"jsonrpc\":\"2.0\",\"method\":\"exit\"}",
        NULL
    };
    
    TestServer *ts = test_server_start();
    
    for (int i = 0; messages[i]; i++) {
        test_server_send(ts, messages[i]);
        
        // Check for response if it's a request
        if (strstr(messages[i], "\"id\":")) {
            char *response = test_server_receive(ts);
            assert(strstr(response, "\"result\"") != NULL ||
                   strstr(response, "\"error\"") != NULL);
            free(response);
        }
    }
    
    test_server_stop(ts);
    printf("test_session_replay: PASS\n");
}
```

### Mock Transport

```c
// test/mock_transport.c
typedef struct {
    char **input_messages;
    size_t input_count;
    size_t input_pos;
    
    char **output_messages;
    size_t output_count;
    size_t output_capacity;
} MockTransport;

MockTransport *mock_transport_create(const char **input, size_t count) {
    MockTransport *mt = calloc(1, sizeof(MockTransport));
    mt->input_messages = (char **)input;
    mt->input_count = count;
    mt->output_capacity = 100;
    mt->output_messages = malloc(mt->output_capacity * sizeof(char *));
    return mt;
}

char *mock_transport_read(MockTransport *mt) {
    if (mt->input_pos >= mt->input_count) {
        return NULL;
    }
    return strdup(mt->input_messages[mt->input_pos++]);
}

void mock_transport_write(MockTransport *mt, const char *msg) {
    if (mt->output_count < mt->output_capacity) {
        mt->output_messages[mt->output_count++] = strdup(msg);
    }
}

char *mock_transport_get_output(MockTransport *mt, size_t index) {
    if (index < mt->output_count) {
        return mt->output_messages[index];
    }
    return NULL;
}
```

### Testing with Real Editor

```bash
# test_with_neovim.sh
nvim --headless -c "
lua << EOF
  vim.lsp.start({
    name = 'wordlib-test',
    cmd = { './wordlib-lsp' },
    on_init = function(client)
      -- Create test file
      vim.cmd('edit /tmp/test.txt')
      vim.api.nvim_buf_set_lines(0, 0, -1, false, {'Hello wrold'})
      
      -- Wait for diagnostics
      vim.defer_fn(function()
        local diagnostics = vim.diagnostic.get(0)
        assert(#diagnostics == 1, 'Expected 1 diagnostic')
        print('TEST PASSED')
        vim.cmd('quit!')
      end, 1000)
    end
  })
EOF
"
```

### Test Fixtures

```c
// test/fixtures.h
// Common test utilities

Server *test_server_create(void) {
    Server *s = server_create();
    // Use in-memory dictionary
    // Disable persistence
    return s;
}

void test_server_destroy(Server *s) {
    server_destroy(s);
}

JsonValue *test_parse(const char *json) {
    JsonValue *v = json_parse(json);
    assert(v != NULL);
    return v;
}

void assert_diagnostic_count(Server *server, const char *uri, size_t expected) {
    Document *doc = document_store_get(server->documents, uri);
    JsonValue *diags = compute_diagnostics(server, doc);
    size_t actual = json_array_length(diags);
    assert(actual == expected);
    json_free(diags);
}
```

### Continuous Integration

```makefile
# Makefile
.PHONY: test test-unit test-integration test-valgrind

test: test-unit test-integration

test-unit: $(TEST_UNIT_BINS)
	@for t in $(TEST_UNIT_BINS); do ./$$t || exit 1; done

test-integration: $(TEST_INT_BINS)
	@for t in $(TEST_INT_BINS); do ./$$t || exit 1; done

test-valgrind: $(TEST_BINS)
	@for t in $(TEST_BINS); do \
		valgrind --leak-check=full --error-exitcode=1 ./$$t || exit 1; \
	done
```

## Where to Learn

1. **Testing C programs:** Check, Unity, or custom assertions
2. **LSP test suites:** How other servers are tested
3. **CI/CD for C:** GitHub Actions, GitLab CI

## Practice Exercise

Write a comprehensive test:

```c
void test_full_workflow(void) {
    // Setup
    Server *server = test_server_create();
    
    // Initialize
    JsonValue *init_result = handle_initialize(server, test_parse(
        "{\"rootUri\":null,\"capabilities\":{}}"));
    assert(json_has_key(init_result, "capabilities"));
    json_free(init_result);
    
    handle_initialized(server, test_parse("{}"));
    
    // Open document with typo
    handle_did_open(server, test_parse(
        "{\"textDocument\":{\"uri\":\"file:///test.txt\","
        "\"languageId\":\"plaintext\",\"version\":1,"
        "\"text\":\"Hello wrold\"}}"));
    
    // Should have diagnostic
    assert_diagnostic_count(server, "file:///test.txt", 1);
    
    // Add word to dictionary
    handle_execute_command(server, test_parse(
        "{\"command\":\"wordlib.addWord\",\"arguments\":[\"wrold\"]}"));
    
    // Should have no diagnostics
    assert_diagnostic_count(server, "file:///test.txt", 0);
    
    // Cleanup
    test_server_destroy(server);
    printf("test_full_workflow: PASS\n");
}
```

## Connection to Project

Testing ensures your server works:

| Without Tests | With Tests |
|---------------|------------|
| "I think it works" | "Tests prove it works" |
| Bugs found by users | Bugs caught before release |
| Fear of changes | Confidence in refactoring |
| Manual verification | Automated CI/CD |

A well-tested server is a trustworthy server.
