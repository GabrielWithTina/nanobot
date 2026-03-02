# CLI Commands Architecture

This directory contains analysis documents for each CLI command group in `nanobot/cli/commands.py`.

## Overview

The CLI is built with [Typer](https://typer.tiangolo.com/) and exposes the following command tree:

```
nanobot
├── onboard                  # First-time setup
├── agent [-m MESSAGE]       # Chat (single-shot or interactive)
├── gateway                  # Long-running multi-channel server
├── status                   # System status
├── channels
│   ├── status               # Channel configuration overview
│   └── login                # WhatsApp QR-code device linking
├── cron
│   ├── list                 # List scheduled jobs
│   ├── add                  # Create a new job
│   ├── remove               # Delete a job
│   ├── enable               # Enable/disable a job
│   └── run                  # Manually trigger a job
└── provider
    └── login                # OAuth authentication (OpenAI Codex, GitHub Copilot)
```

## Shared Infrastructure

All commands share a few key pieces:

| Component | Role |
|-----------|------|
| `_make_provider(config)` | Factory that resolves model → provider (OpenAI Codex, Custom, LiteLLM) |
| `_init_prompt_session()` | Sets up `prompt_toolkit` for interactive input with history |
| `_print_agent_response()` | Renders assistant output (Markdown or plain text) via Rich |
| `_flush_pending_tty_input()` | Drains stale keystrokes before showing the next prompt |
| `_restore_terminal()` | Restores saved termios state on exit |

## Documents

| File | Covers |
|------|--------|
| [onboard.md](onboard.md) | `nanobot onboard` — config + workspace setup |
| [agent.md](agent.md) | `nanobot agent` — single-shot and interactive chat modes |
| [gateway.md](gateway.md) | `nanobot gateway` — full multi-channel server |
| [status.md](status.md) | `nanobot status` — system status display |
| [channels.md](channels.md) | `nanobot channels status/login` |
| [cron.md](cron.md) | `nanobot cron list/add/remove/enable/run` |
| [provider.md](provider.md) | `nanobot provider login` — OAuth flows |
