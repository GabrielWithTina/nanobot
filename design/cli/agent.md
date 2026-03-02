# `nanobot agent` — Chat Command

**Source:** `nanobot/cli/commands.py:419-587`

## Purpose

The primary user-facing command. Operates in two modes:
- **Single-shot** (`-m "message"`): send one message, print the response, exit.
- **Interactive** (no `-m`): REPL-style chat session with history navigation and paste support.

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-m`, `--message` | `None` | Message text (single-shot mode if provided) |
| `-s`, `--session` | `cli:direct` | Session key (`channel:chat_id`) |
| `--markdown/--no-markdown` | `True` | Render response as Rich Markdown |
| `--logs/--no-logs` | `False` | Show internal loguru logs |

---

## Single-Shot Mode

```mermaid
flowchart TD
    A["nanobot agent -m 'Hello'"] --> B[load_config]
    B --> C[sync_workspace_templates]
    C --> D[_make_provider]
    D --> E[Create AgentLoop]

    E --> F["agent.process_direct(message, session_id)"]
    F --> G["Build context<br/>(system prompt + history)"]
    G --> H["Call LLM"]
    H --> I{Tool calls?}
    I -- Yes --> J[Execute tools]
    J --> H
    I -- No --> K[Return response text]

    K --> L[_print_agent_response]
    L --> M[close_mcp]
    M --> N[Exit]

    style F fill:#e1f5fe
    style H fill:#fff3e0
```

### Key Behavior

- `process_direct()` bypasses the MessageBus entirely — direct function call → LLM → response string.
- A spinner (`nanobot is thinking...`) displays while waiting (unless `--logs` is on).
- The agent may loop up to `max_iterations` (default 40) tool-call rounds before returning.

---

## Interactive Mode

```mermaid
flowchart TD
    A[nanobot agent] --> B[load_config + setup]
    B --> C[_init_prompt_session]
    C --> D["Print: Interactive mode"]

    D --> E["Start agent_loop.run() — bus consumer task"]
    E --> F["Start _consume_outbound() — response listener"]

    F --> G[REPL Loop]

    G --> H[_flush_pending_tty_input]
    H --> I["_read_interactive_input_async()"]
    I --> J{exit command?}

    J -- Yes --> K[_restore_terminal]
    K --> L[Cleanup & Exit]

    J -- No --> M["bus.publish_inbound(InboundMessage)"]
    M --> N["Show spinner"]
    N --> O["Wait for turn_done event"]
    O --> P[_print_agent_response]
    P --> G

    style E fill:#e8f5e9
    style M fill:#e1f5fe
    style O fill:#fff3e0
```

### Message Flow (Interactive)

```mermaid
sequenceDiagram
    participant User
    participant REPL as CLI REPL
    participant Bus as MessageBus
    participant Agent as AgentLoop.run()
    participant LLM

    User->>REPL: types message
    REPL->>Bus: publish_inbound(InboundMessage)
    Bus->>Agent: consume_inbound()
    Agent->>Agent: _process_message()
    Agent->>LLM: call LLM
    LLM-->>Agent: response (possibly with tool calls)

    loop Tool iterations
        Agent->>Agent: execute tool
        Agent->>LLM: call LLM with tool result
        LLM-->>Agent: next response
    end

    Agent->>Bus: publish_outbound(OutboundMessage)
    Bus-->>REPL: consume_outbound()
    REPL->>REPL: turn_done.set()
    REPL->>User: _print_agent_response()
```

### Interactive Mode Details

- **Input handling:** `prompt_toolkit` provides line editing, up/down history navigation, and bracketed paste mode.
- **History persistence:** Saved to `~/.nanobot/history/cli_history`.
- **Exit commands:** `exit`, `quit`, `/exit`, `/quit`, `:q`, `Ctrl+C`, `EOF`.
- **Progress updates:** Outbound messages with `_progress` metadata are rendered inline as dim `↳ ...` hints (subject to `send_progress`/`send_tool_hints` channel config).
- **Terminal safety:** Original termios attributes are saved on init and restored on every exit path (including SIGINT).

---

## Provider Selection (`_make_provider`)

```mermaid
flowchart TD
    A[_make_provider] --> B{provider_name?}

    B -- "openai_codex" --> C[OpenAICodexProvider<br/>OAuth-based]
    B -- "custom" --> D[CustomProvider<br/>OpenAI-compatible endpoint]
    B -- "other" --> E{Has API key?}

    E -- Yes --> F[LiteLLMProvider]
    E -- "No + not OAuth + not bedrock" --> G[Error: No API key]

    style C fill:#e8f5e9
    style D fill:#fff3e0
    style F fill:#e1f5fe
    style G fill:#ffcdd2
```

Three provider types are supported:
1. **OpenAI Codex** — OAuth token-based, via `oauth_cli_kit`.
2. **Custom** — Direct OpenAI-compatible API (bypasses LiteLLM routing).
3. **LiteLLM** — All other providers, routed through LiteLLM's unified interface.
