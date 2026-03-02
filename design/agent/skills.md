# SkillsLoader — Skill Discovery and Loading

**Source:** `nanobot/agent/skills.py`

## Purpose

Discovers, loads, and manages agent skills — markdown files (`SKILL.md`) that teach the agent how to use specific tools or perform certain tasks. Skills are loaded from two sources with workspace taking priority.

## Class Overview

```mermaid
classDiagram
    class SkillsLoader {
        +workspace: Path
        +workspace_skills: Path
        +builtin_skills: Path

        +list_skills(filter_unavailable) → list[dict]
        +load_skill(name) → str | None
        +load_skills_for_context(names) → str
        +build_skills_summary() → str
        +get_always_skills() → list[str]
        +get_skill_metadata(name) → dict | None
        -_check_requirements(meta) → bool
        -_get_skill_meta(name) → dict
        -_get_skill_description(name) → str
        -_get_missing_requirements(meta) → str
        -_strip_frontmatter(content) → str
        -_parse_nanobot_metadata(raw) → dict
    }
```

## Skill Sources (Priority Order)

```mermaid
flowchart TD
    A["list_skills()"] --> B["Scan workspace/skills/"]
    B --> C["Scan nanobot/skills/ (builtin)"]
    C --> D{"Same name exists<br/>in workspace?"}
    D -- Yes --> E["Skip builtin (workspace wins)"]
    D -- No --> F["Include builtin"]

    B & F --> G{filter_unavailable?}
    G -- Yes --> H["Check requirements (bins, env)"]
    H --> I["Return filtered list"]
    G -- No --> J["Return all skills"]
```

| Source | Path | Priority |
|--------|------|----------|
| Workspace | `~/.nanobot/workspace/skills/{name}/SKILL.md` | Highest (overrides builtin) |
| Builtin | `nanobot/skills/{name}/SKILL.md` | Lower (fallback) |

## Skill File Structure

```
skills/
├── weather/
│   └── SKILL.md       # Frontmatter + instructions
├── github/
│   └── SKILL.md
├── tmux/
│   └── SKILL.md
└── ...
```

### SKILL.md Frontmatter

```yaml
---
description: "Weather lookups using wttr.in"
always: false
metadata: '{"nanobot": {"requires": {"bins": ["curl"]}, "always": false}}'
---

# Weather Skill

Instructions for the agent on how to use this skill...
```

## Progressive Loading Strategy

```mermaid
flowchart TD
    A["System Prompt Assembly"]
    A --> B["Always-on skills:<br/>Full content included"]
    A --> C["Skills summary:<br/>XML catalog (name + description + path)"]

    C --> D["Agent reads summary"]
    D --> E{"Need a skill?"}
    E -- Yes --> F["read_file(SKILL.md path)"]
    F --> G["Full skill loaded on demand"]
    E -- No --> H["No additional cost"]
```

This two-tier approach keeps the system prompt lean:
1. **Always-on skills** (e.g., critical skills marked `always: true`) are injected into every prompt.
2. **Other skills** appear only as a lightweight XML summary. The agent loads full content via `read_file` when needed.

## Skills Summary Format

```mermaid
flowchart TD
    A["build_skills_summary()"] --> B["list_skills(filter=false)"]
    B --> C["For each skill"]
    C --> D["Get metadata + check requirements"]
    D --> E{Available?}
    E -- Yes --> F["<skill available='true'>"]
    E -- No --> G["<skill available='false'><br/><requires>CLI: curl</requires>"]
    F & G --> H["Add name, description, location"]
    H --> C
    C -- done --> I["Return XML string"]
```

Output example:
```xml
<skills>
  <skill available="true">
    <name>weather</name>
    <description>Weather lookups using wttr.in</description>
    <location>/home/user/.nanobot/workspace/skills/weather/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>tmux</name>
    <description>Terminal multiplexer management</description>
    <location>/home/user/.nanobot/workspace/skills/tmux/SKILL.md</location>
    <requires>CLI: tmux</requires>
  </skill>
</skills>
```

## Requirement Checking

```mermaid
flowchart TD
    A["_check_requirements(meta)"] --> B["requires.bins"]
    B --> C["For each binary"]
    C --> D{"shutil.which(bin)?"}
    D -- Not found --> FAIL["Return False"]
    D -- Found --> C

    A --> E["requires.env"]
    E --> F["For each env var"]
    F --> G{"os.environ.get(var)?"}
    G -- Not set --> FAIL
    G -- Set --> F

    C & F -- all pass --> OK["Return True"]
```

Requirements support two types:
- **`bins`**: CLI executables that must be in PATH (checked via `shutil.which`)
- **`env`**: Environment variables that must be set

## Always-On Skills

```mermaid
flowchart TD
    A["get_always_skills()"] --> B["list_skills(filter=true)"]
    B --> C["For each skill"]
    C --> D["Get metadata"]
    D --> E{"always=true?"}
    E -- Yes --> F["Add to result"]
    E -- No --> C
    F --> C
    C -- done --> G["Return skill name list"]
```

Skills marked `always: true` in their frontmatter metadata are loaded into every system prompt. Use sparingly — each always-on skill adds to every LLM call's token count.
