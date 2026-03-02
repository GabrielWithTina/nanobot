# `nanobot provider` — Provider Authentication

**Source:** `nanobot/cli/commands.py:1036-1112`

## Subcommands

| Command | Description |
|---------|-------------|
| `nanobot provider login <provider>` | Authenticate with an OAuth provider |

Currently supported OAuth providers:
- `openai-codex` — OpenAI Codex (via `oauth_cli_kit`)
- `github-copilot` — GitHub Copilot (device flow via LiteLLM)

---

## Login Dispatch

```mermaid
flowchart TD
    A["nanobot provider login PROVIDER"] --> B[Normalize name: - → _]
    B --> C{Match in PROVIDERS registry<br/>where is_oauth=true?}

    C -- No match --> D["Error: Unknown OAuth provider<br/>Show supported list"]
    C -- Match --> E{Handler registered?}

    E -- No --> F["Error: Login not implemented"]
    E -- Yes --> G["Execute handler()"]

    style D fill:#ffcdd2
    style F fill:#ffcdd2
```

The handler registry pattern (`_LOGIN_HANDLERS` + `@_register_login`) decouples provider discovery from login implementation.

---

## OpenAI Codex Login

**Source:** Lines 1073-1094

```mermaid
flowchart TD
    A[_login_openai_codex] --> B{oauth_cli_kit installed?}
    B -- No --> C["Error: pip install oauth-cli-kit"]

    B -- Yes --> D["get_token() — check cached token"]
    D --> E{Valid cached token?}

    E -- Yes --> F["Print: ✓ Authenticated"]
    E -- No --> G["login_oauth_interactive()"]

    G --> H["Browser opens OAuth flow"]
    H --> I{Token obtained?}
    I -- Yes --> F
    I -- No --> J["Error: Authentication failed"]

    style C fill:#ffcdd2
    style J fill:#ffcdd2
```

### Details

1. First attempts to retrieve a cached token via `get_token()`.
2. If no valid token exists, launches an interactive OAuth flow (opens browser).
3. On success, prints the account ID. The token is cached by `oauth_cli_kit` for future use.

---

## GitHub Copilot Login

**Source:** Lines 1097-1112

```mermaid
flowchart TD
    A[_login_github_copilot] --> B["Print: Starting device flow"]
    B --> C["litellm.acompletion(<br/>model='github_copilot/gpt-4o',<br/>messages=[hi],<br/>max_tokens=1)"]

    C --> D{Success?}
    D -- Yes --> E["Print: ✓ Authenticated"]
    D -- No --> F["Print: Authentication error"]

    style F fill:#ffcdd2
```

### Details

This leverages LiteLLM's built-in GitHub Copilot device flow:
1. Calling `acompletion` with a `github_copilot/` model triggers LiteLLM's device code flow.
2. The user is prompted to visit `https://github.com/login/device` and enter a code.
3. Once approved, LiteLLM caches the Copilot token for subsequent requests.
4. The actual API response is discarded — only the auth side-effect matters.

---

## Registration Pattern

```mermaid
classDiagram
    class _LOGIN_HANDLERS {
        <<dict>>
        openai_codex: callable
        github_copilot: callable
    }

    class _register_login {
        <<decorator>>
        +__call__(name: str) → decorator
    }

    class provider_login {
        +provider: str
        Looks up PROVIDERS registry
        Dispatches to handler
    }

    _register_login --> _LOGIN_HANDLERS : registers into
    provider_login --> _LOGIN_HANDLERS : dispatches from
```

To add a new OAuth provider:
1. Add a `ProviderSpec(is_oauth=True)` to the `PROVIDERS` registry.
2. Decorate a function with `@_register_login("provider_name")`.
