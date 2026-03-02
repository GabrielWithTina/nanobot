# Web Tools — WebSearch and WebFetch

**Source:** `nanobot/agent/tools/web.py`

## Purpose

Two tools for web interaction:
- **`web_search`**: Search the web via Brave Search API
- **`web_fetch`**: Fetch and extract readable content from URLs

Both support optional HTTP proxy configuration.

---

## WebSearchTool

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Search query |
| `count` | integer | No | Number of results (1-10, default 5) |

### Execution Flow

```mermaid
flowchart TD
    A["web_search(query, count)"] --> B{API key configured?}
    B -- No --> C["Error: API key not configured"]

    B -- Yes --> D["Clamp count to 1-10"]
    D --> E["GET api.search.brave.com<br/>params: q, count<br/>header: X-Subscription-Token"]

    E --> F{Success?}
    F -- No --> G["Error: {exception}"]
    F -- Yes --> H{Results?}
    H -- No --> I["No results for: query"]
    H -- Yes --> J["Format results"]

    J --> K["Return formatted list"]
```

### Output Format

```
Results for: nanobot ai assistant

1. Title One
   https://example.com/one
   Description snippet...
2. Title Two
   https://example.com/two
   Description snippet...
```

### API Key Resolution

```mermaid
flowchart TD
    A["api_key property"] --> B{Init key provided?}
    B -- Yes --> C["Use init key"]
    B -- No --> D["os.environ.get('BRAVE_API_KEY')"]
    D --> E{Found?}
    E -- Yes --> F["Use env var"]
    E -- No --> G["Return empty string (will fail)"]
```

The key is resolved at call-time (not init-time), so env var changes take effect without restart.

---

## WebFetchTool

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | URL to fetch |
| `extractMode` | string | No | `"markdown"` (default) or `"text"` |
| `maxChars` | integer | No | Max characters (default 50,000) |

### Execution Flow

```mermaid
flowchart TD
    A["web_fetch(url, extractMode, maxChars)"] --> B["_validate_url(url)"]
    B --> C{Valid?}
    C -- No --> D["Return JSON error"]

    C -- Yes --> E["httpx.AsyncClient.get(url)<br/>follow_redirects=true<br/>max_redirects=5"]

    E --> F{Success?}
    F -- No --> G["Return JSON error"]
    F -- Yes --> H["Check content-type"]

    H --> I{JSON?}
    I -- Yes --> J["Pretty-print JSON"]

    H --> K{HTML?}
    K -- Yes --> L["readability.Document(html)"]
    L --> M{extractMode?}
    M -- markdown --> N["_to_markdown(summary)"]
    M -- text --> O["_strip_tags(summary)"]

    H --> P{Other?}
    P -- Yes --> Q["Raw text"]

    J & N & O & Q --> R{len > maxChars?}
    R -- Yes --> S["Truncate"]
    R -- No --> T["Return JSON envelope"]
    S --> T
```

### URL Validation

```mermaid
flowchart TD
    A["_validate_url(url)"] --> B["urlparse(url)"]
    B --> C{scheme is http/https?}
    C -- No --> D["Invalid: wrong scheme"]
    C -- Yes --> E{Has domain?}
    E -- No --> F["Invalid: missing domain"]
    E -- Yes --> G["Valid"]
```

### HTML-to-Markdown Conversion

The `_to_markdown()` method performs lightweight HTML → markdown conversion:

| HTML | Markdown |
|------|----------|
| `<a href="url">text</a>` | `[text](url)` |
| `<h1>text</h1>` | `# text` |
| `<li>text</li>` | `- text` |
| `</p>`, `</div>` | `\n\n` |
| `<br>`, `<hr>` | `\n` |
| All other tags | Stripped |

### Return Format

Both success and error return JSON:

```json
{
  "url": "https://example.com",
  "finalUrl": "https://example.com/redirected",
  "status": 200,
  "extractor": "readability",
  "truncated": false,
  "length": 4523,
  "text": "# Page Title\n\nExtracted content..."
}
```

---

## Shared Utilities

| Function | Purpose |
|----------|---------|
| `_strip_tags(text)` | Remove HTML tags, scripts, styles; decode entities |
| `_normalize(text)` | Collapse whitespace, limit blank lines to 2 |
| `_validate_url(url)` | Validate URL scheme and domain |

## Proxy Support

Both tools accept an optional `proxy` parameter (e.g., `http://proxy:8080`). When set:

```mermaid
flowchart LR
    TOOL["Web Tool"] -->|proxy| PROXY["HTTP Proxy"]
    PROXY --> TARGET["Target URL"]
    TOOL -.->|no proxy| TARGET
```

Proxy errors are caught and reported distinctly from other errors.
