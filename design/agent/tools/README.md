# Tool System Overview

**Source:** `nanobot/agent/tools/`

## Purpose

The tool system gives the agent the ability to interact with the environment — execute shell commands, read/write files, search the web, send messages, schedule tasks, and connect to external services via MCP.

## Architecture

```mermaid
flowchart TD
    subgraph "Tool Abstraction"
        BASE["Tool (ABC)<br/>base.py"]
        REG["ToolRegistry<br/>registry.py"]
    end

    subgraph "Built-in Tools"
        FS["ReadFileTool<br/>WriteFileTool<br/>EditFileTool<br/>ListDirTool"]
        EXEC["ExecTool"]
        WEB["WebSearchTool<br/>WebFetchTool"]
        MSG["MessageTool"]
        SPAWN["SpawnTool"]
        CRON["CronTool"]
    end

    subgraph "Dynamic Tools"
        MCP["MCPToolWrapper<br/>(per MCP server tool)"]
    end

    BASE --> FS & EXEC & WEB & MSG & SPAWN & CRON & MCP
    REG --> BASE

    LLM["LLM"] -->|"tool_call(name, args)"| REG
    REG -->|"execute(name, args)"| BASE
    BASE -->|"result string"| REG
    REG -->|"result"| LLM
```

## Tool Registration

```mermaid
sequenceDiagram
    participant Loop as AgentLoop.__init__
    participant Reg as ToolRegistry
    participant MCP as connect_mcp_servers

    Loop->>Reg: register(ReadFileTool)
    Loop->>Reg: register(WriteFileTool)
    Loop->>Reg: register(EditFileTool)
    Loop->>Reg: register(ListDirTool)
    Loop->>Reg: register(ExecTool)
    Loop->>Reg: register(WebSearchTool)
    Loop->>Reg: register(WebFetchTool)
    Loop->>Reg: register(MessageTool)
    Loop->>Reg: register(SpawnTool)
    opt cron_service provided
        Loop->>Reg: register(CronTool)
    end
    opt MCP servers configured
        MCP->>Reg: register(MCPToolWrapper) × N
    end
```

## Tool Inventory

| Tool | Name | Source | Capabilities |
|------|------|--------|-------------|
| [ReadFileTool](filesystem.md) | `read_file` | filesystem.py | Read file contents |
| [WriteFileTool](filesystem.md) | `write_file` | filesystem.py | Create/overwrite files |
| [EditFileTool](filesystem.md) | `edit_file` | filesystem.py | Find-and-replace in files |
| [ListDirTool](filesystem.md) | `list_dir` | filesystem.py | List directory contents |
| [ExecTool](shell.md) | `exec` | shell.py | Execute shell commands |
| [WebSearchTool](web.md) | `web_search` | web.py | Brave Search API |
| [WebFetchTool](web.md) | `web_fetch` | web.py | Fetch and extract web content |
| [MessageTool](message.md) | `message` | message.py | Send to chat channels |
| [SpawnTool](spawn.md) | `spawn` | spawn.py | Create background subagents |
| [CronTool](cron.md) | `cron` | cron.py | Schedule reminders/tasks |
| [MCPToolWrapper](mcp.md) | `mcp_{server}_{tool}` | mcp.py | External MCP server tools |

## Execution Flow

```mermaid
flowchart TD
    A["ToolRegistry.execute(name, params)"] --> B{Tool exists?}
    B -- No --> C["Error: Tool not found<br/>+ available tools list"]
    B -- Yes --> D["tool.validate_params(params)"]
    D --> E{Valid?}
    E -- No --> F["Error: Invalid parameters"]
    E -- Yes --> G["tool.execute(**params)"]
    G --> H{Result starts with 'Error'?}
    H -- Yes --> I["Append: [Analyze error, try different approach]"]
    H -- No --> J["Return result"]

    G -.->|exception| K["Error executing: {exception}"]
    K --> I
```

The error hint `[Analyze the error above and try a different approach.]` is automatically appended to all error results, nudging the LLM to self-correct.

## Documents

| File | Covers |
|------|--------|
| [registry.md](registry.md) | `ToolRegistry` — dynamic tool management |
| [filesystem.md](filesystem.md) | File system tools (read, write, edit, list_dir) |
| [shell.md](shell.md) | `ExecTool` — shell command execution |
| [web.md](web.md) | `WebSearchTool` + `WebFetchTool` |
| [message.md](message.md) | `MessageTool` — channel message delivery |
| [spawn.md](spawn.md) | `SpawnTool` — background subagent creation |
| [cron.md](cron.md) | `CronTool` — scheduling interface |
| [mcp.md](mcp.md) | MCP integration — Model Context Protocol |
