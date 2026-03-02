# MCP Integration — Model Context Protocol

**Source:** `nanobot/agent/tools/mcp.py`

## Purpose

Connects to external MCP (Model Context Protocol) servers and wraps their tools as native nanobot tools. This enables the agent to use arbitrary external capabilities (databases, APIs, custom services) without code changes.

## Architecture

```mermaid
flowchart TD
    subgraph "Config"
        CFG["config.json<br/>tools.mcp_servers"]
    end

    subgraph "Connection (one-time)"
        CONN["connect_mcp_servers()"]
        STACK["AsyncExitStack<br/>(manages lifetimes)"]
    end

    subgraph "MCP Servers"
        S1["stdio server<br/>(subprocess)"]
        S2["HTTP server<br/>(streamable_http)"]
    end

    subgraph "Tool Wrappers"
        W1["MCPToolWrapper<br/>mcp_server1_tool_a"]
        W2["MCPToolWrapper<br/>mcp_server1_tool_b"]
        W3["MCPToolWrapper<br/>mcp_server2_tool_x"]
    end

    CFG --> CONN
    CONN --> S1 & S2
    CONN --> STACK
    S1 --> W1 & W2
    S2 --> W3
    W1 & W2 & W3 --> REG["ToolRegistry"]
```

## Connection Flow

```mermaid
flowchart TD
    A["connect_mcp_servers(mcp_servers, registry, stack)"] --> B["For each server config"]
    B --> C{Transport type?}

    C -- "command (stdio)" --> D["StdioServerParameters(command, args, env)"]
    D --> E["stack.enter_async_context(stdio_client)"]

    C -- "url (HTTP)" --> F["Create httpx.AsyncClient<br/>(custom headers, no timeout)"]
    F --> G["stack.enter_async_context(streamable_http_client)"]

    C -- "neither" --> H["Log warning, skip"]

    E & G --> I["ClientSession(read, write)"]
    I --> J["session.initialize()"]
    J --> K["session.list_tools()"]
    K --> L["For each tool"]
    L --> M["MCPToolWrapper(session, server_name, tool_def)"]
    M --> N["registry.register(wrapper)"]
    N --> L

    L -- done --> O["Log: N tools registered"]
```

## MCPToolWrapper

```mermaid
classDiagram
    class MCPToolWrapper {
        -_session: ClientSession
        -_original_name: str
        -_name: str
        -_description: str
        -_parameters: dict
        -_tool_timeout: int

        +name → str
        +description → str
        +parameters → dict
        +execute(**kwargs) → str
    }
```

### Naming Convention

MCP tools are namespaced to avoid collisions:

```
mcp_{server_name}_{tool_name}
```

Example: A server named `"github"` with a tool `"create_issue"` becomes `mcp_github_create_issue`.

### Execution

```mermaid
flowchart TD
    A["execute(**kwargs)"] --> B["session.call_tool(original_name, arguments=kwargs)"]
    B --> C["wait_for(timeout=tool_timeout)"]
    C -- timeout --> D["Return: tool timed out"]
    C -- success --> E["Extract content blocks"]
    E --> F["Join TextContent as text"]
    F --> G{Any output?}
    G -- Yes --> H["Return joined text"]
    G -- No --> I["Return '(no output)'"]
```

## Transport Types

### stdio (subprocess)

```mermaid
flowchart LR
    AGENT["nanobot"] -->|stdin/stdout| PROC["MCP Server Process"]
```

Config:
```json
{
  "my_server": {
    "command": "python",
    "args": ["-m", "my_mcp_server"],
    "env": { "API_KEY": "..." }
  }
}
```

### HTTP (streamable)

```mermaid
flowchart LR
    AGENT["nanobot"] -->|HTTP POST/SSE| SERVER["MCP HTTP Server"]
```

Config:
```json
{
  "my_server": {
    "url": "http://localhost:3000/mcp",
    "headers": { "Authorization": "Bearer ..." }
  }
}
```

The HTTP client is configured with:
- Custom headers from config
- `follow_redirects=True`
- `timeout=None` (deferred to per-tool `tool_timeout`)

## Lifecycle Management

```mermaid
flowchart TD
    A["AgentLoop.__init__"] --> B["_mcp_connected = False"]
    B --> C["First message arrives"]
    C --> D["_connect_mcp() — lazy, one-time"]
    D --> E["AsyncExitStack manages all connections"]

    F["AgentLoop.close_mcp()"] --> G["stack.aclose()"]
    G --> H["All MCP sessions + subprocesses cleaned up"]
```

The `AsyncExitStack` ensures proper cleanup of all MCP connections, stdio subprocesses, and HTTP clients when the agent shuts down.

## Error Handling

| Error | Behavior |
|-------|----------|
| Server connection failure | Logged, other servers still connect |
| Tool call timeout | Returns timeout message (doesn't crash) |
| Server crash mid-session | Tool calls fail with error (requires restart) |
| No command or URL in config | Warning logged, server skipped |
