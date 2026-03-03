# OpenClaw vs Pi Agent Runtime

Understanding the division of labor between OpenClaw and its underlying agent library.

## The Big Picture

OpenClaw doesn't implement an agent runtime from scratch. It uses [Pi Agent](https://github.com/badlogic/pi-mono) (`@mariozechner/pi-coding-agent`) - a general-purpose coding agent library - and wraps it with everything needed for a multi-channel messaging bot.

Think of it this way: Pi Agent knows how to be a coding assistant. OpenClaw teaches it how to be *your* assistant, reachable from Telegram, WhatsApp, or wherever you message from.

```
┌──────────────────────────────────────────────────────────────┐
│                        OpenClaw                              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Channels        │  Custom Tools    │  Sandbox         │  │
│  │  - Telegram      │  - browser       │  - Docker        │  │
│  │  - WhatsApp      │  - message       │  - Tool Policy   │  │
│  │  - Discord       │  - cron          │  - Mounts        │  │
│  │  - Signal        │  - canvas        │  - Elevated Exec │  │
│  │  - Slack         │  - nodes         │                  │  │
│  │  - iMessage      │  - web_search    │                  │  │
│  │  - CLI/Gateway   │  - subagents     │                  │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              System Prompt Builder                      │  │
│  │  - 20+ sections (tools, skills, safety, identity...)   │  │
│  │  - Bootstrap files (SOUL.md, AGENTS.md, MEMORY.md...)  │  │
│  │  - Prompt modes (full/minimal/none)                    │  │
│  │  - Runtime info, workspace, timezone                   │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Plugins & Hooks                            │  │
│  │  - before_prompt_build (RAG injection)                 │  │
│  │  - before_agent_start (legacy prompt build)            │  │
│  │  - Session memory hooks                                 │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                   │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Pi Agent (Library)                         │  │
│  │                                                         │  │
│  │  Core Tools:  bash, read, write, edit,                 │  │
│  │               grep, find, ls, apply_patch              │  │
│  │                                                         │  │
│  │  Runtime:     LLM session management                   │  │
│  │               Tool execution loop                       │  │
│  │               Response streaming                        │  │
│  │               Auto-compaction                           │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## What Pi Agent Provides

Pi Agent is a monorepo (`pi-mono`) with several packages. OpenClaw primarily uses:

| Package | Purpose |
|---------|---------|
| `@mariozechner/pi-coding-agent` | Main agent runtime, session management, compaction |
| `@mariozechner/pi-agent-core` | Core abstractions (tools, messages, streaming) |
| `@mariozechner/pi-ai` | LLM provider integrations (Claude, GPT, Gemini, Ollama) |

### Core Tools (from Pi Agent)

These tools come directly from Pi Agent's `packages/coding-agent/src/core/tools/`:

```
bash.ts        → Shell command execution
read.ts        → Read file contents  
write.ts       → Write/create files
edit.ts        → Precise file edits
apply_patch.ts → Multi-file patches
grep.ts        → Search file contents
find.ts        → Find files by pattern
ls.ts          → List directories
```

Pi Agent's `bash` tool spawns a local shell process, captures output, handles timeouts, and truncates large outputs. It knows nothing about Docker or sandboxing - that's OpenClaw's job.

### Session Management

Pi Agent handles:
- Creating and resuming agent sessions
- Storing conversation history (`.jsonl` files)
- The agentic loop: prompt → LLM → parse response → execute tools → repeat
- Streaming responses back to the caller
- Auto-compaction when context exceeds limits

### Compaction

Pi Agent automatically compacts session history when it approaches the model's context window limit:
1. Detects context overflow from provider errors
2. Summarizes older turns using `generateSummary()`
3. Retries the prompt with compressed history

OpenClaw extends this with pre-compaction memory flushes (giving the agent a chance to save important context to files first).

## What OpenClaw Adds

### 1. The System Prompt

Pi Agent accepts a system prompt as a parameter. OpenClaw builds an extensive one with 20+ sections:

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts
const appendPrompt = buildEmbeddedSystemPrompt({
  workspaceDir,
  toolNames,
  skillsPrompt,
  contextFiles,       // Bootstrap files
  sandboxInfo,
  reactionGuidance,
  // ... many more params
});
const systemPromptOverride = createSystemPromptOverride(appendPrompt);

({ session } = await createAgentSession({
  // ... Pi Agent receives the prompt
}));
applySystemPromptOverrideToSession(session, systemPromptText);
```

The system prompt defines OpenClaw's identity, available tools, skills, safety guidelines, workspace, and behavior rules. See [System Prompt Sections](reference/system-prompt-sections.md) for the full template.

### 2. Custom Tools

OpenClaw adds tools Pi Agent doesn't have (`src/agents/tools/` and `src/agents/pi-tools.ts`):

| Tool | Purpose |
|------|---------|
| `browser` | Control a web browser (Playwright) |
| `canvas` | Interactive canvas for visualizations |
| `message` | Send messages to channels |
| `cron` | Schedule jobs and reminders |
| `nodes` | Interact with paired devices (mobile, IoT) |
| `web_search` | Search the web (Brave API) |
| `web_fetch` | Fetch and parse web pages |
| `gateway` | Self-update and config management |
| `sessions_spawn` | Spawn isolated sub-agent sessions |
| `sessions_list` | List active sessions |
| `sessions_send` | Send messages to other sessions |
| `subagents` | List, steer, or kill sub-agent runs |
| `session_status` | Show usage and model info |
| `image` | Analyze images with vision models |
| `memory_search` | Semantic search over MEMORY.md |
| `memory_get` | Read specific lines from memory files |
| `tts` | Text-to-speech generation |

These tools are registered alongside Pi Agent's core tools when creating a session.

### 3. Sandboxing

Pi Agent runs commands locally by default. OpenClaw adds optional Docker sandboxing (`src/agents/sandbox/`):

```
src/agents/sandbox/
├── docker.ts           # Container lifecycle (start, stop, exec)
├── runtime-status.ts   # Sandbox state detection
├── elevated.ts         # Elevated exec (host commands with approval)
├── workspace.ts        # Mount workspace into container
└── browser.ts          # Browser control inside sandbox
```

When sandboxing is enabled:
1. OpenClaw spins up a Docker container
2. Mounts the workspace (read-only or read-write based on config)
3. Routes Pi Agent's `bash`/`exec` calls to run inside the container
4. Applies tool policies (some tools blocked in sandbox mode)
5. Optionally allows "elevated" commands to run on the host with user approval

The agent receives sandbox info in its system prompt so it knows the environment constraints, but tool interception is transparent.

### 4. Channels and Routing

Pi Agent has no concept of Telegram, WhatsApp, or any messaging platform. OpenClaw adds:

- **Channel integrations**: Telegram (grammY), WhatsApp (Baileys), Discord (discord.js), Signal, Slack, iMessage, LINE, Google Chat
- **Message routing**: Which agent handles which chat (see [Agent Routing](agent-routing.md))
- **Session keys**: `agent:main:telegram:group:12345`
- **Identity linking**: Same person across channels shares session
- **Reply delivery**: Formatting and sending responses back

### 5. Plugins and Hooks

OpenClaw supports a plugin system with hooks that can modify agent behavior:

| Hook | When | Purpose |
|------|------|---------|
| `before_prompt_build` | Before system prompt is assembled | Inject RAG context, modify prompt |
| `before_agent_start` | Before agent loop begins | Legacy hook for prompt context |
| `session_memory` | On `/new` command | Save session summary to memory files |

Plugins can inject retrieval-augmented context, customize prompts per session, or add custom post-processing.

### 6. Everything Else

- **Skills system**: Loadable capabilities from workspace and system directories
- **Memory tools**: Semantic search and recall from MEMORY.md + memory/*.md
- **Bootstrap files**: SOUL.md, AGENTS.md, TOOLS.md, USER.md, IDENTITY.md injection
- **Bootstrap budget**: Truncation with warnings when files are too large
- **Heartbeats**: Periodic proactive checks
- **Session write locks**: Prevent concurrent modifications
- **Provider-specific handling**: Turn validation, tool ID sanitization, thinking block filtering
- **ACP integration**: Anthropic Computer Player harness spawning

## How They Connect

The integration point is `src/agents/pi-embedded-runner/`. This is OpenClaw's wrapper around Pi Agent:

```
src/agents/pi-embedded-runner/
├── run.ts                    # Main entry: runEmbeddedPiAgent()
├── run/
│   ├── attempt.ts            # Creates Pi Agent session, passes system prompt
│   ├── images.ts             # Image handling for prompts
│   └── history-image-prune.ts # Prune old images from history
├── history.ts                # History limiting and retrieval
├── system-prompt.ts          # Builds system prompt for Pi Agent
├── google.ts                 # Google/Gemini-specific handling
├── thinking.ts               # Reasoning block handling
├── skills-runtime.ts         # Skills discovery and loading
├── extensions.ts             # Extension factory registration
└── cache-ttl.ts              # Prompt caching TTL injection
```

The key function is `runEmbeddedPiAgent()` which:
1. Resolves sandbox context
2. Discovers and loads skills
3. Builds bootstrap files with truncation handling
4. Assembles the system prompt
5. Acquires session write lock
6. Runs plugin hooks
7. Creates Pi Agent session with tools
8. Applies provider-specific stream wrappers
9. Sends the user's message
10. Streams the response back
11. Handles compaction if needed

## Provider-Specific Handling

OpenClaw wraps Pi Agent's stream function to handle provider quirks:

| Provider | Handling |
|----------|----------|
| Google/Gemini | Turn validation (user→assistant alternation), tool schema sanitization |
| Anthropic | Turn validation, thinking block dropping for follow-up calls |
| Ollama | Native `/api/chat` streaming, `num_ctx` injection for context size |
| OpenAI | WebSocket streaming for Responses API, function call downgrading |
| Mistral | Tool call ID format sanitization |

These wrappers intercept outbound and inbound messages transparently.

## Key Takeaways

1. **Pi Agent is a library, not an application** - It provides the agent loop and core file tools
2. **OpenClaw is the application** - It provides identity, channels, custom tools, sandboxing, plugins
3. **The system prompt is OpenClaw's** - Pi Agent just receives it as a parameter
4. **Sandboxing is OpenClaw's** - Pi Agent runs locally; OpenClaw intercepts and routes to Docker
5. **Plugins extend OpenClaw** - They hook into the context assembly and agent lifecycle
6. **You could swap Pi Agent** - In theory, OpenClaw could use a different agent runtime (though it's tightly integrated)

## Code References

| What | Where |
|------|-------|
| Pi Agent integration | `src/agents/pi-embedded-runner/` |
| System prompt builder | `src/agents/system-prompt.ts` |
| System prompt params | `src/agents/system-prompt-params.ts` |
| Custom tool creation | `src/agents/pi-tools.ts` |
| Tool definitions | `src/agents/tools/` |
| Sandboxing | `src/agents/sandbox/` |
| Session management | `src/agents/session-write-lock.ts` |
| Provider handling | `src/agents/pi-embedded-runner/google.ts` |
| Agent scope | `src/agents/agent-scope.ts` |
| Pi Agent source | `pi-mono/packages/coding-agent/` |
| Pi Agent core tools | `pi-mono/packages/coding-agent/src/core/tools/` |
