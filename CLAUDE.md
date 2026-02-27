# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight personal AI assistant framework (~4k lines of core agent code). It provides a core agent loop with multi-platform chat integration across 10+ channels (Telegram, Discord, Slack, Matrix, WhatsApp, Feishu, DingTalk, Email, QQ, Mochat). Python 3.11+, MIT licensed.

## Common Commands

```bash
# Install (editable, with dev deps)
pip install -e ".[dev]"
# Or with matrix support:
pip install -e ".[dev,matrix]"

# Run tests
pytest tests/
pytest tests/test_commands.py                          # single file
pytest tests/test_commands.py::test_onboard_fresh_install  # single test

# Lint
ruff check nanobot/
ruff check --fix nanobot/    # auto-fix
ruff format nanobot/         # format

# CLI entry points
nanobot onboard              # first-time setup
nanobot agent                # interactive chat
nanobot agent -m "message"   # single message
nanobot gateway              # start gateway (channels)
nanobot status               # system status
```

## Code Style

- **Ruff** linter: line-length 100, target Python 3.11, rules `E/F/I/N/W`, `E501` ignored
- **pytest-asyncio** with `asyncio_mode = "auto"` — async test functions are auto-detected
- Logging via `loguru` (not stdlib logging)

## Architecture

### Agent Loop (`nanobot/agent/loop.py`)

The core processing cycle: receive `InboundMessage` from bus → build context (system prompt + history + memory + skills) → call LLM → execute tool calls → send `OutboundMessage` back. Runs up to `max_iterations` (default 40) tool-call rounds per message.

### Context Assembly (`nanobot/agent/context.py`)

`ContextBuilder` assembles the system prompt from layers:
1. Core identity (system info, runtime metadata)
2. Bootstrap files: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md` (from workspace)
3. Long-term memory: `memory/MEMORY.md`
4. Always-on skills (SKILL.md files)
5. Skills summary (available skills list)

### Two-Layer Memory (`nanobot/agent/memory.py`)

- **MEMORY.md**: persistent long-term facts (updated by LLM consolidation)
- **HISTORY.md**: grep-searchable event log with `[YYYY-MM-DD HH:MM]` timestamps
- Consolidation is triggered by `memory_window` threshold — LLM summarizes old messages into these files via a `save_memory` tool call

### Sessions (`nanobot/session/manager.py`)

Append-only JSONL storage keyed by `channel:chat_id`. The `last_consolidated` offset tracks which messages have been summarized to avoid re-processing. `get_history()` returns only unconsolidated messages, aligned to start at a user turn.

### Provider Registry (`nanobot/providers/registry.py`)

Single source of truth: a `PROVIDERS` tuple of `ProviderSpec` dataclasses. To add a new provider: (1) add a `ProviderSpec` to `PROVIDERS`, (2) add a field to `ProvidersConfig` in `config/schema.py`. Env vars, model prefixing, config matching, and status display all derive from the registry. Order matters — it controls match priority (gateways first).

### Channel Adapters (`nanobot/channels/`)

All channels extend `BaseChannel` (in `base.py`) and implement `start()`, `stop()`, `send()`. Each channel translates platform-specific events into `InboundMessage` objects routed through `MessageBus`. Access control via `is_allowed()` checking `allowFrom` config. `manager.py` orchestrates channel lifecycle.

### Message Bus (`nanobot/bus/`)

`MessageBus` (in `queue.py`) provides async message routing. `InboundMessage` flows channel→agent, `OutboundMessage` flows agent→channel. Event types defined in `events.py`.

### Tool System (`nanobot/agent/tools/`)

`ToolRegistry` manages tool discovery and validation. Built-in tools: `ExecTool` (shell), `ReadFileTool`/`WriteFileTool`/`EditFileTool`/`ListDirTool` (filesystem), `WebSearchTool`/`WebFetchTool` (web), `MessageTool`, `SpawnTool` (background tasks), `CronTool`. MCP server tools are loaded dynamically.

### Skills (`nanobot/agent/skills.py`)

`SkillsLoader` loads skills from workspace `skills/` directory. Each skill has a `SKILL.md` and optional code. Skills marked `available: always` are included in every system prompt; others are loaded on demand via `read_file`.

### Configuration (`nanobot/config/`)

Pydantic-based schema in `schema.py`, loaded from `~/.nanobot/config.json`. Top-level sections: `providers`, `agents` (model defaults), `channels`, `tools` (exec restrictions, MCP servers), `cron`, `workspace`.

## Key Directories

- `nanobot/agent/` — core loop, context, memory, subagent, tools
- `nanobot/channels/` — chat platform adapters (one file per platform)
- `nanobot/providers/` — LLM provider abstraction and registry
- `nanobot/config/` — Pydantic config schema and loader
- `nanobot/session/` — conversation session management (JSONL)
- `nanobot/skills/` — bundled skills (weather, github, tmux, etc.)
- `nanobot/templates/` — workspace template files (SOUL.md, TOOLS.md, etc.)
- `nanobot/bus/` — message bus and event types
- `nanobot/cli/` — typer CLI commands
- `nanobot/cron/` — scheduled task service
- `nanobot/heartbeat/` — periodic proactive wake-up
- `bridge/` — WhatsApp bridge (Node.js, compiled into wheel)
- `tests/` — pytest test suite
