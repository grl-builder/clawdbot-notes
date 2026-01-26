# Session Schema

File format and message structure for session files.

---

## File Location

```
~/.clawdbot/agents/{{agentId}}/sessions/{{sessionId}}.jsonl
```

Examples:
- `~/.clawdbot/agents/main/sessions/main.jsonl` - default agent, main DM session
- `~/.clawdbot/agents/main/sessions/telegram:group:12345.jsonl` - Telegram group

---

## File Format

Session files are newline-delimited JSON (`.jsonl`). Each line is a standalone JSON object.

### Session Header (first line)

```json
{"type":"session","id":"main","cwd":"/Users/you/workspace","created":1705312200000}
```

### Message Entry

```json
{"type":"message","message":{"role":"user","content":"Hello!"},"timestamp":1705312200000}
{"type":"message","message":{"role":"assistant","content":"Hi there!"},"timestamp":1705312205000}
```

### Custom Entry (metadata)

```json
{"type":"custom","customType":"model-snapshot","data":{"provider":"anthropic","modelId":"claude-3-5-sonnet"}}
```

---

## Message Content Types

### Text only

```json
{"role":"user","content":"Hello!"}
```

### With images

```json
{"role":"user","content":[
  {"type":"text","text":"Check this out"},
  {"type":"image","source":{"type":"base64","mediaType":"image/png","data":"..."}}
]}
```

### Tool call

```json
{"role":"assistant","content":[
  {"type":"text","text":"Let me read that."},
  {"type":"toolCall","id":"call_abc","name":"read","arguments":{"path":"/tmp/file.txt"}}
]}
```

### Tool result

```json
{"role":"user","content":[
  {"type":"toolResult","toolUseId":"call_abc","content":"File contents..."}
]}
```

### Thinking block

```json
{"role":"assistant","content":[
  {"type":"thinking","thinking":"Let me analyze...","signature":"base64=="},
  {"type":"text","text":"Based on my analysis..."}
]}
```

---

## Complete Session Example

```jsonl
{"type":"session","id":"main","cwd":"/Users/you/workspace","created":1705312200000}
{"type":"message","message":{"role":"user","content":"[Telegram From You 2024-01-15 10:30:00] What's in /tmp/notes.txt?"},"timestamp":1705312200000}
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Let me check."},{"type":"toolCall","id":"call_1","name":"read","arguments":{"path":"/tmp/notes.txt"}}]},"timestamp":1705312201000}
{"type":"message","message":{"role":"user","content":[{"type":"toolResult","toolUseId":"call_1","content":"Meeting notes from Monday"}]},"timestamp":1705312202000}
{"type":"message","message":{"role":"assistant","content":"The file contains: Meeting notes from Monday"},"timestamp":1705312203000}
{"type":"message","message":{"role":"user","content":"[Telegram From You 2024-01-15 10:31:00] Thanks!"},"timestamp":1705312260000}
{"type":"message","message":{"role":"assistant","content":"You're welcome!"},"timestamp":1705312261000}
```

---

## See Also

- [Session History Construction](../session-history-construction.md) - how history is loaded and processed
