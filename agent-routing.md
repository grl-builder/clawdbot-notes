# Agent Routing

How OpenClaw determines which agent handles a message and what session to use.

## Overview

Agent routing resolves incoming messages to a specific agent and session. For most users running a single agent, routing is automatic - all messages go to the default agent with session key `agent:main:main`.

This becomes relevant when:
- Running multiple agents with different personalities or capabilities
- Binding specific agents to specific channels or chat groups
- Using per-peer sessions (separate conversation history per contact)
- Routing threads to inherit their parent's agent binding
- Assigning agents based on Discord roles

## Session Key Structure

Every agent session is identified by a session key with this pattern:

```
agent:<agentId>:<rest>
```

The `<rest>` portion varies by context:

| Context | Session Key Pattern | Example |
|---------|---------------------|---------|
| Default DM (collapsed) | `agent:<agentId>:main` | `agent:main:main` |
| Per-peer DM | `agent:<agentId>:direct:<peerId>` | `agent:main:direct:123456` |
| Per-channel-peer DM | `agent:<agentId>:<channel>:direct:<peerId>` | `agent:main:telegram:direct:123456` |
| Per-account-channel-peer DM | `agent:<agentId>:<channel>:<accountId>:direct:<peerId>` | `agent:main:telegram:default:direct:123456` |
| Group chat | `agent:<agentId>:<channel>:group:<chatId>` | `agent:main:telegram:group:-12345` |
| Channel (Discord) | `agent:<agentId>:<channel>:channel:<channelId>` | `agent:main:discord:channel:98765` |
| Thread | `agent:<agentId>:...:thread:<threadId>` | `agent:main:telegram:group:-12345:thread:999` |
| Subagent | `agent:<agentId>:subagent:<runId>` | `agent:main:subagent:abc123` |
| Cron job | `agent:<agentId>:cron:<jobId>:run:<runId>` | `agent:main:cron:daily-check:run:xyz` |
| ACP session | `agent:<agentId>:acp:<sessionId>` | `agent:main:acp:claude-code-123` |

## DM Session Scopes

OpenClaw supports different levels of session isolation for direct messages, controlled by the `session.dmScope` config option:

| Scope | Behavior | Use Case |
|-------|----------|----------|
| `main` (default) | All DMs share one session per agent | Single conversational thread with you |
| `per-peer` | Each sender gets their own session | Multiple users, each with isolated history |
| `per-channel-peer` | Each sender on each channel gets own session | Same person on Telegram vs WhatsApp has separate histories |
| `per-account-channel-peer` | Full isolation by account + channel + peer | Multiple bot accounts with full separation |

## Identity Linking

OpenClaw can collapse sessions for the same person across different channels using `session.identityLinks`:

```json
{
  "session": {
    "dmScope": "per-peer",
    "identityLinks": {
      "alice": ["telegram:12345", "signal:+15551234567", "discord:98765"]
    }
  }
}
```

With this config, messages from any of Alice's accounts route to the same session key, preserving conversation continuity across channels.

## Binding Priority

When a message arrives, the router evaluates bindings in this order (first match wins):

```
┌─────────────────────────────────────────────────────────────┐
│                     Binding Evaluation Order                │
├─────────────────────────────────────────────────────────────┤
│  1. binding.peer         - Exact peer (DM/group) match      │
│  2. binding.peer.parent  - Thread inherits parent binding   │
│  3. binding.guild+roles  - Discord guild + role membership  │
│  4. binding.guild        - Discord guild (no role filter)   │
│  5. binding.team         - Slack team/workspace             │
│  6. binding.account      - Specific channel account         │
│  7. binding.channel      - Any account on channel (*)       │
│  8. default              - Falls back to default agent      │
└─────────────────────────────────────────────────────────────┘
```

### Thread Parent Inheritance

For threaded messages (Telegram topics, Discord threads, Slack threads), if no direct binding matches the thread, OpenClaw checks if the parent peer has a binding. This allows groups with bindings to have their threads automatically use the same agent.

### Role-Based Routing (Discord)

Discord-specific: you can route different roles to different agents:

```json
{
  "bindings": [
    {
      "agentId": "support",
      "match": {
        "channel": "discord",
        "guildId": "123456",
        "roles": ["987654321"]  
      }
    }
  ]
}
```

Members with that role get routed to the `support` agent; others fall through to lower-priority bindings.

## Binding Configuration

Bindings are defined in `openclaw.json` under the `bindings` array:

```json
{
  "bindings": [
    {
      "agentId": "coder",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "group", "id": "-12345" }
      }
    },
    {
      "agentId": "assistant",
      "match": {
        "channel": "discord",
        "accountId": "bot-account-1",
        "guildId": "98765"
      }
    },
    {
      "agentId": "main",
      "match": {
        "channel": "telegram",
        "accountId": "*"
      }
    }
  ]
}
```

### Match Fields

| Field | Description |
|-------|-------------|
| `channel` | Channel type: `telegram`, `discord`, `signal`, `whatsapp`, `slack`, etc. |
| `accountId` | Specific account (or `*` for any account on that channel) |
| `peer.kind` | Peer type: `direct`, `group`, or `channel` |
| `peer.id` | The chat/group/channel ID |
| `guildId` | Discord server ID |
| `teamId` | Slack workspace ID |
| `roles` | Array of Discord role IDs (any match routes) |

## Caching

The routing system is heavily cached to avoid repeated binding evaluation:

1. **Evaluated bindings cache**: Pre-processes and indexes bindings per channel+account
2. **Binding index**: Organizes bindings by match type (peer, guild, team, account, channel) for O(1) lookups
3. **Resolved route cache**: Caches final routing decisions by full input key

Caches auto-clear when size limits are exceeded (~4000 routes, ~2000 binding sets).

## Special Session Types

OpenClaw recognizes several special session key patterns:

| Pattern | Detection | Purpose |
|---------|-----------|---------|
| `subagent:*` or contains `:subagent:` | `isSubagentSessionKey()` | Spawned sub-agent sessions |
| `cron:*` or contains `:cron:` | `isCronSessionKey()` | Scheduled job executions |
| `acp:*` or contains `:acp:` | `isAcpSessionKey()` | Anthropic Computer Player sessions |

These affect behavior like:
- System prompt mode (`minimal` for subagents)
- Tool availability
- Session isolation policies

## Key Code Locations

| Component | File |
|-----------|------|
| Route resolution | `src/routing/resolve-route.ts` |
| Session key building | `src/routing/session-key.ts` |
| Session key parsing | `src/sessions/session-key-utils.ts` |
| Agent bindings | `src/routing/bindings.ts` |
| Account ID handling | `src/routing/account-id.ts` |
| Agent scope resolution | `src/agents/agent-scope.ts` |

## For Single-Agent Users

If you're running the default setup with one agent, you don't need to configure anything. All your messages (Telegram DMs, WhatsApp chats, CLI, etc.) route to `agent:main:main` automatically.

The routing system exists to support more complex multi-agent deployments, role-based access, and cross-platform identity linking.
