# `nanobot gateway` — Multi-Channel Server

**Source:** `nanobot/cli/commands.py:244-409`

## Purpose

The long-running server mode. Starts the agent loop, all enabled channels, cron scheduler, and heartbeat service. This is the production deployment command.

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-p`, `--port` | `18790` | Gateway port |
| `-v`, `--verbose` | `False` | Enable debug logging |

---

## Startup Sequence

```mermaid
flowchart TD
    A[nanobot gateway] --> B[load_config]
    B --> C[sync_workspace_templates]
    C --> D[Create MessageBus]
    D --> E[_make_provider]
    E --> F[Create SessionManager]
    F --> G[Create CronService]
    G --> H[Create AgentLoop]
    H --> I[Wire cron callback → agent.process_direct]
    I --> J[Create ChannelManager]
    J --> K[Create HeartbeatService]
    K --> L[Print status]
    L --> M["asyncio.run(run())"]

    style M fill:#e8f5e9
```

## Runtime Architecture

```mermaid
flowchart LR
    subgraph Channels
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        WA[WhatsApp]
        EM[Email]
        OT[Other...]
    end

    subgraph Bus["MessageBus"]
        IQ["Inbound Queue"]
        OQ["Outbound Queue"]
    end

    subgraph Agent["AgentLoop.run()"]
        PROC["_process_message()"]
        LLM["LLM Provider"]
        TOOLS["Tool Execution"]
    end

    subgraph Services
        CRON["CronService"]
        HB["HeartbeatService"]
    end

    TG & DC & SL & WA & EM & OT -->|publish_inbound| IQ
    IQ -->|consume_inbound| PROC
    PROC <-->|iterate| LLM
    PROC <-->|tool calls| TOOLS
    PROC -->|publish_outbound| OQ
    OQ -->|consume_outbound| TG & DC & SL & WA & EM & OT

    CRON -->|"process_direct()"| PROC
    HB -->|"process_direct()"| PROC
    HB -.->|on_notify| OQ
```

## Concurrent Tasks (`run()`)

```mermaid
flowchart TD
    RUN["async run()"] --> START_CRON["cron.start()"]
    START_CRON --> START_HB["heartbeat.start()"]
    START_HB --> GATHER["asyncio.gather()"]

    GATHER --> AGENT["agent.run()<br/>— consumes inbound bus"]
    GATHER --> CHANNELS["channels.start_all()<br/>— starts all channel adapters<br/>— runs outbound dispatcher"]

    subgraph Background
        CRON_TICK["CronService timer loop"]
        HB_TICK["HeartbeatService interval loop"]
    end

    START_CRON -.-> CRON_TICK
    START_HB -.-> HB_TICK
```

## Cron Job Execution Flow

```mermaid
sequenceDiagram
    participant Timer as CronService Timer
    participant Callback as on_cron_job()
    participant Agent as AgentLoop
    participant LLM
    participant Bus as MessageBus
    participant Channel

    Timer->>Callback: job triggered
    Callback->>Agent: process_direct(reminder_note)
    Agent->>LLM: call LLM with task
    LLM-->>Agent: response

    alt MessageTool sent during turn
        Note right of Agent: Agent already delivered<br/>via MessageTool — skip
    else deliver=true and has recipient
        Callback->>Bus: publish_outbound(response)
        Bus->>Channel: send(response)
    end
```

## Heartbeat Execution Flow

```mermaid
sequenceDiagram
    participant HB as HeartbeatService
    participant LLM as LLM (decide)
    participant Agent as AgentLoop
    participant Bus as MessageBus

    loop Every interval_s seconds
        HB->>HB: Read HEARTBEAT.md
        alt File empty or missing
            Note right of HB: Skip — nothing to do
        else Has content
            HB->>LLM: "Should I act?" (decide phase)
            LLM-->>HB: heartbeat(action, tasks)
            alt action = "skip"
                Note right of HB: No action needed
            else action = "run"
                HB->>Agent: on_execute(tasks)
                Agent-->>HB: response
                HB->>Bus: on_notify(response) → outbound
            end
        end
    end
```

## Shutdown Sequence

```mermaid
flowchart TD
    A[KeyboardInterrupt or error] --> B[agent.close_mcp]
    B --> C[heartbeat.stop]
    C --> D[cron.stop]
    D --> E[agent.stop]
    E --> F[channels.stop_all]
    F --> G[Process exits]
```

## Heartbeat Target Selection

`_pick_heartbeat_target()` determines where heartbeat responses are delivered:

1. Scans `SessionManager.list_sessions()` (most recently updated first).
2. Picks the first session on an enabled, non-internal channel (`cli` and `system` are excluded).
3. Falls back to `("cli", "direct")` if no external channel session exists.
