# ContextBuilder — System Prompt Assembly

**Source:** `nanobot/agent/context.py`

## Purpose

Assembles the complete LLM message list from layers: identity, bootstrap files, memory, skills, runtime metadata, and the user's message. This is the "brain builder" that determines what the agent knows and can do.

## Class Overview

```mermaid
classDiagram
    class ContextBuilder {
        +workspace: Path
        +memory: MemoryStore
        +skills: SkillsLoader
        +BOOTSTRAP_FILES: list

        +build_system_prompt(skill_names) → str
        +build_messages(history, message, media, channel, chat_id) → list
        +add_tool_result(messages, id, name, result) → list
        +add_assistant_message(messages, content, tool_calls) → list
        -_get_identity() → str
        -_load_bootstrap_files() → str
        -_build_runtime_context(channel, chat_id) → str
        -_build_user_content(text, media) → str|list
    }
```

## System Prompt Assembly

```mermaid
flowchart TD
    A["build_system_prompt()"] --> B["_get_identity()"]
    B --> C["_load_bootstrap_files()"]
    C --> D["memory.get_memory_context()"]
    D --> E["skills.get_always_skills()"]
    E --> F["skills.load_skills_for_context()"]
    F --> G["skills.build_skills_summary()"]
    G --> H["Join all parts with ---"]

    style B fill:#e8f5e9
    style C fill:#e1f5fe
    style D fill:#fff3e0
    style E fill:#f3e5f5
    style G fill:#fce4ec
```

### Layer Details

```mermaid
flowchart LR
    subgraph "System Prompt Layers"
        direction TB
        L1["1. Core Identity<br/>Runtime info, workspace path,<br/>guidelines"]
        L2["2. Bootstrap Files<br/>AGENTS.md, SOUL.md, USER.md,<br/>TOOLS.md, IDENTITY.md"]
        L3["3. Long-term Memory<br/>MEMORY.md content"]
        L4["4. Always-on Skills<br/>Skills marked always=true"]
        L5["5. Skills Summary<br/>XML catalog of all available skills"]
    end

    L1 --> L2 --> L3 --> L4 --> L5
```

| Layer | Source | When Included |
|-------|--------|---------------|
| Core Identity | Hardcoded in `_get_identity()` | Always |
| Bootstrap Files | `workspace/{AGENTS,SOUL,USER,TOOLS,IDENTITY}.md` | If file exists |
| Memory | `workspace/memory/MEMORY.md` | If non-empty |
| Always-on Skills | Skills with `always: true` in frontmatter | If any exist and requirements met |
| Skills Summary | All skills (XML catalog) | If any skills exist |

### Identity Section Content

The identity section includes:
- Agent name and persona ("nanobot")
- Runtime info: OS, architecture, Python version
- Workspace path and key file locations
- Behavioral guidelines (state intent, read before modify, etc.)

---

## Full Message List Construction

```mermaid
flowchart TD
    A["build_messages(history, message, media, channel, chat_id)"]
    A --> B["{ role: system, content: build_system_prompt() }"]
    A --> C["...history messages (from session)"]
    A --> D["{ role: user, content: _build_runtime_context() }"]
    A --> E["{ role: user, content: _build_user_content() }"]

    B --> RESULT["Final message list"]
    C --> RESULT
    D --> RESULT
    E --> RESULT
```

### Message list structure:

```
[
  { role: "system",    content: "<assembled system prompt>" },
  ...history[],        // unconsolidated session messages
  { role: "user",      content: "[Runtime Context] time, channel, chat_id" },
  { role: "user",      content: "<user message or multimodal content>" },
]
```

### Runtime Context Block

```mermaid
flowchart TD
    A["_build_runtime_context(channel, chat_id)"] --> B["Tag: [Runtime Context — metadata only]"]
    B --> C["Current Time: 2026-03-02 14:30 (Sunday) (PST)"]
    C --> D["Channel: telegram"]
    D --> E["Chat ID: 123456"]
```

This is injected as a separate user message just before the actual user input. The tag `_RUNTIME_CONTEXT_TAG` is used by `_save_turn()` to filter it out of session persistence.

---

## Media Handling

```mermaid
flowchart TD
    A["_build_user_content(text, media)"] --> B{media provided?}
    B -- No --> C["Return text string"]
    B -- Yes --> D["For each media path"]
    D --> E{Is image file?}
    E -- No --> D
    E -- Yes --> F["Base64 encode"]
    F --> G["Add image_url content block"]
    G --> D
    D -- done --> H{Any images?}
    H -- No --> C
    H -- Yes --> I["Return [images..., {text}]"]
```

When media is attached, the user content becomes a multimodal array:
```json
[
  { "type": "image_url", "image_url": { "url": "data:image/png;base64,..." } },
  { "type": "text", "text": "user message" }
]
```

---

## Message Mutation Helpers

These methods are used by `_run_agent_loop()` to incrementally build the conversation:

### `add_assistant_message()`

```mermaid
flowchart LR
    A[Input: content, tool_calls,<br/>reasoning_content, thinking_blocks]
    A --> B["{ role: assistant,<br/>content, tool_calls,<br/>reasoning_content,<br/>thinking_blocks }"]
    B --> C[Append to messages]
```

Supports extended thinking (`reasoning_content` for DeepSeek, `thinking_blocks` for Claude).

### `add_tool_result()`

```mermaid
flowchart LR
    A["Input: tool_call_id, name, result"]
    A --> B["{ role: tool,<br/>tool_call_id, name,<br/>content: result }"]
    B --> C[Append to messages]
```
