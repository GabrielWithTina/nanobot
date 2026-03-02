# ToolRegistry — Dynamic Tool Management

**Source:** `nanobot/agent/tools/registry.py`

## Purpose

A simple dictionary-based registry that manages tool discovery, validation, and execution. Serves as the bridge between LLM tool calls and actual tool implementations.

## Class Overview

```mermaid
classDiagram
    class ToolRegistry {
        -_tools: dict[str, Tool]

        +register(tool: Tool)
        +unregister(name: str)
        +get(name: str) → Tool | None
        +has(name: str) → bool
        +get_definitions() → list[dict]
        +execute(name: str, params: dict) → str
        +tool_names → list[str]
    }
```

## Execution Pipeline

```mermaid
flowchart TD
    A["execute(name, params)"] --> B{name in _tools?}
    B -- No --> C["'Error: Tool not found.<br/>Available: read_file, exec, ...'"]
    C --> RET[Return error]

    B -- Yes --> D["tool = _tools[name]"]
    D --> E["errors = tool.validate_params(params)"]
    E --> F{errors?}
    F -- Yes --> G["'Error: Invalid parameters: ...'<br/>+ hint"]
    G --> RET

    F -- No --> H["result = await tool.execute(**params)"]
    H --> I{Exception?}
    I -- Yes --> J["'Error executing {name}: {e}'<br/>+ hint"]
    J --> RET

    I -- No --> K{result starts with 'Error'?}
    K -- Yes --> L["result + hint"]
    L --> RET
    K -- No --> M["Return result"]

    style C fill:#ffcdd2
    style G fill:#ffcdd2
    style J fill:#ffcdd2
    style L fill:#fff3e0
```

The hint `[Analyze the error above and try a different approach.]` is appended to all error paths.

## Schema Generation

`get_definitions()` returns all tools in OpenAI function-calling format:

```json
[
  {
    "type": "function",
    "function": {
      "name": "read_file",
      "description": "Read the contents of a file at the given path.",
      "parameters": {
        "type": "object",
        "properties": { "path": { "type": "string" } },
        "required": ["path"]
      }
    }
  }
]
```

This list is passed to `provider.chat(tools=...)` on every LLM call.
