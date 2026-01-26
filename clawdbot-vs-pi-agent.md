# Clawdbot vs Pi Agent Runtime

Understanding the division of labor between Clawdbot and its underlying agent library.

## The Big Picture

Clawdbot doesn't implement an agent runtime from scratch. It uses [Pi Agent](https://github.com/badlogic/pi-mono) (`@mariozechner/pi-coding-agent`) - a general-purpose coding agent library - and wraps it with everything needed for a multi-channel messaging bot.

Think of it this way: Pi Agent knows how to be a coding assistant. Clawdbot teaches it how to be *your* assistant, reachable from Telegram, WhatsApp, or wherever you message from.

```
┌─────────────────────────────────────────────────────────┐
│                      Clawdbot                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Channels        │  Custom Tools   │  Sandbox   │   │
│  │  - Telegram      │  - browser      │  - Docker  │   │
│  │  - WhatsApp      │  - message      │  - Policy  │   │
│  │  - Discord       │  - cron         │  - Mounts  │   │
│  │  - Signal        │  - canvas       │            │   │
│  │  - CLI/Gateway   │  - web_search   │            │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              System Prompt Builder               │   │
│  │  - 23 sections (tools, skills, identity...)     │   │
│  │  - Bootstrap files (SOUL.md, AGENTS.md...)      │   │
│  │  - Runtime info, workspace, timezone            │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│                          ▼                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Pi Agent (Library)                  │   │
│  │                                                  │   │
│  │  Core Tools:  bash, read, write, edit,          │   │
│  │               grep, find, ls                     │   │
│  │                                                  │   │
│  │  Runtime:     LLM session management            │   │
│  │               Tool execution loop               │   │
│  │               Response streaming                │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## What Pi Agent Provides

Pi Agent is a monorepo (`pi-mono`) with several packages. Clawdbot primarily uses:

| Package | Purpose |
|---------|---------|
| `@mariozechner/pi-coding-agent` | Main agent runtime, session management |
| `@mariozechner/pi-agent-core` | Core abstractions (tools, messages) |
| `@mariozechner/pi-ai` | LLM provider integrations |

### Core Tools (from Pi Agent)

These tools come directly from Pi Agent's `packages/coding-agent/src/core/tools/`:

```
bash.ts   → Shell command execution
read.ts   → Read file contents
write.ts  → Write/create files
edit.ts   → Precise file edits
grep.ts   → Search file contents
find.ts   → Find files by pattern
ls.ts     → List directories
```

Pi Agent's `bash` tool spawns a local shell process, captures output, handles timeouts, and truncates large outputs. It knows nothing about Docker or sandboxing - that's Clawdbot's job.

### Session Management

Pi Agent handles:
- Creating and resuming agent sessions
- Storing conversation history (`.jsonl` files)
- The agentic loop: prompt → LLM → parse response → execute tools → repeat
- Streaming responses back to the caller

## What Clawdbot Adds

### 1. The System Prompt

Pi Agent accepts a system prompt as a parameter. Clawdbot builds an extensive one with ~23 sections:

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts
const systemPrompt = createSystemPromptOverride(appendPrompt);

({ session } = await createAgentSession({
  systemPrompt,  // ← Clawdbot's prompt passed to Pi Agent
  tools: builtInTools,
  ...
}));
```

The system prompt defines Clawdbot's identity, available tools, skills, workspace, and behavior guidelines. See [System Prompt Sections](reference/system-prompt-sections.md) for the full template.

### 2. Custom Tools

Clawdbot adds tools Pi Agent doesn't have (`src/agents/tools/`):

| Tool | Purpose |
|------|---------|
| `browser` | Control a web browser (Playwright) |
| `canvas` | Interactive canvas for visualizations |
| `message` | Send messages to channels |
| `cron` | Schedule jobs and reminders |
| `nodes` | Interact with paired devices |
| `web_search` | Search the web (Brave API) |
| `web_fetch` | Fetch and parse web pages |
| `gateway` | Self-update and config management |
| `sessions_*` | Multi-session and sub-agent management |
| `image` | Analyze images with vision models |

These tools are registered alongside Pi Agent's core tools when creating a session.

### 3. Sandboxing

Pi Agent runs commands locally by default. Clawdbot adds optional Docker sandboxing (`src/agents/sandbox/`):

```
src/agents/sandbox/
├── docker.ts        # Container lifecycle (start, stop, exec)
├── tool-policy.ts   # Which tools allowed in sandbox
├── workspace.ts     # Mount workspace into container
├── browser.ts       # Browser control inside sandbox
└── constants.ts     # Default sandbox image config
```

When sandboxing is enabled:
1. Clawdbot spins up a Docker container
2. Mounts the workspace (read-only or read-write)
3. Routes Pi Agent's `bash`/`exec` calls to run inside the container
4. Applies tool policies (some tools blocked in sandbox mode)

The agent doesn't know it's sandboxed - Clawdbot intercepts tool execution transparently.

### 4. Channels and Routing

Pi Agent has no concept of Telegram, WhatsApp, or any messaging platform. Clawdbot adds:

- **Channel integrations**: Telegram (grammY), WhatsApp (Baileys), Discord, Signal, Slack, etc.
- **Message routing**: Which agent handles which chat
- **Session keys**: `agent:main:telegram:group:12345`
- **Reply delivery**: Formatting and sending responses back

### 5. Everything Else

- **Skills system**: Loadable capabilities from workspace
- **Memory tools**: Search and recall from MEMORY.md
- **Bootstrap files**: SOUL.md, AGENTS.md, TOOLS.md injection
- **Heartbeats**: Periodic health checks
- **Compaction**: Summarizing old history to save context

## How They Connect

The integration point is `src/agents/pi-embedded-runner/`. This is Clawdbot's wrapper around Pi Agent:

```
src/agents/pi-embedded-runner/
├── run.ts              # Main entry: runEmbeddedPiAgent()
├── run/
│   └── attempt.ts      # Creates Pi Agent session, passes system prompt
├── history.ts          # History limiting and retrieval
├── system-prompt.ts    # Builds system prompt for Pi Agent
└── google.ts           # Google/Gemini-specific handling
```

The key function is `runEmbeddedPiAgent()` which:
1. Builds the system prompt
2. Resolves tools (Pi Agent's + Clawdbot's)
3. Applies sandboxing if configured
4. Creates a Pi Agent session
5. Sends the user's message
6. Streams the response back

## Key Takeaways

1. **Pi Agent is a library, not an application** - It provides the agent loop and core file tools
2. **Clawdbot is the application** - It provides identity, channels, custom tools, and sandboxing
3. **The system prompt is Clawdbot's** - Pi Agent just receives it as a parameter
4. **Sandboxing is Clawdbot's** - Pi Agent runs locally; Clawdbot intercepts and routes to Docker
5. **You could swap Pi Agent** - In theory, Clawdbot could use a different agent runtime (though it's tightly integrated)

## Code References

| What | Where |
|------|-------|
| Pi Agent integration | `src/agents/pi-embedded-runner/` |
| System prompt builder | `src/agents/system-prompt.ts` |
| Custom tools | `src/agents/tools/` |
| Sandboxing | `src/agents/sandbox/` |
| Pi Agent source | `pi-mono/packages/coding-agent/` |
| Pi Agent core tools | `pi-mono/packages/coding-agent/src/core/tools/` |
