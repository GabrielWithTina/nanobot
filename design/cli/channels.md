# `nanobot channels` — Channel Management

**Source:** `nanobot/cli/commands.py:595-773`

## Subcommands

| Command | Description |
|---------|-------------|
| `nanobot channels status` | Display configuration status of all channels |
| `nanobot channels login` | Start WhatsApp bridge for QR-code device linking |

---

## `channels status`

**Source:** Lines 599-689

### Flow Diagram

```mermaid
flowchart TD
    A[nanobot channels status] --> B[load_config]
    B --> C[Build Rich Table]

    C --> D[WhatsApp]
    C --> E[Discord]
    C --> F[Feishu]
    C --> G[Mochat]
    C --> H[Telegram]
    C --> I[Slack]
    C --> J[DingTalk]
    C --> K[QQ]
    C --> L[Email]

    D & E & F & G & H & I & J & K & L --> M[Print table]
```

Each channel row shows:

| Column | Content |
|--------|---------|
| Channel | Platform name |
| Enabled | `✓` / `✗` |
| Configuration | Key config value (URL, token prefix, app_id prefix, etc.) |

### Channel Config Details

```mermaid
graph LR
    subgraph "Config Inspection"
        WA["WhatsApp → bridge_url"]
        DC["Discord → gateway_url"]
        FS["Feishu → app_id (truncated)"]
        MC["Mochat → base_url"]
        TG["Telegram → token (truncated)"]
        SL["Slack → socket mode check"]
        DT["DingTalk → client_id (truncated)"]
        QQ["QQ → app_id (truncated)"]
        EM["Email → imap_host"]
    end
```

---

## `channels login`

**Source:** Lines 750-773

### Purpose

Links a WhatsApp device by starting the bridge process and displaying a QR code for scanning.

### Flow Diagram

```mermaid
flowchart TD
    A[nanobot channels login] --> B[load_config]
    B --> C[_get_bridge_dir]
    C --> D{bridge built?}

    D -- Yes --> E[Use existing bridge]
    D -- No --> F[Build bridge]

    F --> G{npm available?}
    G -- No --> ERR1[Error: install Node.js]
    G -- Yes --> H[Find bridge source]

    H --> I{Source found?}
    I -- No --> ERR2[Error: bridge source missing]
    I -- Yes --> J[Copy to ~/.nanobot/bridge]
    J --> K[npm install]
    K --> L[npm run build]
    L --> E

    E --> M{bridge_token configured?}
    M -- Yes --> N[Set BRIDGE_TOKEN env var]
    M -- No --> O[Skip]

    N & O --> P["subprocess.run: npm start"]
    P --> Q[QR code displayed in terminal]
```

### Bridge Setup (`_get_bridge_dir`)

```mermaid
flowchart TD
    A[_get_bridge_dir] --> B{"~/.nanobot/bridge/dist/index.js exists?"}
    B -- Yes --> C[Return bridge path]
    B -- No --> D{npm installed?}
    D -- No --> E[Exit with error]
    D -- Yes --> F{Find source}

    F --> G["Check nanobot/bridge/<br/>(installed package)"]
    G --> H["Check repo-root/bridge/<br/>(dev mode)"]

    G -- found --> I[source = pkg_bridge]
    H -- found --> J[source = src_bridge]
    G & H -- not found --> K[Exit with error]

    I & J --> L["copytree → ~/.nanobot/bridge<br/>(excludes node_modules, dist)"]
    L --> M[npm install]
    M --> N[npm run build]
    N --> C
```
