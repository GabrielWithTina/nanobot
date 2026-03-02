# SubagentManager — Background Task Execution

**Source:** `nanobot/agent/subagent.py`

## Purpose

Manages background subagents — lightweight, independent agent instances that execute tasks concurrently without blocking the main conversation. Each subagent gets its own tool set (no message/spawn/cron) and runs up to 15 tool iterations.

## Class Overview

```mermaid
classDiagram
    class SubagentManager {
        +provider: LLMProvider
        +workspace: Path
        +bus: MessageBus
        +model: str
        -_running_tasks: dict[str, Task]
        -_session_tasks: dict[str, set[str]]

        +spawn(task, label, origin_channel, origin_chat_id) → str
        +cancel_by_session(session_key) → int
        +get_running_count() → int
        -_run_subagent(task_id, task, label, origin)
        -_announce_result(task_id, label, task, result, origin, status)
        -_build_subagent_prompt() → str
    }
```

## Spawn Flow

```mermaid
flowchart TD
    A["spawn(task, label)"] --> B["Generate task_id (uuid[:8])"]
    B --> C["Create asyncio.Task(_run_subagent)"]
    C --> D["Store in _running_tasks[task_id]"]
    D --> E["Track in _session_tasks[session_key]"]
    E --> F["Add cleanup callback"]
    F --> G["Return confirmation string"]
```

## Subagent Execution

```mermaid
flowchart TD
    A["_run_subagent(task_id, task, label, origin)"]
    A --> B["Build restricted ToolRegistry"]
    B --> C["Build subagent system prompt"]
    C --> D["messages = [system, user:task]"]
    D --> E["iteration = 0"]

    E --> F{"iteration < 15?"}
    F -- No --> MAX["final_result = 'no response'"]
    F -- Yes --> G["iteration++"]
    G --> H["provider.chat(messages, tools)"]

    H --> I{has_tool_calls?}
    I -- Yes --> J["Execute each tool"]
    J --> K["Append results to messages"]
    K --> F

    I -- No --> L["final_result = response.content"]

    L & MAX --> M["_announce_result(ok)"]

    A -.->|exception| N["_announce_result(error)"]

    style N fill:#ffcdd2
```

## Subagent vs Main Agent

```mermaid
flowchart LR
    subgraph "Main Agent"
        MA_TOOLS["All tools:<br/>read_file, write_file, edit_file,<br/>list_dir, exec, web_search,<br/>web_fetch, message, spawn, cron"]
        MA_ITER["40 max iterations"]
        MA_SESSION["Has session history"]
        MA_MEMORY["Has memory context"]
    end

    subgraph "Subagent"
        SA_TOOLS["Restricted tools:<br/>read_file, write_file, edit_file,<br/>list_dir, exec, web_search,<br/>web_fetch"]
        SA_ITER["15 max iterations"]
        SA_SESSION["No session history"]
        SA_MEMORY["No memory context"]
    end
```

Key restrictions for subagents:
- **No `message` tool** — cannot send messages directly to users
- **No `spawn` tool** — cannot spawn sub-subagents (prevents recursion)
- **No `cron` tool** — cannot schedule tasks
- **No session/memory** — starts with a clean slate each time
- **15 iterations** (vs 40 for main agent) — bounded execution

## Result Announcement

When a subagent completes, it announces back to the main agent via the message bus:

```mermaid
sequenceDiagram
    participant Sub as Subagent
    participant Bus as MessageBus
    participant Main as Main Agent (via run())
    participant LLM
    participant Channel

    Sub->>Sub: Task completes
    Sub->>Bus: publish_inbound(InboundMessage)<br/>channel="system"<br/>sender="subagent"<br/>chat_id="telegram:12345"

    Bus->>Main: consume_inbound()
    Note over Main: System messages route to<br/>_process_message with channel="system"

    Main->>LLM: Summarize subagent result
    LLM-->>Main: Natural language summary
    Main->>Bus: publish_outbound(OutboundMessage)
    Bus->>Channel: Deliver to user
```

### Announcement Content Format

```
[Subagent 'weather lookup' completed successfully]

Task: Check the weather in Vancouver

Result:
Currently 12°C and partly cloudy in Vancouver...

Summarize this naturally for the user. Keep it brief (1-2 sentences).
Do not mention technical details like "subagent" or task IDs.
```

The main agent processes this as a "system" channel message and produces a clean summary for the user.

## Subagent System Prompt

```mermaid
flowchart TD
    A["_build_subagent_prompt()"] --> B["Runtime context (time, etc.)"]
    B --> C["Role: focused subagent"]
    C --> D["Workspace path"]
    D --> E{Skills available?}
    E -- Yes --> F["Skills summary (XML)"]
    E -- No --> G[Done]
    F --> G
```

The subagent prompt is intentionally minimal:
- Runtime metadata (time)
- Role instruction ("you are a subagent, stay focused")
- Workspace path
- Skills summary (can `read_file` to load full skill)

No identity files, no memory, no conversation history.

## Cancellation

```mermaid
flowchart TD
    A["cancel_by_session(session_key)"] --> B["Find task IDs for session"]
    B --> C["For each running task"]
    C --> D["task.cancel()"]
    D --> C
    C -- done --> E["asyncio.gather(*tasks)"]
    E --> F["Return count cancelled"]
```

Cancellation is triggered by the `/stop` command in `AgentLoop._handle_stop()`.

## Task Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Running : spawn()
    Running --> Completed : task finishes
    Running --> Failed : exception
    Running --> Cancelled : cancel_by_session()
    Completed --> [*] : cleanup callback
    Failed --> [*] : cleanup callback
    Cancelled --> [*] : cleanup callback

    note right of Running
        Tracked in _running_tasks[task_id]
        and _session_tasks[session_key]
    end note

    note right of Completed
        _announce_result(status="ok")
    end note

    note right of Failed
        _announce_result(status="error")
    end note
```
