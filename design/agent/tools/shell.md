# ExecTool — Shell Command Execution

**Source:** `nanobot/agent/tools/shell.py`

## Purpose

Runs shell commands via `asyncio.create_subprocess_shell` with safety guards, timeout enforcement, and output truncation. The most powerful — and most dangerous — tool in the agent's toolbox.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command` | string | Yes | Shell command to run |
| `working_dir` | string | No | Override working directory |

## Execution Flow

```mermaid
flowchart TD
    A["run command"] --> B["Resolve cwd"]
    B --> C["_guard_command(command, cwd)"]
    C --> D{Blocked?}
    D -- Yes --> E["Return guard error"]

    D -- No --> F["Prepare env (PATH append)"]
    F --> G["Create subprocess"]
    G --> H["Wait with timeout"]

    H -- timeout --> I["Kill process"]
    I --> J["Error: timed out after Ns"]

    H -- success --> K["Decode stdout + stderr"]
    K --> L{returncode != 0?}
    L -- Yes --> M["Append exit code"]
    L -- No --> N["Result ready"]

    M & N --> O{len > 10000?}
    O -- Yes --> P["Truncate with count"]
    O -- No --> Q["Return result"]
    P --> Q

    style E fill:#ffcdd2
    style J fill:#ffcdd2
```

## Safety Guard System

```mermaid
flowchart TD
    A["_guard_command(command, cwd)"] --> B["Check deny_patterns"]
    B --> C{Match?}
    C -- Yes --> DENY["Blocked: dangerous pattern"]

    C -- No --> D{allow_patterns set?}
    D -- Yes --> E{Any match?}
    E -- No --> DENY2["Blocked: not in allowlist"]
    E -- Yes --> F

    D -- No --> F{restrict_to_workspace?}
    F -- No --> OK["Pass"]
    F -- Yes --> G["Check for ../ traversal"]
    G --> H{Found?}
    H -- Yes --> DENY3["Blocked: path traversal"]
    H -- No --> I["Extract absolute paths"]
    I --> J{"All inside workspace?"}
    J -- No --> DENY4["Blocked: path outside workspace"]
    J -- Yes --> OK

    style DENY fill:#ffcdd2
    style DENY2 fill:#ffcdd2
    style DENY3 fill:#ffcdd2
    style DENY4 fill:#ffcdd2
```

### Default Deny Patterns

| Pattern | Blocks |
|---------|--------|
| `rm -rf`, `rm -r` | Recursive deletion |
| `del /f`, `del /q` | Windows force delete |
| `rmdir /s` | Windows recursive remove |
| `format` (standalone) | Disk format |
| `mkfs`, `diskpart` | Disk operations |
| `dd if=` | Raw disk write |
| `> /dev/sd` | Direct disk write |
| `shutdown`, `reboot`, `poweroff` | System power |
| Fork bomb pattern | Fork bomb |

### Workspace Restriction

When `restrict_to_workspace` is enabled:

```mermaid
flowchart TD
    A["Extract absolute paths from command"] --> B["Windows paths: C colon backslash ..."]
    A --> C["POSIX paths: /absolute/..."]
    B & C --> D["For each path"]
    D --> E{Path inside workspace?}
    E -- No --> F["Block command"]
    E -- Yes --> D
    D -- all pass --> G["Allow command"]
```

## Output Handling

| Output Component | Condition | Format |
|-----------------|-----------|--------|
| stdout | Always (if non-empty) | Raw text |
| stderr | If non-empty | Prefixed with `STDERR:\n` |
| Exit code | If non-zero | `\nExit code: N` |
| Truncation | If > 10,000 chars | `... (truncated, N more chars)` |
| No output | Empty stdout + stderr | `(no output)` |

## Configuration

The tool accepts configuration from `ExecToolConfig`:

| Config | Default | Description |
|--------|---------|-------------|
| `timeout` | 60s | Max run time before kill |
| `path_append` | `""` | Additional PATH entries |
| `restrict_to_workspace` | `false` | Sandbox to workspace directory |
