# Agent Context and Runtime

What happens when your message reaches the agent - how context is assembled and sent to the LLM.

## Overview

Once your message is routed to an agent (typically `agent:main:main` for the default agent), the runtime assembles all the context needed to generate a meaningful response. This includes your message, conversation history, system instructions, and workspace context files.

Clawdbot uses [Pi Agent](https://github.com/badlogic/pi-mono) (`@mariozechner/pi-coding-agent`) as its agent runtime, wrapped in `pi-embedded-runner` to integrate with sessions and channels. Pi Agent handles the agentic loop - sending prompts to the LLM, parsing tool calls, executing tools, and continuing until the agent produces a final response.

This doc explains what goes into that context and how it flows through the runtime.

```mermaid
graph LR
    subgraph routing [Routing]
        A[Message Arrives] --> B[Route to Agent]
    end
    
    subgraph assembly [Context Assembly]
        B --> C[System Prompt]
        B --> D[Session History]
        B --> E[Your Message]
    end
    
    subgraph llmLoop [LLM Loop]
        C & D & E --> F[Send to LLM]
        F --> G[Response]
        G --> H{Tools?}
        H -->|Yes| I[Execute Tools]
        I --> F
        H -->|No| J[Final Response]
    end
    
    subgraph memory [Memory - just files]
        MEM[(memory files)]
    end
    
    I -.->|memory_search| MEM
    I -.->|"write/edit"| MEM
    
    subgraph completion [Turn Completion]
        J --> K[Save to Session]
        K --> L[Deliver Reply]
        L --> M[Compaction Check]
    end
```

## Context Assembly

When you send a message, the runtime assembles several pieces of context before calling the LLM:

```mermaid
graph TB
    subgraph assembled [Final Prompt to LLM]
        F[Complete Context]
    end
    
    subgraph components [Context Components]
        A[Your Message] --> F
        B[System Prompt] --> F
        C[Session History] --> F
    end
    
    subgraph sysPromptDetails [System Prompt Includes]
        B1[Available Tools]
        B2[Loaded Skills]
        B3[Workspace Info]
        B4[Bootstrap Files]
        B5[Runtime Info]
    end
    
    subgraph bootstrapDetails [Bootstrap Files]
        D1[AGENTS.md]
        D2[SOUL.md]
        D3[TOOLS.md]
    end
```

### 1. Your Message

Your actual message text, plus any media (images, files) you attached. The message is wrapped with metadata:

```
[Telegram From YourName 2024-01-15 10:30:00] Did the test implementations complete? What's the status?
```

**No additional context is injected.** There's no automatic "remember X, Y, Z" prepended to your message. Memory recall happens via tool calls during the agent loop (see [Memory System](#memory-system)).

### 2. System Prompt

The system prompt tells the LLM who it is and what it can do. Built by `buildAgentSystemPrompt()`:

| Section | Content |
|---------|---------|
| Identity | Base identity line ("You are Clawdbot...") |
| Tools | Available tools with descriptions |
| Skills | Loaded skills from workspace |
| Workspace | Current working directory, notes |
| User Identity | Owner info for personalization |
| Runtime | Agent ID, host OS, model in use |
| Guidance | Reply formatting, tool call style, etc. |
| Bootstrap Files | Workspace context files (see below) |

**Bootstrap Files** - These workspace files customize the agent's behavior and are injected into the system prompt:

| File | Purpose |
|------|---------|
| `AGENTS.md` | Agent behavior instructions |
| `SOUL.md` | Personality and communication style |
| `TOOLS.md` | Tool-specific guidance |

Bootstrap files are truncated if too large (70% head + 20% tail, max 20k chars).

For a complete breakdown of all 23 sections that can appear in the system prompt, see [System Prompt Sections](reference/system-prompt-sections.md).

### 3. Session History

Previous conversation turns from the session file. Each turn includes:
- Role (user/assistant)
- Content (message text)
- Tool calls and results (if any)
- Timestamps

History is loaded from the session's `.jsonl` file and provides conversation continuity.

For the full construction flow, see [Session History Construction](session-history-construction.md).

## Agent Execution Flow

Once context is assembled, it flows through the agent runtime:

```mermaid
graph LR
    A[Context Assembled] --> B[Pi Agent Session]
    B --> C[LLM Provider]
    C --> D[Response Stream]
    D --> E[Tool Execution]
    E -->|Loop if needed| B
    D --> F[Save to Session]
    F --> G[Deliver Reply]
```

1. **Context Assembled** - System prompt (with bootstrap files), history, your message
2. **Pi Agent Session** - Creates or resumes a Pi Agent session
3. **LLM Provider** - Sends to Claude, GPT, or local model
4. **Response Stream** - Streams response tokens back
5. **Tool Execution** - If the LLM requests tools, they're executed and results fed back
6. **Save to Session** - Appends the turn to session history
7. **Deliver Reply** - Sends response back to the channel

### What Pi Agent Does vs What Clawdbot Does

Pi Agent is a generic coding agent library. Clawdbot wraps it and adds everything that makes it "Clawdbot":

| Layer | Responsibility |
|-------|----------------|
| **Pi Agent** | Core file tools (`bash`, `read`, `write`, `edit`, `grep`, `find`, `ls`), LLM session management, tool execution loop |
| **Clawdbot** | System prompt, channels (Telegram, WhatsApp...), custom tools (`browser`, `message`, `cron`...), sandboxing, routing |

The system prompt is entirely Clawdbot's - Pi Agent receives it as a parameter. Sandboxing (Docker execution) is also Clawdbot's - Pi Agent just runs commands locally by default.

For a deeper dive into this architecture, see [Clawdbot vs Pi Agent Runtime](clawdbot-vs-pi-agent.md).

### Memory System

Memory is just **files + instructions** - no special learning mechanism.

**Reading memory**: Clawdbot provides `memory_search` and `memory_get` tools. The system prompt instructs: *"Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search..."*

**Writing memory**: Uses Pi Agent's standard `write` and `edit` tools. The `AGENTS.md` bootstrap file instructs the agent:
```
## Memory system (recommended)
- Daily log: memory/YYYY-MM-DD.md
- Long-term memory: memory.md for durable facts, preferences, decisions
```

When you say "remember I prefer TypeScript":
1. Agent follows AGENTS.md instructions
2. Agent calls `write` or `edit` to update `memory.md` or `memory/YYYY-MM-DD.md`
3. This is a normal tool call during the agent loop - not post-processing

There's no automatic preference extraction. Nothing is written unless the agent decides to (based on your request and AGENTS.md guidance).

## Turn Completion

When the agent produces its final response (no more tool calls), the turn is complete:

1. **Response appended** to session `.jsonl` file
2. **Reply delivered** to the originating channel (Telegram, WhatsApp, etc.)
3. **Usage tracked** (tokens, model, provider) in session metadata
4. **Compaction check** - Pi Agent checks if context is near the limit and proactively compacts history if needed

There's no automatic learning or preference extraction after each turn. For details on session persistence and memory mechanisms, see [Session Persistence](conversation-history.md).

## Key Code Locations

| Component | File |
|-----------|------|
| Pi Agent runner | `src/agents/pi-embedded-runner/run.ts` |
| System prompt | `src/agents/system-prompt.ts` |
| Session history | `src/agents/pi-embedded-runner/history.ts` |
| Bootstrap files | `src/agents/bootstrap-files.ts` |
| Agent execution | `src/agents/pi-embedded-runner/run/attempt.ts` |

## Summary

When you send a message:

1. **Routing** determines the agent and session (`agent:main:main`)
2. **Context assembly** gathers system prompt (with bootstrap files), session history, your message
3. **Pi Agent** sends everything to the LLM
4. **Response** streams back, tools execute if needed (loop until final response)
5. **Turn completes** - response saved, reply delivered

The context window the LLM sees = system prompt + session history + your message.
