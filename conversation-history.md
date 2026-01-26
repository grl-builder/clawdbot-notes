# Session Persistence

What happens after the agent responds - how conversations are saved and how memory works.

## Appending to Session File

After each turn, Pi Agent's `SessionManager` appends entries to the session file:

```
~/.clawdbot/agents/{agentId}/sessions/{sessionId}.jsonl
```

Example entries:

```json
{"type":"message","message":{"role":"user","content":"[Telegram From You...] Hello!"},"timestamp":1705312200}
{"type":"message","message":{"role":"assistant","content":"Hi there!"},"timestamp":1705312205}
```

Each line is a complete JSON object. Tool calls and results are also stored:

```json
{"type":"message","message":{"role":"assistant","content":[{"type":"toolCall","id":"call_1","name":"read",...}]},...}
{"type":"message","message":{"role":"user","content":[{"type":"toolResult","toolUseId":"call_1","content":"..."}]},...}
```

For the complete file format, see [Session Schema](reference/session-schema.md).

---

## Storage Location

| Session Type | Path |
|--------------|------|
| Default DM | `~/.clawdbot/agents/main/sessions/main.jsonl` |
| Telegram group | `~/.clawdbot/agents/main/sessions/telegram:group:12345.jsonl` |
| Custom agent | `~/.clawdbot/agents/{agentId}/sessions/{sessionId}.jsonl` |

Pi Agent's `SessionManager` (from `@mariozechner/pi-coding-agent`) handles all reads and writes.

---

## Group Chat Context (Ephemeral)

**Why this exists**: In group chats, the bot only responds when mentioned. But it needs to know what people were discussing before being pinged. This in-memory buffer captures recent group messages to provide that context.

**How it works**:
- **Storage**: `Map<string, HistoryEntry[]>` - in-memory, not persisted to disk
- **Limit**: 50 messages (`DEFAULT_GROUP_HISTORY_LIMIT`)
- **Lifetime**: Cleared when the process restarts
- **Injected as**: `[Chat messages since your last reply - for context]`

**This is separate from session files**. Group chats can still have persistent `.jsonl` session files (e.g., `telegram:group:12345.jsonl`) for the bot's own conversation history. The ephemeral buffer is *additional* context about what others said in the group.

---

## Post-Turn Memory Events

Memory reading/writing during the agent loop is covered in [Agent Context and Runtime](agent-context-and-runtime.md#memory-system). This section covers events that happen **after** the turn completes or at session boundaries.

### Session Memory Hook (on `/new`)

When you start a new session with `/new`, the **session-memory hook** saves a summary of the previous session:

```
memory/{date}-{slug}.md
```

What it does:
1. Extracts last 15 messages from the ending session
2. Uses LLM to generate a descriptive slug (e.g., `2024-01-15-refactor-auth-flow.md`)
3. Writes a markdown file with session metadata and conversation summary

This only fires on `/new` command, not after every response.

**Source**: `src/hooks/bundled/session-memory/handler.ts`

### Pre-Compaction Memory Flush

If a session approaches the token limit, the system triggers a memory flush before compaction:

1. Prompts the agent: "Pre-compaction memory flush. Store durable memories now"
2. Agent gets one turn to save important info to `memory/YYYY-MM-DD.md`
3. Then compaction summarizes older history

This gives the agent a chance to preserve important context before it gets compressed.

**Source**: `src/auto-reply/reply/memory-flush.ts`

---

## Key Files

| Component | File |
|-----------|------|
| Session Manager | `@mariozechner/pi-coding-agent` (Pi Agent library) |
| Group history | `src/auto-reply/reply/history.ts` |
| Memory tools | `src/agents/tools/memory-tool.ts` |
| Session memory hook | `src/hooks/bundled/session-memory/handler.ts` |
