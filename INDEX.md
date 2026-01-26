# Clawdbot Architecture Notes

Notes on clawdbot's internals, focusing on message handling and prompt construction.

## Topics

- [Message to Response](message-to-response.md) - The journey from incoming message to AI response
- [Conversation History](conversation-history.md) - How history is stored, retrieved, and managed

## Key Source Files

| Area | File |
|------|------|
| Message entry | `src/gateway/server-methods/chat.ts` |
| Processing hub | `src/auto-reply/reply/get-reply.ts` |
| System prompt | `src/agents/system-prompt.ts` |
| Context files | `src/agents/bootstrap-files.ts` |
| History | `src/agents/pi-embedded-runner/history.ts` |
| Compaction | `src/agents/compaction.ts` |
| Agent execution | `src/agents/pi-embedded-runner/run/attempt.ts` |
