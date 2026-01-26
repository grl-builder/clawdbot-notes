# Agent Routing

How Clawdbot determines which agent handles a message and what session to use.

## Overview

Agent routing resolves incoming messages to a specific agent and session. For most users running a single agent, routing is automatic - all messages go to the default agent with session key `agent:main:main`.

This becomes relevant when:
- Running multiple agents with different personalities or capabilities
- Binding specific agents to specific channels or chat groups
- Using per-peer sessions (separate conversation history per contact)

## Session Key Structure

Every agent session is identified by a session key:

```
agent:<agentId>:<channel>:<type>:<id>
```

Examples:
- `agent:main:main` - Default agent, main session (all DMs collapsed)
- `agent:main:telegram:group:12345` - Default agent, specific Telegram group
- `agent:coder:discord:guild:98765` - Custom "coder" agent bound to a Discord server

## Routing Priority

When a message arrives, the router checks bindings in this order:

1. **Peer match** - Exact DM or group ID binding
2. **Guild/Team** - Discord guild or Slack team binding
3. **Account** - Specific channel account binding
4. **Channel** - Any account on a channel binding
5. **Default** - Falls back to the default agent

The first match wins.

## Key Code Locations

| Component | File |
|-----------|------|
| Route resolution | `src/routing/resolve-route.ts` |
| Session key building | `src/routing/session-key.ts` |
| Agent bindings | `clawdbot.json` → `agents[].bindings` |

## For Single-Agent Users

If you're running the default setup with one agent, you don't need to configure anything. All your messages (Telegram DMs, WhatsApp chats, CLI, etc.) route to `agent:main:main` automatically.

The routing system exists to support more complex multi-agent deployments.
