# SpawnTool — Background Subagent Creation

**Source:** `nanobot/agent/tools/spawn.py`

## Purpose

A thin adapter between the LLM's tool-calling interface and the `SubagentManager`. Allows the agent to delegate tasks to background subagents.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `task` | string | Yes | Task description for the subagent |
| `label` | string | No | Short display label |

## Execution Flow

```mermaid
flowchart TD
    A["spawn(task, label)"] --> B["Delegate to SubagentManager.spawn()"]
    B --> C["manager.spawn(<br/>task, label,<br/>origin_channel, origin_chat_id,<br/>session_key)"]
    C --> D["Return confirmation:<br/>'Subagent started (id: abc123)'"]
```

## Context Propagation

```mermaid
flowchart LR
    A["AgentLoop._set_tool_context()"] --> B["SpawnTool.set_context(channel, chat_id)"]
    B --> C["Stored as:<br/>_origin_channel<br/>_origin_chat_id<br/>_session_key"]
    C --> D["Passed to SubagentManager.spawn()"]
    D --> E["Subagent announces result<br/>back to this channel:chat_id"]
```

The origin context ensures subagent results are routed back to the correct conversation.

## Interaction Pattern

```mermaid
sequenceDiagram
    participant User
    participant Agent as Main Agent
    participant Spawn as SpawnTool
    participant Sub as SubagentManager
    participant BG as Background Subagent

    User->>Agent: "Research topic X and write a summary"
    Agent->>Spawn: spawn(task="Research X...", label="Research X")
    Spawn->>Sub: spawn(task, label, channel, chat_id)
    Sub-->>Spawn: "Subagent started (id: abc123)"
    Spawn-->>Agent: tool result
    Agent-->>User: "I've started a background task to research X."

    Note over BG: Running independently...
    BG->>BG: web_search, read_file, etc.
    BG->>Sub: Task complete
    Sub->>Agent: InboundMessage (channel="system")
    Agent->>User: "Here's what I found about X: ..."
```
