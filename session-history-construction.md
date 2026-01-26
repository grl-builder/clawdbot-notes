# Session History Construction

What actually happens when session history is loaded and assembled for the LLM.

---

## The Flow

When you send a message, here's what happens with session history:

### 1. Load the Session File

```typescript
SessionManager.open(sessionFile)
// e.g. ~/.clawdbot/agents/main/sessions/main.jsonl
```

The entire `.jsonl` file is read into memory. Each line is a JSON entry - session header, messages, custom metadata.

### 2. Sanitize History

Before using the messages, several sanitization passes run:

```typescript
// Fix provider-specific quirks
sanitizeSessionHistory(messages, { modelApi, provider, sessionManager })

// Validate turn ordering for Gemini (needs user → assistant alternation)
validateGeminiTurns(messages)

// Validate turn ordering for Anthropic
validateAnthropicTurns(messages)
```

**Google turn ordering fix**: If history starts with an assistant message, a synthetic user "bootstrap" turn is prepended so the conversation starts correctly.

### 3. Limit History (DMs only)

If configured, limit to the last N **user turns** (not messages):

```typescript
limitHistoryTurns(messages, dmHistoryLimit)
```

**Important**: There is no default limit. If `dmHistoryLimit` is not configured, all history is kept.

Configuration:
```json
{
  "channels": {
    "telegram": {
      "dmHistoryLimit": 50
    }
  }
}
```

Per-user override:
```json
{
  "channels": {
    "telegram": {
      "dmHistoryLimit": 100,
      "dms": {
        "123456789": { "historyLimit": 20 }
      }
    }
  }
}
```

The limit is **user turns**, not messages. A turn includes the user message plus all assistant responses until the next user message.

### 4. Replace Agent Messages

The sanitized, limited history replaces the agent's message buffer:

```typescript
activeSession.agent.replaceMessages(limited)
```

### 5. Run before_agent_start Hooks

Plugins can inject context via the `before_agent_start` hook:

```typescript
if (hookRunner?.hasHooks("before_agent_start")) {
  const hookResult = await hookRunner.runBeforeAgentStart({
    prompt: params.prompt,
    messages: activeSession.messages,
  }, ctx);
  
  if (hookResult?.prependContext) {
    effectivePrompt = `${hookResult.prependContext}\n\n${params.prompt}`;
  }
}
```

This is how plugins can inject RAG results, memory snippets, or other context. **There is no built-in RAG** - the agent is instructed (via system prompt) to use `memory_search` tool, but that's agent-initiated.

### 6. Repair Orphaned Messages

If the last entry in history is a user message (orphaned - no assistant response), it's removed to prevent consecutive user turns:

```typescript
const leafEntry = sessionManager.getLeafEntry();
if (leafEntry?.type === "message" && leafEntry.message.role === "user") {
  sessionManager.branch(leafEntry.parentId);
  // Removes orphaned user message
}
```

### 7. Append Your Message

Finally, your actual message is sent to the agent as a new prompt:

```typescript
activeSession.prompt(effectivePrompt, images)
```

---

## What Goes to the LLM

The final context sent to the LLM is:

1. **System prompt** (built separately, see [reference/system-prompt-sections.md](reference/system-prompt-sections.md))
2. **Session history** (sanitized, limited, repaired)
3. **Your message** (possibly with prepended hook context)

---

## Group Chat History (Different System)

For group chats, a separate in-memory history system tracks recent messages:

- **Default limit**: 50 messages (`DEFAULT_GROUP_HISTORY_LIMIT`)
- **Purpose**: Provides context of chat since last agent reply
- **Storage**: In-memory `Map<string, HistoryEntry[]>`, not persisted
- **Marker**: Prepended with `[Chat messages since your last reply - for context]`

This is NOT the same as DM session history. Group history is ephemeral context, not a persistent session file.

---

## Auto-Compaction on Context Overflow

What happens if history gets too large for the model's context window?

**Pi Agent** (the underlying runtime library, not Clawdbot) handles this automatically:

1. **Detect overflow** - LLM provider returns a context length error
2. **Trigger compaction** - Pi Agent summarizes older messages using `generateSummary()`
3. **Retry** - If compaction succeeds, the prompt is retried with compressed history

```
Context overflow → Auto-compaction → Summarize old turns → Retry prompt
```

Compaction uses token-based chunking with adaptive ratios. Large messages get smaller chunks to stay within limits. A 20% safety margin accounts for token estimation inaccuracy.

If compaction fails (nothing left to compact, or still too large), the error propagates to the user.

**Key files** (Pi Agent codebase, not Clawdbot):
- `pi-mono/packages/coding-agent/src/core/compaction/` - compaction logic
- `generateSummary()` from `@mariozechner/pi-coding-agent`

---

## Summary

```
Session File → Load → Sanitize → Limit (if configured) → Hooks → Repair → Append Message → LLM
                                                                                          ↓
                                                                    (if overflow) → Auto-compact → Retry
```

No automatic RAG. No default turn limit. History grows unbounded unless you configure `dmHistoryLimit`, but auto-compaction kicks in if you hit the context window ceiling.

---

## Key Files

| Component | File |
|-----------|------|
| History limiting | `src/agents/pi-embedded-runner/history.ts` |
| Session sanitization | `src/agents/pi-embedded-runner/google.ts` |
| Main run flow | `src/agents/pi-embedded-runner/run/attempt.ts` |
| Hook runner | `src/plugins/hooks.ts` |
| Group history | `src/auto-reply/reply/history.ts` |

---

## See Also

- [Session Schema](reference/session-schema.md) - file format and message structure
