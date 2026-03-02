# AgentLoop — Core Processing Engine

**Source:** `nanobot/agent/loop.py`

## Purpose

The `AgentLoop` is the central brain of nanobot. It receives messages, builds LLM context, calls the model, executes tool calls in a loop, and returns responses. It operates in two modes: bus-driven (gateway) and direct-call (CLI/cron).

## Class Overview

```mermaid
classDiagram
    class AgentLoop {
        +bus: MessageBus
        +provider: LLMProvider
        +workspace: Path
        +model: str
        +max_iterations: int
        +temperature: float
        +max_tokens: int
        +memory_window: int
        +context: ContextBuilder
        +sessions: SessionManager
        +tools: ToolRegistry
        +subagents: SubagentManager

        +run() → None
        +stop() → None
        +process_direct(content, session_key) → str
        +close_mcp() → None
        -_process_message(msg) → OutboundMessage
        -_run_agent_loop(messages) → (content, tools, msgs)
        -_dispatch(msg) → None
        -_handle_stop(msg) → None
        -_save_turn(session, messages, skip)
        -_consolidate_memory(session)
        -_connect_mcp()
        -_register_default_tools()
        -_set_tool_context(channel, chat_id)
    }
```

## Two Entry Points

```mermaid
flowchart LR
    subgraph "Bus-driven (gateway)"
        BUS[MessageBus] -->|consume_inbound| RUN["run()"]
        RUN --> DISPATCH["_dispatch()"]
        DISPATCH --> PM["_process_message()"]
        PM --> PUB[publish_outbound]
    end

    subgraph "Direct-call (CLI/cron)"
        CALLER[CLI / Cron] --> PD["process_direct()"]
        PD --> PM2["_process_message()"]
        PM2 --> RET[return string]
    end
```

### `run()` — Bus Consumer (lines 259-276)

Long-running loop for gateway mode. Consumes `InboundMessage` from the bus and dispatches each as an asyncio task.

```mermaid
flowchart TD
    A["run()"] --> B[_connect_mcp]
    B --> C{"_running?"}
    C -- No --> EXIT[Return]
    C -- Yes --> D["bus.consume_inbound(timeout=1s)"]
    D -- timeout --> C
    D -- message --> E{"/stop"?}
    E -- Yes --> F["_handle_stop()"]
    F --> C
    E -- No --> G["asyncio.create_task(_dispatch(msg))"]
    G --> C
```

Key design: each message is dispatched as an independent asyncio task, tracked in `_active_tasks[session_key]`, so `/stop` can cancel in-flight work.

### `process_direct()` — Direct Call (lines 486-498)

Synchronous-style entry for CLI and cron. Bypasses the bus entirely.

```mermaid
flowchart TD
    A["process_direct(content, session_key)"] --> B[_connect_mcp]
    B --> C["Wrap as InboundMessage"]
    C --> D["_process_message(msg, session_key)"]
    D --> E["Return response.content"]
```

---

## Core Processing Pipeline

### `_process_message()` (lines 330-453)

The heart of the agent. Handles all message types, slash commands, memory consolidation, context building, and the LLM iteration loop.

```mermaid
flowchart TD
    A["_process_message(msg)"] --> B{Is system channel?}

    B -- Yes --> SYS["Parse origin from chat_id<br/>Build context with history<br/>Run agent loop<br/>Save turn<br/>Return OutboundMessage"]

    B -- No --> C["Resolve session key"]
    C --> D{Slash command?}

    D -- "/new" --> NEW["Archive memory (consolidate_all)<br/>Clear session<br/>Return 'New session started'"]

    D -- "/help" --> HELP["Return help text"]

    D -- normal msg --> E{"Unconsolidated >= memory_window?"}
    E -- Yes --> BG["Fire background consolidation task"]
    E -- No --> F

    BG --> F["Set tool context (channel, chat_id)"]
    F --> G["MessageTool.start_turn()"]
    G --> H["Get session history"]
    H --> I["context.build_messages(history, msg)"]
    I --> J["_run_agent_loop(messages)"]
    J --> K["_save_turn(session, messages)"]
    K --> L["sessions.save(session)"]

    L --> M{"MessageTool sent during turn?"}
    M -- Yes --> N["Return None<br/>(already delivered)"]
    M -- No --> O["Return OutboundMessage"]
```

### `_run_agent_loop()` (lines 180-257)

The LLM iteration cycle. Calls the model, executes any tool calls, and loops until the model returns a final text response or hits `max_iterations`.

```mermaid
flowchart TD
    A["_run_agent_loop(messages)"] --> B["iteration = 0"]
    B --> C{"iteration < max_iterations?"}

    C -- No --> MAX["Return max iterations warning"]
    C -- Yes --> D["iteration++"]
    D --> E["provider.chat(messages, tools)"]

    E --> F{has_tool_calls?}

    F -- Yes --> G["Report progress (content + tool hint)"]
    G --> H["Add assistant message with tool_calls"]
    H --> I["For each tool_call"]
    I --> J["tools.execute(name, args)"]
    J --> K["Add tool result to messages"]
    K --> C

    F -- No --> L{is finish_reason equal to error?}
    L -- Yes --> ERR["Log error<br/>Return error message<br/>(don't persist to session)"]
    L -- No --> M["Add assistant message"]
    M --> N["Return (content, tools_used, messages)"]

    style ERR fill:#ffcdd2
```

---

## Session Lifecycle

```mermaid
sequenceDiagram
    participant PM as _process_message
    participant SM as SessionManager
    participant CTX as ContextBuilder
    participant MEM as MemoryStore

    PM->>SM: get_or_create(session_key)
    SM-->>PM: Session (with messages[])

    Note over PM: Check if consolidation needed
    opt unconsolidated >= memory_window
        PM->>MEM: consolidate(session) [background task]
    end

    PM->>SM: session.get_history(max_messages)
    SM-->>PM: history (unconsolidated only)
    PM->>CTX: build_messages(history, current_msg)
    CTX-->>PM: message list

    Note over PM: Run agent loop (LLM + tools)

    PM->>PM: _save_turn(session, new_messages)
    PM->>SM: save(session)
```

## `/stop` Handler

```mermaid
flowchart TD
    A["/stop received"] --> B["Pop _active_tasks[session_key]"]
    B --> C["Cancel all tasks"]
    C --> D["await each task"]
    D --> E["subagents.cancel_by_session()"]
    E --> F["Publish: Stopped N task(s)"]
```

## `/new` Handler

```mermaid
flowchart TD
    A["/new received"] --> B["Acquire consolidation lock"]
    B --> C["Snapshot unconsolidated messages"]
    C --> D["consolidate_memory(archive_all=true)"]
    D --> E{Success?}
    E -- No --> F["Return: archival failed"]
    E -- Yes --> G["session.clear()"]
    G --> H["sessions.save + invalidate"]
    H --> I["Return: New session started"]
```

## Concurrency Model

```mermaid
flowchart LR
    subgraph "Per AgentLoop"
        LOCK["_processing_lock<br/>(asyncio.Lock)"]
        TASKS["_active_tasks<br/>{session → [Task]}"]
        CONSOL["_consolidation_tasks<br/>{Task set}"]
        CLOCKS["_consolidation_locks<br/>{session → Lock}"]
    end

    MSG1["Message 1"] --> DISPATCH1["_dispatch()"]
    MSG2["Message 2"] --> DISPATCH2["_dispatch()"]

    DISPATCH1 --> LOCK
    DISPATCH2 --> LOCK

    Note1["Messages are dispatched as tasks<br/>but serialized by _processing_lock"]
```

- **`_processing_lock`**: Global lock ensuring one message is processed at a time (prevents race conditions on session state).
- **`_active_tasks`**: Per-session task tracking, enabling `/stop` to cancel in-flight work.
- **`_consolidation_locks`**: Per-session locks preventing concurrent consolidation of the same session.
- **`_consolidation_tasks`**: Strong references to keep consolidation tasks alive.

## `_save_turn()` Details (lines 455-477)

Persists new messages to the session, with these transformations:

| Condition | Action |
|-----------|--------|
| Empty assistant message (no content, no tool_calls) | Skip — prevents poisoned context |
| Tool result > 500 chars | Truncate with `... (truncated)` |
| User message starting with runtime context tag | Skip — metadata only |
| Image content (base64 data URI) | Replace with `[image]` placeholder |
| All messages | Add `timestamp` if missing |

## Tool Registration

```mermaid
flowchart TD
    A["_register_default_tools()"] --> B["ReadFileTool"]
    A --> C["WriteFileTool"]
    A --> D["EditFileTool"]
    A --> E["ListDirTool"]
    A --> F["ExecTool"]
    A --> G["WebSearchTool"]
    A --> H["WebFetchTool"]
    A --> I["MessageTool"]
    A --> J["SpawnTool"]
    A --> K{"cron_service?"}
    K -- Yes --> L["CronTool"]

    M["_connect_mcp()"] --> N{"MCP servers configured?"}
    N -- Yes --> O["MCPToolWrapper × N"]
```

## MCP Connection (Lazy, One-Time)

```mermaid
flowchart TD
    A["_connect_mcp()"] --> B{"Already connected<br/>or connecting?"}
    B -- Yes --> C[Return immediately]
    B -- No --> D["Set _mcp_connecting = true"]
    D --> E["Create AsyncExitStack"]
    E --> F["connect_mcp_servers()"]
    F --> G{Success?}
    G -- Yes --> H["_mcp_connected = true"]
    G -- No --> I["Log error, cleanup stack<br/>(will retry next message)"]
```
