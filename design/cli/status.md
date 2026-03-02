# `nanobot status` — System Status

**Source:** `nanobot/cli/commands.py:995-1029`

## Purpose

Displays configuration and connectivity status at a glance: config file presence, workspace existence, active model, and API key status for all registered providers.

## Flow Diagram

```mermaid
flowchart TD
    A[nanobot status] --> B[get_config_path]
    B --> C[load_config]
    C --> D["Print: Config path + exists?"]
    D --> E["Print: Workspace path + exists?"]
    E --> F{config exists?}

    F -- No --> G[Done]
    F -- Yes --> H["Print: Active model name"]
    H --> I[Iterate PROVIDERS registry]

    I --> J{Provider type?}

    J -- OAuth --> K["Print: ✓ (OAuth)"]
    J -- Local --> L{api_base set?}
    L -- Yes --> M["Print: ✓ + api_base URL"]
    L -- No --> N["Print: not set"]
    J -- Standard --> O{api_key set?}
    O -- Yes --> P["Print: ✓"]
    O -- No --> Q["Print: not set"]

    K & M & N & P & Q --> R{More providers?}
    R -- Yes --> J
    R -- No --> G
```

## Example Output

```
🤖 nanobot Status

Config: ~/.nanobot/config.json ✓
Workspace: ~/.nanobot/workspace ✓
Model: openrouter/anthropic/claude-sonnet-4
OpenRouter: ✓
OpenAI: not set
Anthropic: not set
Google Gemini: not set
OpenAI Codex: ✓ (OAuth)
Ollama: not set
```

## Provider Status Logic

The status check uses the `PROVIDERS` registry as the single source of truth:

```mermaid
flowchart LR
    REG["PROVIDERS registry<br/>(list of ProviderSpec)"] --> LOOP["For each spec"]
    LOOP --> GET["getattr(config.providers, spec.name)"]
    GET --> CHECK{spec flags}

    CHECK -- "is_oauth" --> OAUTH["✓ (OAuth)"]
    CHECK -- "is_local" --> LOCAL{api_base?}
    LOCAL -- set --> SHOW_BASE["✓ + URL"]
    LOCAL -- empty --> DIM["not set"]
    CHECK -- "standard" --> KEY{api_key?}
    KEY -- set --> OK["✓"]
    KEY -- empty --> DIM2["not set"]
```

No network calls are made — this is purely a config inspection command.
