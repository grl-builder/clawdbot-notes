# Conversation History

How clawdbot stores, retrieves, and manages conversation history.

## Storage Mechanisms

### 1. File-based (Agent Sessions)

- Location: `~/.clawdbot/sessions/`
- Format: `.jsonl` (newline-delimited JSON)
- Managed by: `SessionManager` from pi-coding-agent library
- Contains: role, content, timestamps, message IDs

### 2. In-memory (Group Chats)

- Structure: `Map<string, HistoryEntry[]>`
- Location: `src/auto-reply/reply/history.ts`
- Default limit: 50 messages (`DEFAULT_GROUP_HISTORY_LIMIT`)
- Cleared when conversation context ends

## Retrieval

### Agent History

`limitHistoryTurns()` in `src/agents/pi-embedded-runner/history.ts`

- Iterates backwards to find last N user turns
- Per-DM limits via config: `channels.{provider}.dmHistoryLimit`

### Group Chat History

Built via `buildHistoryContextFromEntries()`:
- Marker: `[Chat messages since your last reply - for context]`
- Formatted with sender, body, timestamp

## Summarization & Truncation

### Turn-based Limiting

`limitHistoryTurns()` - keeps last N user turns

### Message Count Limiting

`appendHistoryEntry()` - FIFO removal when over limit

### Compaction

Handled by **Pi Agent** (not Clawdbot). When context pressure builds:
- Token-based chunking
- `generateSummary()` from `@mariozechner/pi-coding-agent` creates summaries
- Compression ratio: 0.4 down to 0.15
- Safety margin: 1.2x buffer

Pi Agent codebase: `pi-mono/packages/coding-agent/src/core/compaction/`

### Context Pruning

`src/agents/pi-extensions/context-pruning/pruner.ts`

- Removes tool results exceeding token budget
- Preserves user/assistant turns
- ~4 chars per token estimate
- Images: ~8k chars each

## Context Window Management

`src/agents/context-window-guard.ts`

```
CONTEXT_WINDOW_HARD_MIN_TOKENS = 16,000
CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32,000
```

Resolution priority:
1. Model's `contextWindow` property
2. Config: `models.providers.{provider}.models.{id}.contextWindow`
3. Agent config: `agents.defaults.contextTokens`
4. Default constant
