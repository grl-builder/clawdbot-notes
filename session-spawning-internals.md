# Session Spawning: Under the Hood

This document tells the story of how OpenClaw creates, manages, and coordinates multiple sessions - particularly subagents and cron jobs. If you've ever used `sessions_spawn` or wondered how cron jobs run in isolation, this is the technical deep-dive.

## The Cast of Characters

Before we trace the journey, let's meet the key players:

| Component | Role |
|-----------|------|
| **Session Store** | JSON file (`sessions.json`) that persists session metadata |
| **Command Lanes** | Separate execution queues for different workload types |
| **Gateway** | The central daemon that coordinates all agent activity |
| **Subagent Registry** | In-memory map tracking spawned child sessions |
| **Session Entry** | Metadata about a session (ID, model, last activity, etc.) |

## Chapter 1: How Sessions Are Created and Stored

### The Session Store File

Sessions are persisted in a JSON5 file, typically at `~/.config/openclaw/sessions.json` (or per-agent at `~/.config/openclaw/agents/{agentId}/sessions.json`). The store is a simple key-value map:

```typescript
// Conceptual structure
Record<string, SessionEntry>
```

Each `SessionEntry` contains:

```typescript
type SessionEntry = {
  sessionId: string;           // UUID for this session's transcript file
  updatedAt: number;           // Last activity timestamp
  sessionFile?: string;        // Path to the .jsonl transcript
  spawnedBy?: string;          // Parent session key (for subagents)
  label?: string;              // Human-readable label (e.g., "docs-helper")
  
  // Model/thinking overrides
  modelOverride?: string;
  thinkingLevel?: string;
  
  // Routing context
  lastChannel?: string;        // Last delivery channel
  lastTo?: string;             // Last recipient
  lastAccountId?: string;
  
  // Group context (inherited by subagents)
  groupId?: string;
  groupChannel?: string;
  space?: string;
  
  // ... plus many more optional fields
};
```

### Session Key Anatomy

Session keys follow a hierarchical structure:

```
agent:{agentId}:{scope}:{identifier}
```

Examples:
- `agent:main:main` - Main session for default agent
- `agent:main:subagent:550e8400-e29b-41d4-a716-446655440000` - A spawned subagent
- `cron:daily-summary` - A cron job's session

When a subagent is spawned, its key is constructed as:
```javascript
const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;
```

### Persistence Mechanics

The session store uses a lock-based write strategy:

```typescript
async function updateSessionStore(storePath, mutator) {
  return await withSessionStoreLock(storePath, async () => {
    // 1. Load store from disk (bypassing cache)
    const store = loadSessionStore(storePath, { skipCache: true });
    
    // 2. Apply mutation
    const result = await mutator(store);
    
    // 3. Write atomically (tmp file → rename)
    await saveSessionStoreUnlocked(storePath, store);
    
    return result;
  });
}
```

The lock file (`sessions.json.lock`) prevents race conditions when multiple processes update sessions simultaneously.

## Chapter 2: How Spawned Sessions Execute

### The Command Lane System

OpenClaw uses a **lane-based queue system** to manage concurrency. Think of lanes as separate highways for different types of traffic:

```typescript
enum CommandLane {
  Main = "main",        // User-initiated chat messages
  Cron = "cron",        // Scheduled jobs
  Subagent = "subagent", // Spawned background tasks
  Nested = "nested"     // Tool-initiated agent calls
}
```

Each lane is a FIFO queue with configurable concurrency:

```typescript
type LaneState = {
  lane: string;
  queue: QueueEntry[];
  active: number;      // Currently running tasks
  maxConcurrent: number; // Max parallel executions
  draining: boolean;
};
```

### When You Call `sessions_spawn`

Here's what happens step by step:

#### 1. Tool Invocation
```typescript
// The LLM calls sessions_spawn
{
  task: "Review the PR and update documentation",
  label: "docs-review",
  model: "claude-sonnet",
  agentId: "main"
}
```

#### 2. Session Key Generation
```typescript
const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;
// e.g., "agent:main:subagent:a1b2c3d4-..."
```

#### 3. Model/Thinking Configuration
If model or thinking level overrides are specified, they're applied via `sessions.patch`:
```typescript
await callGateway({
  method: "sessions.patch",
  params: {
    key: childSessionKey,
    model: resolvedModel
  }
});
```

#### 4. Gateway Agent Request
The actual work is dispatched to the Gateway:
```typescript
await callGateway({
  method: "agent",
  params: {
    message: task,
    sessionKey: childSessionKey,
    idempotencyKey: childIdem,
    deliver: false,              // Don't auto-deliver to channel
    lane: AGENT_LANE_SUBAGENT,   // Route to subagent lane
    extraSystemPrompt: childSystemPrompt,
    spawnedBy: spawnedByKey      // Link to parent
  }
});
```

#### 5. Registration in Subagent Registry
```typescript
registerSubagentRun({
  runId: childRunId,
  childSessionKey,
  requesterSessionKey: requesterInternalKey,
  requesterOrigin,              // Channel/to/accountId to announce back to
  task,
  cleanup: "keep" | "delete",   // What to do with session after completion
  label
});
```

The registry is an in-memory `Map<runId, SubagentRunEntry>` that's also persisted to disk for crash recovery.

### Same Process, Different Queue

**Important:** Spawned sessions run in the **same Gateway process** as the parent. They don't spin up separate workers or processes. The isolation is logical, not physical:

- Each run gets its own session transcript (`.jsonl` file)
- Each run uses the subagent lane queue
- Concurrency is controlled by `subagents.maxConcurrent` config

```typescript
// Default: 2 subagents can run in parallel
const maxConcurrent = resolveSubagentMaxConcurrent(cfg);
setCommandLaneConcurrency(CommandLane.Subagent, maxConcurrent);
```

## Chapter 3: How `sessions_send` Delivers Messages

When one session needs to send a message to another running session, `sessions_send` provides inter-session communication:

### Resolution Phase

First, the target session is resolved by key or label:

```typescript
// By explicit key
const target = await resolveSessionReference({
  sessionKey: "agent:main:subagent:abc123"
});

// Or by label (searches session store)
const resolved = await callGateway({
  method: "sessions.resolve",
  params: { label: "docs-helper" }
});
```

### Message Injection

The message is sent via the Gateway's `agent` method with the `Nested` lane:

```typescript
await callGateway({
  method: "agent",
  params: {
    message: message,
    sessionKey: resolved.key,
    idempotencyKey,
    deliver: false,
    channel: INTERNAL_MESSAGE_CHANNEL,
    lane: AGENT_LANE_NESTED  // Or AGENT_LANE_SUBAGENT
  }
});
```

### Waiting for Response

If the caller wants to wait for the response:

```typescript
const wait = await callGateway({
  method: "agent.wait",
  params: {
    runId,
    timeoutMs: waitMs
  }
});

if (wait?.status === "ok") {
  // Read the latest assistant reply from the session transcript
  const reply = await readLatestAssistantReply({
    sessionKey: resolved.key
  });
}
```

### Sandboxing

For sandboxed sessions, visibility is restricted:

```typescript
// Sandboxed sessions can only see sessions they spawned
const restrictToSpawned = opts?.sandboxed === true 
  && visibility === "spawned" 
  && !!requesterInternalKey;

if (restrictToSpawned) {
  // Check via sessions.list with spawnedBy filter
  const visible = await listSessions({
    spawnedBy: requesterInternalKey
  });
  if (!visible.includes(targetKey)) {
    return { status: "forbidden" };
  }
}
```

## Chapter 4: How Results Get Announced Back

When a subagent completes, the result needs to flow back to the originating session's channel.

### The Announce Flow

```typescript
async function runSubagentAnnounceFlow(params) {
  // 1. Wait for child run to complete
  if (!await waitForEmbeddedPiRunEnd(childSessionId, settleTimeoutMs)) {
    return false; // Still running
  }

  // 2. Read the final reply from child's transcript
  const reply = await readLatestAssistantReply({
    sessionKey: params.childSessionKey
  });

  // 3. Resolve where to announce (from requesterOrigin)
  const announceTarget = await resolveAnnounceTarget({
    sessionKey: params.requesterSessionKey
  });

  // 4. Deliver to the original channel
  await deliverOutboundPayloads({
    channel: announceTarget.channel,
    to: announceTarget.to,
    payloads: [{ text: formattedAnnouncement }]
  });

  // 5. Optionally clean up child session
  if (params.cleanup === "delete") {
    await deleteSession(params.childSessionKey);
  }
}
```

### The Announcement Format

```
✅ Task completed (label: docs-review)
Session: agent:main:subagent:abc123
Duration: 45s

[Summary of what the subagent accomplished]
```

### Lifecycle Events

The Gateway emits lifecycle events that trigger announce flows:

```typescript
onAgentEvent((evt) => {
  if (evt.stream === "lifecycle" && evt.data?.phase === "end") {
    const entry = subagentRuns.get(evt.runId);
    if (entry && !entry.cleanupCompletedAt) {
      runSubagentAnnounceFlow(entry);
    }
  }
});
```

## Chapter 5: Cron Jobs and Isolated Sessions

Cron jobs are similar to subagents but with some key differences.

### Session Key Structure

```
cron:{jobId}                    // Base session for the cron job
agent:{agentId}:cron:{jobId}    // With agent prefix
agent:{agentId}:cron:{jobId}:run:{sessionId}  // Per-run isolation
```

### Isolation Model

Each cron run can create a fresh session:

```typescript
const runSessionKey = baseSessionKey.startsWith("cron:") 
  ? `${agentSessionKey}:run:${runSessionId}` 
  : agentSessionKey;
```

This means:
- The base cron session persists configuration
- Each run gets its own transcript
- History doesn't accumulate across runs (unless configured otherwise)

### Execution Flow

```typescript
async function runCronIsolatedAgentTurn(params) {
  // 1. Resolve agent configuration
  const agentId = resolveAgentId(params.job);
  
  // 2. Resolve or create session
  const cronSession = resolveCronSession({
    cfg: params.cfg,
    sessionKey: agentSessionKey
  });
  
  // 3. Build the command with timestamp
  const timeLine = `Current time: ${formattedTime} (${userTimezone})`;
  const command = `[cron:${job.id} ${job.name}] ${message}\n${timeLine}`;
  
  // 4. Run via the cron lane
  await enqueueCommandInLane(CommandLane.Cron, async () => {
    await runEmbeddedPiAgent({
      command,
      sessionKey: runSessionKey,
      // ... other params
    });
  });
  
  // 5. Optionally deliver results
  if (deliveryRequested) {
    await deliverOutboundPayloads({
      channel: delivery.channel,
      to: delivery.to,
      payloads: [result]
    });
  }
}
```

### Lane Configuration

Cron jobs use the `Cron` lane with its own concurrency:

```typescript
// From config: agents.defaults.maxConcurrent
setCommandLaneConcurrency(CommandLane.Cron, resolveAgentMaxConcurrent(cfg));
```

## The Big Picture

Here's how it all fits together:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Gateway Process                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Main Lane  │  │ Subagent    │  │  Cron Lane  │              │
│  │  (chat msgs)│  │ Lane        │  │  (scheduled)│              │
│  │  max: 1     │  │ max: 2      │  │  max: 1     │              │
│  └─────┬───────┘  └──────┬──────┘  └──────┬──────┘              │
│        │                 │                │                      │
│        └────────────┬────┴────────────────┘                      │
│                     │                                            │
│              ┌──────▼──────┐                                     │
│              │ Agent Core  │◄──── pi-coding-agent               │
│              │ (LLM calls) │                                     │
│              └──────┬──────┘                                     │
│                     │                                            │
│  ┌──────────────────▼───────────────────┐                       │
│  │           Session Store               │                       │
│  │  (sessions.json + *.jsonl transcripts)│                       │
│  └───────────────────────────────────────┘                       │
│                                                                  │
│  ┌───────────────────────────────────────┐                       │
│  │        Subagent Registry              │                       │
│  │  (in-memory + disk persistence)       │                       │
│  └───────────────────────────────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Single Process**: All sessions run in the same Gateway process. Isolation is logical via separate session keys and transcripts.

2. **Queue-Based Concurrency**: Command lanes prevent overwhelming the LLM API while allowing parallel work.

3. **Parent-Child Linking**: The `spawnedBy` field creates a tree of session relationships.

4. **Event-Driven Completion**: Lifecycle events (`start`, `end`, `error`) trigger announce flows and cleanup.

5. **Persistence for Recovery**: Both the session store and subagent registry are persisted, allowing recovery after crashes.

6. **Sandboxing**: Spawned sessions can be restricted to only see sessions they created.

## Related Source Files

| Area | File |
|------|------|
| Session store | `src/config/sessions/store.ts` |
| Session keys | `src/config/sessions/session-key.ts` |
| Command lanes | `src/process/command-queue.ts` |
| sessions_spawn tool | `src/agents/tools/sessions-spawn-tool.ts` |
| sessions_send tool | `src/agents/tools/sessions-send-tool.ts` |
| Announce flow | `src/agents/tools/sessions-announce-target.ts` |
| Subagent registry | `src/agents/subagent-registry.ts` |
| Cron isolated run | `src/cron/isolated-agent/run.ts` |
| Gateway agent handler | `src/gateway/server-methods/agent.ts` |
