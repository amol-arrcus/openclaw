# OpenClaw - Deep Dive: Event Queue, Agent Logic & Claude Integration

**Generated**: 2026-02-15
**Source**: Exhaustive scan of agent runtime, event system, and Pi SDK integration

---

## How Claude Code is Connected to OpenClaw

OpenClaw does **not** spawn Claude Code as a subprocess. Instead, it **embeds the Pi agent SDK** directly as a Node.js library:

### The Pi SDK Stack

```
@mariozechner/pi-agent-core   — Agent loop, message types, streaming
@mariozechner/pi-ai            — Model registry, provider abstraction, API calls
@mariozechner/pi-coding-agent  — SessionManager, tools, workspace management
```

OpenClaw imports these packages and calls `createAgentSession()` to create an in-process agent that:
1. Talks to the Anthropic API (or any supported provider)
2. Receives streamed responses (text, tool calls, thinking)
3. Executes tools locally
4. Manages conversation transcripts

**This is the same core engine that powers Claude Code CLI** — OpenClaw wraps it with multi-channel routing, session management, and a gateway control plane.

### Key Integration Point

```
src/agents/pi-embedded-runner/run/attempt.ts
  └─→ createAgentSession({
        sessionManager,     // JSONL transcript store
        model,              // Resolved model (claude-opus-4-6, etc.)
        tools,              // 60+ tools from createOpenClawCodingTools()
        systemPrompt,       // Dynamically assembled prompt
        extensions,         // Compaction safeguard, context pruning
      })
  └─→ session.prompt(userMessage)  // Streaming agent loop
```

---

## Event Queue Architecture

### Lane-Based Command Queue

**File**: `src/process/command-queue.ts`

The command queue serializes agent execution per-session while allowing cross-session parallelism:

```
Lane: "session:agent:main:telegram:direct:123"
  ├── Run 1 (in progress) ─── session.prompt() running
  ├── Run 2 (queued) ──────── waiting for Run 1
  └── Run 3 (queued) ──────── waiting for Run 2

Lane: "session:agent:coding:discord:channel:456"
  └── Run 1 (in progress) ─── parallel with above lane
```

**API:**
```typescript
enqueueCommandInLane(lane: string, task: () => Promise<T>, opts?) → Promise<T>
setCommandLaneConcurrency(concurrency: number)  // Default: 1 per lane
```

**Behavior:**
- Each session gets its own lane → one agent turn at a time
- Cross-session runs execute in parallel
- Queue warns if a task waits > 2 seconds
- Subagent lanes are separate from parent lanes

### Active Run Tracking

**File**: `src/agents/pi-embedded-runner/runs.ts`

```typescript
ACTIVE_EMBEDDED_RUNS: Map<sessionId, EmbeddedPiQueueHandle>

setActiveEmbeddedRun(sessionId, handle)    // Register when run starts
clearActiveEmbeddedRun(sessionId)          // Deregister when run ends
queueEmbeddedPiMessage(sessionId, msg)     // Queue new message during streaming
abortEmbeddedPiRun(sessionId)              // Cancel in-flight run
waitForEmbeddedPiRunEnd(sessionId, timeout) // Poll for completion
```

This is how the system knows what's running, can queue follow-up messages, and can abort runs.

### FollowupRun Queue

**File**: `src/auto-reply/queue.ts`

When an inbound message arrives and the agent is already running for that session, a **FollowupRun** is created:

```typescript
createFollowupRun({
  sessionKey,
  agentId,
  provider,
  model,
  command,        // The user's message
  idempotencyKey, // Dedup
})
```

The run is picked up when the current agent turn completes.

---

## Agent Execution Pipeline (Full Detail)

### Step 1: Message Arrives

```
Channel Monitor (Telegram/Discord/etc.)
  → normalizeMessage() → MessageInput
    → dispatchInboundMessage()
```

**OR**

```
WebSocket RPC: agent or chat.send
  → agentHandlers["agent"] in gateway/server-methods/agent.ts
```

### Step 2: Route Resolution

```typescript
// src/routing/resolve-route.ts
resolveAgentRoute({
  channel: "telegram",
  peer: { kind: "direct", id: "123456" },
  accountId: "default"
})
→ {
  agentId: "main",
  sessionKey: "agent:main:telegram:direct:123456",
  matchedBy: "binding.peer"
}
```

### Step 3: Enqueue & Execute

```typescript
// Enqueue into lane-based queue
enqueueCommandInLane(`session:${sessionKey}`, async () => {
  return runReplyAgent(context);
});
```

### Step 4: Agent Turn (Core Loop)

**File**: `src/auto-reply/reply/agent-runner-execution.ts`

```
runAgentTurnWithFallback()
  │
  ├─ 1. Resolve model + provider (with fallback chain)
  │     resolveAgentConfig() → runWithModelFallback()
  │
  ├─ 2. Load session transcript
  │     resolveSessionTranscriptPath()
  │     SessionManager.open() → reads JSONL
  │     limitHistoryTurns() → context window management
  │
  ├─ 3. Build system prompt
  │     resolveAgentPrompt() assembles from:
  │       ├── Tool descriptions (available tools)
  │       ├── Tool call style guidance
  │       ├── Safety guardrails
  │       ├── Skills (SKILL.md loading)
  │       ├── Memory (MEMORY.md)
  │       ├── Workspace context
  │       ├── Messaging options
  │       ├── Runtime info (agent ID, model, OS)
  │       ├── Heartbeat config
  │       └── Custom extraSystemPrompt
  │
  ├─ 4. Execute AI model
  │     runEmbeddedPiAgent()
  │       └─ createAgentSession()
  │            └─ session.prompt(message)
  │                 └─ Streaming response:
  │                      text_delta → onBlockReply()
  │                      tool_execution → onToolResult()
  │                      thinking → onReasoningStream()
  │                      message_end → finalize
  │
  ├─ 5. Tool execution loop (inside Pi SDK)
  │     Each tool_use block:
  │       ├── Validate against tool policy
  │       ├── Invoke tool implementation
  │       ├── Collect result
  │       ├── Feed back to agent
  │       └── Agent decides: more tools or respond
  │
  ├─ 6. Response processing
  │     normalizeStreamingText()
  │       ├── Strip <think>/<thinking> blocks
  │       ├── Sanitize for channel
  │       ├── Skip SILENT_REPLY_TOKEN
  │       └── Extract MEDIA:<url> directives
  │
  ├─ 7. Block reply pipeline
  │     block-reply-pipeline.ts
  │       ├── Coalesce streaming text blocks
  │       ├── Chunk for channel limits (2000-4000 chars)
  │       └── Handle streaming vs discrete blocks
  │
  └─ 8. Delivery
        reply-dispatcher.ts → infra/outbound/deliver.ts
          ├── Send to source channel via plugin
          ├── Mirror to WebChat (if connected)
          └── Update transcript, emit agent events
```

### Step 5: Event Emission

**File**: `src/infra/agent-events.ts`

```typescript
emitAgentEvent({
  runId: "run_abc123",
  stream: "tool" | "lifecycle" | "model" | "error",
  data: { ... }
})
```

Events are broadcast to all WebSocket clients via `registerToolEventRecipient()`.

---

## Streaming Event Subscription (Detail)

**File**: `src/agents/pi-embedded-subscribe.ts` (700 lines)

This is the bridge between Pi SDK's event stream and OpenClaw's callback system:

```typescript
subscribeEmbeddedPiSession({
  session,          // Pi AgentSession
  sessionKey,
  agentId,
  onBlockReply,     // Text chunks for delivery
  onToolResult,     // Tool invocation display
  onReasoningStream,// Thinking blocks
  onSystemMessage,  // System notifications
})
```

### Event Types Handled

| Pi Event | OpenClaw Handler | Action |
|---|---|---|
| `text_delta` | `onBlockReply()` | Accumulate in EmbeddedBlockChunker, flush at breaks |
| `text_end` | Flush chunker | Final text delivery |
| `tool_execution_start` | `onToolResult()` | Show tool card in UI |
| `tool_execution_update` | `onToolResult()` | Update tool output stream |
| `tool_execution_end` | `onToolResult()` | Close tool card |
| `message_end` | Finalize | Close run, update transcript |
| `thinking` | `onReasoningStream()` | Forward reasoning (if enabled) |
| `auto_compaction` | System message | Context overflow recovery |

### Block Chunker

**File**: `src/agents/pi-embedded-block-chunker.ts`

Accumulates streaming text and flushes at natural break points:

```
Text arrives character-by-character:
  "The weather in NYC is..."  ← accumulating
  "\n\n"                      ← paragraph break → FLUSH
  "Temperature: 72°F"        ← accumulating
  [text_end]                  ← force FLUSH
```

Configurable: `minChars`, `idleMs`, break preferences (paragraph > newline > sentence).

### Deduplication

The subscriber tracks sent texts to prevent duplicate delivery when:
- The agent uses the `message` tool to send text AND also outputs the same text
- Repeated `text_end` events
- Multiple block replies for the same content

---

## Multi-Agent & Subagent System

### Agent Configuration

```json5
{
  agents: {
    default: {
      name: "Main",
      workspace: "~/.openclaw/workspace",
      model: "anthropic/claude-opus-4-6",
      thinkingDefault: "high"
    },
    coding: {
      name: "Coder",
      workspace: "~/projects",
      model: "anthropic/claude-sonnet-4-5-20250929"
    }
  }
}
```

### Subagent Spawning

Agents can spawn child agents via the `sessions_spawn` tool:

```
Parent agent (main, depth=0)
  ├── Child agent (research, depth=1)
  └── Child agent (coding, depth=1)
```

**Constraints:**
- `maxSpawnDepth`: 1 (default) — children can't spawn grandchildren
- `maxChildrenPerAgent`: 5 (default) — max concurrent children

**Lifecycle:**
1. Parent calls `sessions_spawn` with task description
2. `SubagentRegistry` tracks the child
3. Child runs independently with its own session
4. Parent can: list, steer (send message), or kill children
5. Child results are available via `sessions_history`

### Model Fallback Chain

```
Session override → Agent config → Global default
  ↓ (if primary fails)
Fallback model 1 → Fallback model 2 → Error
```

---

## System Prompt Assembly

**File**: `src/agents/system-prompt.ts` (673 lines)

The system prompt is **dynamically assembled** from sections:

| Section | Content |
|---|---|
| **Tooling** | Available tools with schemas and descriptions |
| **Tool Call Style** | Narration/explanation guidance |
| **Safety** | Guardrails and restrictions |
| **OpenClaw CLI** | Self-management instructions |
| **Skills** | SKILL.md loading protocol |
| **Memory** | MEMORY.md content (persistent across sessions) |
| **Workspace** | File paths, project context |
| **Docs** | Local + cloud documentation references |
| **Sandbox** | Container/sandbox constraints (if applicable) |
| **Messaging** | Channel options, reply format |
| **Voice** | TTS hints (if voice mode) |
| **Runtime** | Agent ID, model, OS, shell, timezone |
| **Heartbeat** | Proactive mode instructions |
| **Silent Replies** | When to suppress output |
| **Extra** | Per-session `extraSystemPrompt` |

**Prompt Modes:**
- `"full"` — All sections (main agent)
- `"minimal"` — Reduced for subagents (skip Docs, Skills, Update)
- `"none"` — Bare identity line only

---

## Session Persistence

### JSONL Transcript Format

Location: `~/.openclaw/agents/<agentId>/sessions/<sessionKey>.jsonl`

Each line is a JSON object representing a message or event:

```jsonl
{"id":"msg_1","parentId":null,"role":"user","content":"hello","timestamp":1708012800}
{"id":"msg_2","parentId":"msg_1","role":"assistant","content":"Hi!","usage":{"input":10,"output":5}}
{"id":"msg_3","parentId":"msg_2","role":"user","content":"what's the weather?"}
{"id":"msg_4","parentId":"msg_3","role":"assistant","content":[{"type":"tool_use","id":"t1","name":"web_search","input":{"query":"weather NYC"}}]}
{"id":"msg_5","parentId":"msg_4","role":"tool","content":"72°F, sunny","tool_use_id":"t1"}
{"id":"msg_6","parentId":"msg_5","role":"assistant","content":"The weather in NYC is 72°F and sunny."}
```

### Session Operations

| Operation | Method | Description |
|---|---|---|
| Load | `SessionManager.open()` | Read JSONL, build message tree |
| Append | `sessionManager.appendMessage()` | Write to transcript |
| Compact | `compactEmbeddedPiSession()` | Prune + summarize old turns |
| Reset | `sessions.reset` RPC | Clear history |
| Query | `chat.history` RPC | Read message range |

### Write Locking

`src/agents/session-write-lock.ts` prevents concurrent writes to the same session. Only one agent turn can write at a time.

---

## How This Maps to Your Multi-Agent Testing System

### OpenClaw Patterns You Can Reuse

| OpenClaw Pattern | Your Application |
|---|---|
| **Lane-based command queue** | Serialize topology → action → suite → test execution |
| **FollowupRun queue** | Queue discovered test cases during execution |
| **Subagent spawning** | Your 6 agents as child tasks of an orchestrator |
| **Session persistence (JSONL)** | Track state across topology → actions → suites → tests |
| **Event subscription** | Stream progress updates to your UI |
| **Tool system** | Each agent's capabilities as tools (create topology, validate, generate suite, etc.) |
| **Model fallback** | Retry with different models if generation fails |
| **Block chunking** | Stream intermediate results to users |

### Key Differences

| OpenClaw | Your System |
|---|---|
| Generic AI assistant | Domain-specific testing pipeline |
| User-driven (messages) | Task-driven (automated workflow) |
| Session = conversation | Session = full test lifecycle |
| Tools are general-purpose | Tools are domain-specific (device APIs, topology validation) |
| Multi-channel delivery | Single output (test results) |
| Flat agent execution | Sequential pipeline with branching |
