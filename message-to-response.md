# Message to Response

How an incoming message becomes an AI response.

## Entry Points

Messages enter through two main paths:

1. **Gateway** (`src/gateway/server-methods/chat.ts`)
   - `chat.send` RPC method
   - Used by CLI, web UI, mobile apps
   - Creates `MsgContext`, calls `dispatchInboundMessage()`

2. **Web/WhatsApp** (`src/web/inbound/monitor.ts`)
   - `monitorWebInbox()` listens on Baileys socket
   - Handles WhatsApp messages
   - Other channels: Telegram, Signal, Discord, etc. via plugins

## Processing Pipeline

Central hub: `getReplyFromConfig()` in `src/auto-reply/reply/get-reply.ts`

Steps before reaching AI:

1. **Debounce** - batch rapid messages from same sender
2. **Dedup** - skip recently seen messages
3. **Access control** - check allowlist
4. **Extract** - pull text, mentions, media placeholders
5. **Media staging** - download attachments to workspace
6. **Envelope** - wrap with metadata `[Channel From Timestamp] Message`
7. **Routing** - resolve agent/session via `resolveAgentRoute()`
8. **Session init** - load or create session state
9. **Context building** - build `MsgContext` object
10. **Directives** - parse inline commands (`!model`, `/think`, etc.)

## Prompt Construction

Core: `buildAgentSystemPrompt()` in `src/agents/system-prompt.ts`

### System Prompt Sections

- Base identity line
- Tooling section (available tools with descriptions)
- Tool call style guidance
- CLI quick reference
- Skills section
- Memory section (memory_search, memory_get)
- Model aliases
- Workspace (cwd, notes)
- Documentation links
- User identity (owner phone numbers)
- Time/timezone
- Reply tags guidance
- Messaging section (channel routing)
- Voice/TTS hints
- Sandbox constraints (if sandboxed)
- Project context (SOUL.md, AGENTS.md, etc.)
- Silent replies (NO_REPLY token)
- Heartbeats protocol
- Runtime line (agent, host, OS, model)

### Context File Injection

`src/agents/bootstrap-files.ts` - loads workspace context files

Truncation: 70% head + 20% tail, max 20k chars (`src/agents/pi-embedded-helpers/bootstrap.ts`)

### Prompt Modes

| Mode | Use | Includes |
|------|-----|----------|
| full | Main agent | All sections |
| minimal | Subagents | Tooling, Workspace, Runtime only |
| none | Minimal agents | Just identity line |

Determined by: `isSubagentSessionKey(params.sessionKey)`

## Agent Execution

`src/agents/pi-embedded-runner/run/attempt.ts`

1. Create session with system prompt
2. Run `before_agent_start` hooks (can prepend context)
3. Detect/load images in prompt
4. Call `activeSession.prompt(effectivePrompt, images)`
5. Subscribe to response events
6. Format response via `buildEmbeddedRunPayloads()`
