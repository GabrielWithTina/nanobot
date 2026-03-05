# Notes
- In Cli mode, when processing a user input, the CLI refuse more user input until the previous procecessing is completed
- In Gateway mode, we can send multiple messages from QQ, but they are processed squentially.
  - For each session, the input messages are in the same queue and processed sequentially by using a global lock. (This is not applicable in production environment)
- In gateway mode, the inbound messages are processed sequentially even if they are coming from multiple channels concurrently.
  - `async def _dispatch(self, msg: InboundMessage) -> None:`
  - IMHO, this is because states are associated with `AgentLoop`. **all the states should be associated with an specific session instead.**

# Conversation consolidation
## Run `/new`
It uses a individual agent to consolidate all the conversation history in current session with long term memory `~/.nanobot/workspace/memory/MEMORY.md`

The sent message = Content in `MEMORY.md` + content sections extracted from roles in current session

**Caveats**: 
- The content are truncated to avoid large content overwhelming the context but this leads to un-compreshensive memory consolidation
```markdown
Process this conversation and call the save_memory tool with your consolidation.

## Current Long-term Memory
.....


## Conversation to Process
[2026-03-02T17:38] USER: 1
[2026-03-02T17:38] ASSISTANT: Hey there! 👋 I'm nanobot, your personal AI assistant. How can I help you today?
[2026-03-02T17:41] USER: Send me nanobot design doc for agent loop
[2026-03-02T17:41] TOOL: /home/xiaos/git/nanobot/CLAUDE.md
/home/xiaos/git/nanobot/.venv/lib/python3.12/site-packages/litellm/llms/litellm_proxy/skills/README.md
...

...
```

## Auto consolidation
When unconsolidated conversation history >= self.memory_window, it will trigger a background asynchronous task for consoliation.

The sent message = Content in `MEMORY.md` + content sections extracted from ready-to-consolidated roles
