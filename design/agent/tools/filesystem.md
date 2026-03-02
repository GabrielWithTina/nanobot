# Filesystem Tools — ReadFile, WriteFile, EditFile, ListDir

**Source:** `nanobot/agent/tools/filesystem.py`

## Purpose

Four tools for file system operations: reading, writing, editing (find-and-replace), and directory listing. All support workspace-relative path resolution and optional directory restriction.

## Tools Summary

| Tool | Name | Parameters | Description |
|------|------|-----------|-------------|
| ReadFileTool | `read_file` | `path` | Read file contents |
| WriteFileTool | `write_file` | `path`, `content` | Create/overwrite file |
| EditFileTool | `edit_file` | `path`, `old_text`, `new_text` | Find-and-replace |
| ListDirTool | `list_dir` | `path` | List directory contents |

## Path Resolution

All tools share a common path resolver:

```mermaid
flowchart TD
    A["_resolve_path(path, workspace, allowed_dir)"] --> B["Expand ~ in path"]
    B --> C{Absolute path?}
    C -- Yes --> D[Use as-is]
    C -- No --> E["Prepend workspace"]
    D & E --> F[".resolve() — normalize"]
    F --> G{allowed_dir set?}
    G -- No --> H[Return resolved path]
    G -- Yes --> I{"Path inside allowed_dir?"}
    I -- Yes --> H
    I -- No --> J["PermissionError!"]

    style J fill:#ffcdd2
```

When `restrict_to_workspace` is enabled in config, `allowed_dir` is set to the workspace path, preventing the agent from accessing files outside its sandbox.

---

## ReadFileTool

```mermaid
flowchart TD
    A["read_file(path)"] --> B["_resolve_path(path)"]
    B --> C{File exists?}
    C -- No --> D["Error: File not found"]
    C -- Yes --> E{Is file?}
    E -- No --> F["Error: Not a file"]
    E -- Yes --> G["Read UTF-8 content"]
    G --> H["Return content"]
```

---

## WriteFileTool

```mermaid
flowchart TD
    A["write_file(path, content)"] --> B["_resolve_path(path)"]
    B --> C["Create parent dirs (mkdir -p)"]
    C --> D["Write UTF-8 content"]
    D --> E["Return: wrote N bytes to path"]
```

---

## EditFileTool

The most complex filesystem tool. Performs exact string matching with helpful error messages.

```mermaid
flowchart TD
    A["edit_file(path, old_text, new_text)"] --> B["_resolve_path(path)"]
    B --> C{File exists?}
    C -- No --> D["Error: File not found"]
    C -- Yes --> E["Read current content"]
    E --> F{"old_text in content?"}

    F -- No --> G["_not_found_message()"]
    G --> G1{Best match > 50% similar?}
    G1 -- Yes --> G2["Show unified diff of best match"]
    G1 -- No --> G3["No similar text found"]

    F -- Yes --> H{"Appears > 1 time?"}
    H -- Yes --> I["Warning: appears N times.<br/>Provide more context."]
    H -- No --> J["Replace (once)"]
    J --> K["Write updated content"]
    K --> L["Return: Successfully edited"]
```

### Fuzzy Match on Failure

When `old_text` is not found, the tool scans for the most similar block using `difflib.SequenceMatcher`:

```mermaid
flowchart TD
    A["_not_found_message(old_text, content)"] --> B["Split content and old_text into lines"]
    B --> C["Slide window over content lines"]
    C --> D["SequenceMatcher.ratio() for each window"]
    D --> E{Best ratio > 0.5?}
    E -- Yes --> F["Generate unified diff<br/>showing expected vs actual"]
    E -- No --> G["'No similar text found.<br/>Verify file content.'"]
```

This helps the LLM self-correct when it guesses wrong about a file's exact content.

---

## ListDirTool

```mermaid
flowchart TD
    A["list_dir(path)"] --> B["_resolve_path(path)"]
    B --> C{Dir exists?}
    C -- No --> D["Error: Directory not found"]
    C -- Yes --> E{Is directory?}
    E -- No --> F["Error: Not a directory"]
    E -- Yes --> G["Sort entries"]
    G --> H["Format with emoji prefixes"]
    H --> I["Return listing"]
```

Output format:
```
📁 src
📁 tests
📄 README.md
📄 setup.py
```
