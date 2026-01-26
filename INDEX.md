# Clawdbot Architecture Notes

These notes are for developers and curious users who want to understand how Clawdbot actually works under the hood. Rather than documentation on *how to use* Clawdbot, this is a nuts-and-bolts breakdown of the internal architecture - tracing the path from when you send a message to when you get a response.

If you've ever wondered "what exactly happens when I message my bot?" or want to dive into the codebase with some context, start here.

## Topics

The topics below follow the journey of a message through the system, roughly in order:

1. **[Telegram Message Flow](telegram-message-flow.md)** - How does a Telegram message even reach your server? Covers long polling, grammY, and the path from your phone to the agent.

2. **[Agent Routing](agent-routing.md)** - Once a message arrives, which agent handles it? Explains session keys and routing for multi-agent setups. (For single-agent users, this is automatic.)

3. **[Agent Context and Runtime](agent-context-and-runtime.md)** - The core of what makes responses work. How the system assembles your message, conversation history, system prompt, and workspace files into one context for the LLM.

4. **[Conversation History](conversation-history.md)** - Where your conversation lives on disk, how it's loaded, and how compaction keeps things manageable.

## Key Source Files

If you want to jump straight into the code, these are the main files to look at:

| Area | File |
|------|------|
| Message entry | `src/gateway/server-methods/chat.ts` |
| Processing hub | `src/auto-reply/reply/get-reply.ts` |
| System prompt | `src/agents/system-prompt.ts` |
| Context files | `src/agents/bootstrap-files.ts` |
| History | `src/agents/pi-embedded-runner/history.ts` |
| Compaction | `src/agents/compaction.ts` |
| Agent execution | `src/agents/pi-embedded-runner/run/attempt.ts` |
