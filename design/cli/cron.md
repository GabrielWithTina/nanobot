# `nanobot cron` — Scheduled Task Management

**Source:** `nanobot/cli/commands.py:779-987`

## Subcommands

| Command | Description |
|---------|-------------|
| `nanobot cron list` | List scheduled jobs |
| `nanobot cron add` | Create a new job |
| `nanobot cron remove` | Delete a job |
| `nanobot cron enable` | Enable or disable a job |
| `nanobot cron run` | Manually trigger a job |

All commands operate on the persistent store at `~/.nanobot/cron/jobs.json`.

---

## `cron list`

**Source:** Lines 783-833

```mermaid
flowchart TD
    A["nanobot cron list [--all]"] --> B[Load CronService]
    B --> C["list_jobs(include_disabled)"]
    C --> D{Any jobs?}

    D -- No --> E["Print: No scheduled jobs"]
    D -- Yes --> F[Build Rich Table]

    F --> G[For each job]
    G --> H{schedule.kind?}

    H -- every --> I["Format: every Ns"]
    H -- cron --> J["Format: expr (tz)"]
    H -- at --> K["Format: one-time"]

    I & J & K --> L[Format next_run timestamp]
    L --> M[Add row: ID, Name, Schedule, Status, Next Run]
    M --> G

    G -- done --> N[Print table]
```

---

## `cron add`

**Source:** Lines 836-886

### Options

| Flag | Description |
|------|-------------|
| `-n`, `--name` | Job name (required) |
| `-m`, `--message` | Agent instruction (required) |
| `-e`, `--every` | Interval in seconds |
| `-c`, `--cron` | Cron expression |
| `--tz` | IANA timezone (cron only) |
| `--at` | ISO timestamp for one-time execution |
| `-d`, `--deliver` | Deliver response to a channel |
| `--to` | Recipient identifier |
| `--channel` | Target channel name |

### Flow Diagram

```mermaid
flowchart TD
    A[nanobot cron add] --> B{Schedule type?}

    B -- "--every N" --> C["CronSchedule(kind='every', every_ms=N*1000)"]
    B -- "--cron EXPR" --> D["CronSchedule(kind='cron', expr, tz)"]
    B -- "--at ISO" --> E["CronSchedule(kind='at', at_ms)"]
    B -- "none given" --> F[Error: must specify schedule]

    C & D & E --> G["service.add_job(name, schedule, message, ...)"]
    G --> H{Valid?}
    H -- Yes --> I["Print: ✓ Added job"]
    H -- No --> J[Error + exit]

    style F fill:#ffcdd2
    style J fill:#ffcdd2
```

### Validation

- `--tz` is only valid with `--cron` (rejected otherwise).
- Exactly one of `--every`, `--cron`, or `--at` must be provided.

---

## `cron remove`

**Source:** Lines 889-903

```mermaid
flowchart TD
    A["nanobot cron remove JOB_ID"] --> B["service.remove_job(job_id)"]
    B --> C{Found?}
    C -- Yes --> D["Print: ✓ Removed"]
    C -- No --> E["Print: not found"]
```

---

## `cron enable`

**Source:** Lines 906-923

```mermaid
flowchart TD
    A["nanobot cron enable JOB_ID [--disable]"] --> B["service.enable_job(job_id, enabled)"]
    B --> C{Found?}
    C -- Yes --> D["Print: ✓ enabled/disabled"]
    C -- No --> E["Print: not found"]
```

---

## `cron run`

**Source:** Lines 926-987

### Purpose

Manually triggers a job by spinning up a temporary `AgentLoop`, executing the job's message through `process_direct()`, and displaying the response.

### Flow Diagram

```mermaid
flowchart TD
    A["nanobot cron run JOB_ID [--force]"] --> B[load_config]
    B --> C[_make_provider]
    C --> D[Create AgentLoop]
    D --> E[Load CronService]
    E --> F[Set on_job callback]

    F --> G["service.run_job(job_id, force)"]
    G --> H{Job found & runnable?}

    H -- No --> I["Print: Failed to run"]
    H -- Yes --> J["on_job callback invoked"]

    J --> K["agent.process_direct(job.payload.message)"]
    K --> L[LLM processes message]
    L --> M[Response captured]
    M --> N["Print: ✓ Job executed"]
    N --> O[_print_agent_response]

    style I fill:#ffcdd2
```

### Sequence Diagram

```mermaid
sequenceDiagram
    participant CLI as cron run
    participant Svc as CronService
    participant CB as on_job callback
    participant Agent as AgentLoop
    participant LLM

    CLI->>Svc: run_job(job_id, force)
    Svc->>CB: on_job(job)
    CB->>Agent: process_direct(job.message)
    Agent->>LLM: call LLM
    LLM-->>Agent: response
    Agent-->>CB: response string
    CB-->>Svc: response
    Svc-->>CLI: success
    CLI->>CLI: print response
```

### Notes

- `--force` allows running disabled jobs.
- Unlike `gateway` mode, no bus or channels are involved — purely a direct agent call.
- The temporary `AgentLoop` is created without `cron_service` injected (the tool won't be available), since we're just executing one job's payload.
