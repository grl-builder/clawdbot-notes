# OpenClaw Architecture Notes

These notes are for curious users who want to understand how OpenClaw actually works under the hood. Rather than documentation on *how to use* OpenClaw, this is a nuts-and-bolts breakdown of the internal implementation - tracing the path from when you send a message to when you get a response.

If you've ever wondered "what exactly happens when I message my bot?" or want to dive into the codebase with some context, start here.

> **Note**: OpenClaw was previously known as Clawdbot. Some internal references may still use the old name.

## Topics

The topics below follow the journey of a message through the system, roughly in order:

1. **[Telegram Message Flow](telegram-message-flow.md)** - How does a Telegram message even reach your server? Covers long polling, grammY, and the path from your phone to the agent.

2. **[Message Routing Flow](message-routing-flow.md)** - The complete journey from incoming message to routed session. Covers peer identification, binding evaluation, session key construction, and context loading with diagrams and concrete examples for DMs, groups, and forum topics.

3. **[Agent Routing](agent-routing.md)** - Reference for the routing system. Explains session keys, DM scopes, binding priority, identity linking, and routing for multi-agent setups.

4. **[Agent Context and Runtime](agent-context-and-runtime.md)** - The core of what makes responses work. How the system assembles your message, conversation history, system prompt, and workspace files into one context for the LLM.
   - [OpenClaw vs Pi Agent](clawdbot-vs-pi-agent.md) - Deep dive into the division of labor between OpenClaw and its underlying agent library.
   - [Session History Construction](session-history-construction.md) - Step-by-step flow of loading, sanitizing, and preparing session history.

5. **[Conversation History](conversation-history.md)** - What happens after the agent responds: session persistence, storage locations, and memory/learning mechanisms.

6. **[Session Spawning Internals](session-spawning-internals.md)** - How `sessions_spawn` and cron jobs work under the hood. Covers session creation, command lanes, inter-session communication via `sessions_send`, result announcement, and the relationship between subagents and isolated cron sessions.

## Reference

Technical specifications and templates:

- [System Prompt Sections](reference/system-prompt-sections.md) - The full system prompt template with all sections
- [Session Schema](reference/session-schema.md) - File format and message structure for `.jsonl` session files

## Key Source Files

If you want to jump straight into the code, these are the main files to look at:

| Area | File |
|------|------|
| Message entry | `src/gateway/server-methods/chat.ts` |
| Processing hub | `src/auto-reply/reply/get-reply.ts` |
| System prompt | `src/agents/system-prompt.ts` |
| System prompt params | `src/agents/system-prompt-params.ts` |
| Context files | `src/agents/bootstrap-files.ts` |
| Bootstrap budget | `src/agents/bootstrap-budget.ts` |
| History | `src/agents/pi-embedded-runner/history.ts` |
| Compaction | `src/agents/compaction.ts` |
| Agent execution | `src/agents/pi-embedded-runner/run/attempt.ts` |
| Agent routing | `src/routing/resolve-route.ts` |
| Session keys | `src/routing/session-key.ts` |
| Agent scope | `src/agents/agent-scope.ts` |
| Session store | `src/config/sessions/store.ts` |
| Command lanes | `src/process/command-queue.ts` |
| sessions_spawn | `src/agents/tools/sessions-spawn-tool.ts` |
| sessions_send | `src/agents/tools/sessions-send-tool.ts` |
| Subagent registry | `src/agents/subagent-registry.ts` |
| Cron isolated run | `src/cron/isolated-agent/run.ts` |

## Changes from Clawdbot to OpenClaw

The codebase has evolved significantly:

- **Renamed**: Clawdbot → OpenClaw
- **New routing features**: Identity linking, thread inheritance, role-based routing
- **DM scopes**: Configurable session isolation (main, per-peer, per-channel-peer, per-account-channel-peer)
- **Safety section**: Explicit safety guidelines in system prompt
- **Prompt modes**: Full, minimal, none - subagents get leaner prompts
- **Bootstrap budget**: Truncation analysis with warnings
- **Plugin hooks**: before_prompt_build, before_agent_start
- **Session write locks**: Prevent concurrent modifications
- **ACP support**: Anthropic Computer Player harness spawning
- **More channels**: Slack, LINE, Google Chat, iMessage
- **More tools**: subagents, tts, memory_search, memory_get
