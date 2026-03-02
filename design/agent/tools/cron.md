# CronTool — Scheduling Interface

**Source:** `nanobot/agent/tools/cron.py`

## Purpose

Allows the agent to manage scheduled jobs through natural language — creating reminders, listing scheduled tasks, and removing them. Delegates to `CronService` for persistence and execution.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | `"add"`, `"list"`, or `"remove"` |
| `message` | string | For add | Reminder/task instruction |
| `every_seconds` | integer | - | Recurring interval |
| `cron_expr` | string | - | Cron expression (e.g. `0 9 * * *`) |
| `tz` | string | - | IANA timezone (cron only) |
| `at` | string | - | ISO datetime for one-shot |
| `job_id` | string | For remove | Job ID to remove |

## Action Dispatch

```mermaid
flowchart TD
    A["cron(action, ...)"] --> B{action?}
    B -- add --> C["_add_job()"]
    B -- list --> D["_list_jobs()"]
    B -- remove --> E["_remove_job()"]
    B -- other --> F["Unknown action"]
```

---

## Add Job

```mermaid
flowchart TD
    A["_add_job(message, every_seconds, cron_expr, tz, at)"] --> B{message provided?}
    B -- No --> ERR1["Error: message required"]

    B -- Yes --> C{channel + chat_id set?}
    C -- No --> ERR2["Error: no session context"]

    C -- Yes --> D{tz without cron_expr?}
    D -- Yes --> ERR3["Error: tz only with cron"]

    D -- No --> E{tz provided?}
    E -- Yes --> F["Validate ZoneInfo(tz)"]
    F --> G{Valid?}
    G -- No --> ERR4["Error: unknown timezone"]

    E -- No & G -- Yes --> H{Schedule type?}
    H -- every_seconds --> I["CronSchedule(kind='every')"]
    H -- cron_expr --> J["CronSchedule(kind='cron')"]
    H -- at --> K["CronSchedule(kind='at')<br/>delete_after=true"]
    H -- none --> ERR5["Error: schedule required"]

    I & J & K --> L["cron_service.add_job(...)"]
    L --> M["Return: Created job '{name}' (id: {id})"]
```

### Schedule Types

| Type | Parameter | Behavior |
|------|-----------|----------|
| Recurring | `every_seconds` | Runs every N seconds |
| Cron | `cron_expr` (+`tz`) | Runs on cron schedule |
| One-shot | `at` | Runs once, then deleted |

### Delivery Configuration

Jobs created by the agent are always set with `deliver=True`, targeting the current `channel:chat_id`. This means when the job fires, the response will be sent back to the user who requested it.

---

## List Jobs

```mermaid
flowchart TD
    A["_list_jobs()"] --> B["cron_service.list_jobs()"]
    B --> C{Any jobs?}
    C -- No --> D["'No scheduled jobs.'"]
    C -- Yes --> E["Format each:<br/>'- {name} (id: {id}, {kind})'"]
    E --> F["Return formatted list"]
```

---

## Remove Job

```mermaid
flowchart TD
    A["_remove_job(job_id)"] --> B{job_id provided?}
    B -- No --> C["Error: job_id required"]
    B -- Yes --> D["cron_service.remove_job(job_id)"]
    D --> E{Found?}
    E -- Yes --> F["Removed job {id}"]
    E -- No --> G["Job {id} not found"]
```

---

## Typical Conversation

```mermaid
sequenceDiagram
    participant User
    participant Agent
    participant Cron as CronTool
    participant Svc as CronService

    User->>Agent: "Remind me to check email every morning at 9am"
    Agent->>Cron: cron(action="add", message="Check email",<br/>cron_expr="0 9 * * *", tz="America/Vancouver")
    Cron->>Svc: add_job(...)
    Svc-->>Cron: Job created
    Cron-->>Agent: "Created job 'Check email' (id: abc123)"
    Agent-->>User: "Done! I'll remind you every day at 9am."

    Note over Svc: Next morning at 9am...
    Svc->>Agent: on_cron_job(job)
    Agent-->>User: "Time to check your email!"
```
