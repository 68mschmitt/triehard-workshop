# Workspace Dictionaries

## Why You Need This

Your requirements say: "Support a global dictionary and a workspace-specific dictionary (stored in workspace root)" and "Prefer workspace dictionaries when present."

Different projects need different words—a medical project vs a gaming project have very different vocabularies.

## What to Learn

### Dictionary Hierarchy

```
Global Dictionary
    └── /home/user/.wordlib/dictionary.txt
    └── Contains: common words, user's personal additions

Workspace Dictionary
    └── /project/.wordlib/dictionary.txt
    └── Contains: project-specific terms, tech jargon

Effective Dictionary = Global + Workspace (workspace wins on conflict)
```

### Path Resolution

```c
typedef struct {
    char *global_dict_path;
    char *workspace_dict_path;
} DictionaryPaths;

DictionaryPaths resolve_dictionary_paths(const char *workspace_root_uri) {
    DictionaryPaths paths = {0};
    
    // Global dictionary
    const char *home = getenv("HOME");
    if (home) {
        size_t len = strlen(home) + 30;
        paths.global_dict_path = malloc(len);
        snprintf(paths.global_dict_path, len, 
                 "%s/.wordlib/dictionary.txt", home);
    }
    
    // Workspace dictionary
    if (workspace_root_uri) {
        char *workspace_path = uri_to_path(workspace_root_uri);
        if (workspace_path) {
            size_t len = strlen(workspace_path) + 30;
            paths.workspace_dict_path = malloc(len);
            snprintf(paths.workspace_dict_path, len,
                     "%s/.wordlib/dictionary.txt", workspace_path);
            free(workspace_path);
        }
    }
    
    return paths;
}
```

### Loading Multiple Dictionaries

```c
void load_dictionaries(Server *server) {
    // Load global dictionary first
    if (server->paths.global_dict_path) {
        if (file_exists(server->paths.global_dict_path)) {
            wordlib_load(server->engine, server->paths.global_dict_path);
            log_info("Loaded global dictionary: %s", 
                     server->paths.global_dict_path);
        }
    }
    
    // Load workspace dictionary (adds to existing)
    if (server->paths.workspace_dict_path) {
        if (file_exists(server->paths.workspace_dict_path)) {
            wordlib_load(server->engine, server->paths.workspace_dict_path);
            log_info("Loaded workspace dictionary: %s",
                     server->paths.workspace_dict_path);
        }
    }
}
```

### Choosing Where to Save

When user adds a word, where should it go?

```c
// Option 1: Always workspace (if available)
char *get_save_dictionary_path(Server *server) {
    if (server->paths.workspace_dict_path) {
        // Ensure directory exists
        ensure_directory_exists(server->paths.workspace_dict_path);
        return server->paths.workspace_dict_path;
    }
    return server->paths.global_dict_path;
}

// Option 2: Let user choose (via code action)
JsonValue *create_add_word_actions(const char *word, const JsonValue *diagnostic,
                                   Server *server) {
    JsonValue *actions = json_array();
    
    // Add to workspace
    if (server->paths.workspace_dict_path) {
        JsonValue *action = json_object();
        char title[256];
        snprintf(title, sizeof(title), "Add '%s' to workspace dictionary", word);
        json_set_string(action, "title", title);
        // ... command with "workspace" argument
        json_array_push(actions, action);
    }
    
    // Add to global
    if (server->paths.global_dict_path) {
        JsonValue *action = json_object();
        char title[256];
        snprintf(title, sizeof(title), "Add '%s' to global dictionary", word);
        json_set_string(action, "title", title);
        // ... command with "global" argument
        json_array_push(actions, action);
    }
    
    return actions;
}
```

### Handling Add Word with Target

```c
JsonValue *execute_add_word(Server *server, const JsonValue *args) {
    const char *word = json_get_string(json_array_get(args, 0), NULL);
    const char *target = json_get_string(json_array_get(args, 1), NULL);
    
    if (!word) return json_null();
    
    // Add to engine
    wordlib_add_word(server->engine, word);
    
    // Determine save path
    const char *save_path;
    if (target && strcmp(target, "global") == 0) {
        save_path = server->paths.global_dict_path;
    } else {
        // Default to workspace if available
        save_path = server->paths.workspace_dict_path 
                    ? server->paths.workspace_dict_path
                    : server->paths.global_dict_path;
    }
    
    // Save
    if (save_path) {
        ensure_directory_exists(save_path);
        append_word_to_file(save_path, word);
    }
    
    revalidate_all_documents(server);
    return json_null();
}

void append_word_to_file(const char *path, const char *word) {
    FILE *f = fopen(path, "a");
    if (f) {
        fprintf(f, "%s\n", word);
        fclose(f);
    }
}
```

### Ensuring Directory Exists

```c
#include <sys/stat.h>
#include <errno.h>

void ensure_directory_exists(const char *file_path) {
    // Get directory part
    char *dir = strdup(file_path);
    char *last_slash = strrchr(dir, '/');
    if (last_slash) {
        *last_slash = '\0';
        
        // Create directory if it doesn't exist
        struct stat st;
        if (stat(dir, &st) == -1) {
            mkdir(dir, 0755);
        }
    }
    free(dir);
}
```

### Workspace Change Handling

When workspace root changes (rare):

```c
void handle_workspace_folders_change(Server *server, const JsonValue *params) {
    const JsonValue *added = json_get(params, "event.added");
    const JsonValue *removed = json_get(params, "event.removed");
    
    // Handle folder changes
    // Typically: reload dictionaries, revalidate documents
    
    if (json_array_length(added) > 0) {
        // New workspace folder
        const JsonValue *folder = json_array_get(added, 0);
        const char *uri = json_get_string(folder, "uri");
        
        // Update paths
        free(server->paths.workspace_dict_path);
        server->paths.workspace_dict_path = /* resolve from uri */;
        
        // Reload
        load_dictionaries(server);
        revalidate_all_documents(server);
    }
}
```

### .wordlib Directory Structure

```
project/
├── .wordlib/
│   ├── dictionary.txt      # Project-specific words
│   └── config.json         # (Optional) Project-specific settings
├── src/
└── docs/
```

## Where to Learn

1. **LSP Specification - Workspace Folders:**
   - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_workspaceFolders

2. **File system operations in C:**
   - POSIX file handling, directory creation

## Practice Exercise

Test dictionary path resolution:

```c
void test_dictionary_paths(void) {
    DictionaryPaths paths;
    
    // With workspace
    paths = resolve_dictionary_paths("file:///home/user/project");
    assert(strstr(paths.global_dict_path, ".wordlib/dictionary.txt") != NULL);
    assert(strstr(paths.workspace_dict_path, "project/.wordlib") != NULL);
    free(paths.global_dict_path);
    free(paths.workspace_dict_path);
    
    // Without workspace
    paths = resolve_dictionary_paths(NULL);
    assert(paths.global_dict_path != NULL);
    assert(paths.workspace_dict_path == NULL);
    free(paths.global_dict_path);
    
    printf("test_dictionary_paths: PASS\n");
}
```

## Connection to Project

Workspace dictionaries enable project-specific vocabularies:

```
Project A (Medical):
  .wordlib/dictionary.txt: cardiomyopathy, angioplasty, ...

Project B (Gaming):
  .wordlib/dictionary.txt: respawn, powerup, leaderboard, ...

Global:
  ~/.wordlib/dictionary.txt: common words, your name, ...
```

Each project gets its own vocabulary without polluting others.
