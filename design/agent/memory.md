# MemoryStore — Two-Layer Persistent Memory

**Source:** `nanobot/agent/memory.py`

## Purpose

Implements nanobot's persistent memory system with two complementary layers:
- **MEMORY.md** — structured long-term facts (updated by LLM consolidation)
- **HISTORY.md** — append-only, grep-searchable event log

## Class Overview

```mermaid
classDiagram
    class MemoryStore {
        +memory_dir: Path
        +memory_file: Path
        +history_file: Path

        +read_long_term() → str
        +write_long_term(content)
        +append_history(entry)
        +get_memory_context() → str
        +consolidate(session, provider, model) → bool
    }
```

## File Layout

```
workspace/memory/
├── MEMORY.md     # Long-term facts (rewritten on consolidation)
└── HISTORY.md    # Event log (append-only, timestamped)
```

## Two-Layer Design

```mermaid
flowchart LR
    subgraph "Layer 1: MEMORY.md"
        M["Structured facts<br/>(full rewrite on update)"]
    end

    subgraph "Layer 2: HISTORY.md"
        H["Chronological log<br/>(append-only)"]
    end

    AGENT["Agent (via system prompt)"] --> M
    SEARCH["User / Agent grep"] --> H
    CONSOL["consolidate()"] --> M & H
```

| Property | MEMORY.md | HISTORY.md |
|----------|-----------|------------|
| Format | Markdown (any structure) | `[YYYY-MM-DD HH:MM]` entries |
| Update mode | Full rewrite | Append-only |
| Purpose | Facts the agent always sees | Searchable log of past events |
| Included in prompt | Yes (via `get_memory_context()`) | No (too large; grep instead) |
| Updated by | LLM `save_memory` tool call | LLM `save_memory` tool call |

---

## Consolidation Flow

Consolidation is triggered when unconsolidated messages exceed `memory_window`. The LLM summarizes old conversation into both memory layers.

```mermaid
flowchart TD
    A["consolidate(session, provider, model)"] --> B{archive_all?}

    B -- Yes --> C["old_messages = all session messages<br/>keep_count = 0"]
    B -- No --> D["keep_count = memory_window / 2<br/>old_messages = messages[last_consolidated : -keep_count]"]

    C & D --> E{Any old messages?}
    E -- No --> NOOP["Return True (no-op)"]
    E -- Yes --> F["Format messages as timestamped text"]

    F --> G["Read current MEMORY.md"]
    G --> H["Build consolidation prompt"]

    H --> I["provider.chat(<br/>system: 'You are a memory consolidation agent',<br/>user: prompt,<br/>tools: [save_memory]<br/>)"]

    I --> J{LLM called save_memory?}
    J -- No --> FAIL["Log warning, return False"]
    J -- Yes --> K["Parse tool arguments"]

    K --> L["append_history(history_entry)"]
    K --> M{memory_update != current?}
    M -- Yes --> N["write_long_term(memory_update)"]
    M -- No --> O["Skip (no change)"]

    L & N & O --> P["Update session.last_consolidated"]
    P --> Q["Return True"]

    style FAIL fill:#ffcdd2
```

### `save_memory` Virtual Tool

The consolidation LLM is given a single tool to structure its output:

```json
{
  "name": "save_memory",
  "parameters": {
    "history_entry": "Timestamped paragraph (2-5 sentences) for HISTORY.md",
    "memory_update": "Full updated MEMORY.md content"
  }
}
```

This forces the LLM to produce structured output rather than free text.

---

## Consolidation Trigger Points

```mermaid
sequenceDiagram
    participant PM as _process_message
    participant Session
    participant MEM as MemoryStore

    PM->>Session: Check unconsolidated count
    Note over PM: unconsolidated = len(messages) - last_consolidated

    alt unconsolidated >= memory_window
        PM->>MEM: consolidate(session) [background task]
        MEM->>MEM: LLM summarizes old messages
        MEM->>MEM: Write MEMORY.md + append HISTORY.md
        MEM->>Session: Update last_consolidated
    end

    alt "/new" command
        PM->>MEM: consolidate(session, archive_all=true)
        Note over MEM: Archives ALL messages before clearing
        PM->>Session: clear()
    end
```

### Consolidation Boundary Calculation

```mermaid
flowchart LR
    subgraph "Session Messages"
        A["msg 0"] --- B["msg 1"] --- C["..."] --- D["msg N-k"] --- E["..."] --- F["msg N"]
    end

    CONS["last_consolidated"] -.-> A
    KEEP["keep_count<br/>(window/2)"] -.-> D

    subgraph "Consolidate these"
        style A fill:#fff3e0
        style B fill:#fff3e0
        style C fill:#fff3e0
    end

    subgraph "Keep these"
        style D fill:#e8f5e9
        style E fill:#e8f5e9
        style F fill:#e8f5e9
    end
```

- **Normal consolidation**: Processes `messages[last_consolidated : -keep_count]`, keeps the most recent `keep_count` messages.
- **Archive all** (`/new`): Processes all messages, sets `last_consolidated = 0`.

## Read Path (used by ContextBuilder)

```mermaid
flowchart TD
    A["get_memory_context()"] --> B["read_long_term()"]
    B --> C{Content?}
    C -- empty --> D["Return empty string"]
    C -- has content --> E["Return '## Long-term Memory\\n{content}'"]
```

This is included in the system prompt on every LLM call, giving the agent persistent knowledge across sessions.
