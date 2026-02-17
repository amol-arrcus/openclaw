# OpenClaw - AI Agent Design

**Generated**: 2026-02-15
**Focus**: How the AI agent is designed — the core loop, prompt assembly, tool execution, streaming, and orchestration

---

## The Core Concept

The AI agent in OpenClaw is a **stateful, event-driven loop** built on top of the Pi SDK (`@mariozechner/pi-coding-agent`). It follows the classic **Think → Act → Observe** pattern:

1. **Think**: The model receives the system prompt + conversation history and generates a response
2. **Act**: If the response includes tool calls, each tool is executed
3. **Observe**: Tool results are injected back into the conversation, and the model gets another turn

This loop repeats until the model produces a final text response with no tool calls, or a stop condition is reached (timeout, abort, max turns).

---

## Agent Session Lifecycle

### Starting a Run

```
User sends message
  │
  ├─ 1. CREATE SESSION: createAgentSession()
  │     → Model configuration (provider, model, thinking level)
  │     → Tool list (filtered by policy)
  │     → Session manager (conversation history)
  │     → Settings manager (agent config)
  │     → Stream function (handles API communication)
  │
  ├─ 2. SUBSCRIBE: subscribeEmbeddedPiSession()
  │     → Attaches event handlers for streaming
  │     → Sets up block chunking for smooth output
  │     → Configures tag stripping (<think>, <final>)
  │
  ├─ 3. PRE-PROMPT HOOKS: before_agent_start
  │     → Plugins can inject context into prompt
  │     → Plugins can prepend content to system prompt
  │
  ├─ 4. IMAGE DETECTION: Auto-load referenced images
  │     → Scan for image paths/URLs in the prompt
  │     → Load as base64 for vision models
  │
  ├─ 5. PROMPT: session.prompt(text, { images })
  │     → Model begins generating response
  │     → Events stream back through subscription
  │     → Tool calls execute inline during streaming
  │
  ├─ 6. COMPACTION WAIT: Wait for any auto-compaction
  │     → If context window was approaching limit, compaction runs
  │     → Wait for it to settle before returning
  │
  └─ 7. POST-PROMPT: Cleanup and finalize
        → Cache TTL tracking
        → Snapshot selection (normal or pre-compaction)
        → agent_end event emitted
```

### A Single Turn in Detail

When `session.prompt()` is called, the Pi SDK sends the conversation to the model and streams back the response. During streaming:

```
Model generates response
  │
  ├─ Text tokens → accumulated, chunked, emitted as message_update events
  │
  ├─ Tool call detected →
  │     ├─ tool_execution_start event
  │     ├─ before_tool_call hook (can modify params or block)
  │     ├─ Tool executes (may stream output → tool_execution_update)
  │     ├─ after_tool_call hook (fire-and-forget)
  │     ├─ tool_execution_end event
  │     └─ Result injected into conversation as tool_result message
  │
  └─ End of response → message_end event
       → If tool calls were made → model gets another turn (loop)
       → If text only → run complete
```

---

## System Prompt Assembly

The system prompt is the **most critical piece** of the agent's behavior. It's assembled dynamically by `buildAgentSystemPrompt()` with up to ~20 sections, depending on context.

### Three Prompt Modes

| Mode | When Used | What's Included |
|------|-----------|----------------|
| **full** | Main agent (direct chat) | All sections — identity, tools, safety, skills, memory, messaging, runtime |
| **minimal** | Sub-agents, cron jobs | Only tooling, workspace, runtime (no skills, memory, docs) |
| **none** | Bare minimum contexts | Just: "You are a personal assistant running inside OpenClaw." |

### Prompt Sections (Full Mode)

The sections are assembled in order, each conditionally included:

```
┌─────────────────────────────────────────────┐
│ 1. IDENTITY                                   │
│    "You are a personal assistant running       │
│     inside OpenClaw."                          │
├─────────────────────────────────────────────┤
│ 2. TOOLING                                     │
│    Lists available tools with summaries.        │
│    Guidance on when/how to use them.            │
│    Notes about background processes.            │
├─────────────────────────────────────────────┤
│ 3. SAFETY                                      │
│    Constitutional AI principles.                │
│    No self-preservation or power-seeking.       │
│    Human oversight requirements.                │
├─────────────────────────────────────────────┤
│ 4. SKILLS (conditional)                        │
│    Available skills from SKILL.md files.        │
│    Instructions: scan, read, follow.            │
├─────────────────────────────────────────────┤
│ 5. MEMORY (conditional)                        │
│    memory_search / memory_get guidance.          │
│    Citation preferences (on/off).               │
├─────────────────────────────────────────────┤
│ 6. CLI QUICK REFERENCE                         │
│    OpenClaw command shortcuts.                   │
├─────────────────────────────────────────────┤
│ 7. MODEL ALIASES (conditional)                 │
│    Human-friendly model name mappings.           │
├─────────────────────────────────────────────┤
│ 8. WORKSPACE                                   │
│    Working directory, containment, commits.      │
├─────────────────────────────────────────────┤
│ 9. DOCS (conditional)                          │
│    Paths to local and online documentation.      │
├─────────────────────────────────────────────┤
│ 10. SANDBOX (conditional)                      │
│     Container paths, elevated exec, browser.     │
├─────────────────────────────────────────────┤
│ 11. USER IDENTITY (conditional)                │
│     Owner contact info.                          │
├─────────────────────────────────────────────┤
│ 12. DATE & TIME (conditional)                  │
│     Current timestamp.                           │
├─────────────────────────────────────────────┤
│ 13. REPLY TAGS                                 │
│     [[reply_to_current]] for native replies.     │
├─────────────────────────────────────────────┤
│ 14. MESSAGING                                  │
│     Cross-session messaging, inline buttons.     │
├─────────────────────────────────────────────┤
│ 15. VOICE / TTS (conditional)                  │
│     Text-to-speech preferences.                  │
├─────────────────────────────────────────────┤
│ 16. GROUP / SUBAGENT CONTEXT (conditional)     │
│     Extra prompt for group or spawned context.   │
├─────────────────────────────────────────────┤
│ 17. REACTIONS (conditional)                    │
│     Emoji reaction guidance (Telegram/Signal).   │
├─────────────────────────────────────────────┤
│ 18. REASONING FORMAT (conditional)             │
│     <think> and <final> tag instructions.        │
├─────────────────────────────────────────────┤
│ 19. PROJECT CONTEXT                            │
│     Injected files: SOUL.md, AGENTS.md, etc.    │
│     SOUL.md triggers persona embodiment.         │
├─────────────────────────────────────────────┤
│ 20. SILENT REPLIES                             │
│     NO_REPLY token usage.                        │
├─────────────────────────────────────────────┤
│ 21. HEARTBEATS                                 │
│     HEARTBEAT_OK health check protocol.          │
├─────────────────────────────────────────────┤
│ 22. RUNTIME                                    │
│     Agent ID, host, OS, model, shell, channel.   │
└─────────────────────────────────────────────┘
```

### What Determines the Sections

- **Prompt mode** (full/minimal/none) — primary filter
- **Available tools** — determines which tool guidance sections appear
- **Agent configuration** — per-agent overrides
- **Runtime context** — sandbox, channel, capabilities
- **Injected workspace files** — SOUL.md, AGENTS.md, USER.md, etc.

---

## Tool Execution Model

### How Tools Are Created

Tools come from **5 sources**, merged into a single list:

| Source | Examples | Notes |
|--------|----------|-------|
| **Pi SDK base tools** | read, write, edit, grep, find, ls | Core file/search tools |
| **Execution tools** | exec, process | Shell commands, TTY, background |
| **OpenClaw native tools** | browser, canvas, nodes, cron, message, memory_search, web_search, image, sessions_spawn, subagents | Platform capabilities |
| **Channel tools** | whatsapp_login | Channel-specific, owner-only |
| **Plugin tools** | MCP server tools | From enabled extensions |

### The 8-Layer Tool Policy Pipeline

Before tools reach the agent, they pass through a **multi-layer policy pipeline** that filters what's available:

```
Layer 1: PROFILE POLICY
  → Predefined tool bundles: minimal, coding, messaging, full
  → Example: "coding" = file tools + runtime + sessions + memory + image

Layer 2: PROVIDER PROFILE POLICY
  → Same as above, but per-provider (e.g., different tools for OpenAI vs Anthropic)

Layer 3: GLOBAL POLICY
  → Top-level tool allowlist/denylist across all agents

Layer 4: GLOBAL PROVIDER POLICY
  → Provider-specific global policy

Layer 5: AGENT POLICY
  → Per-agent tool allowlist/denylist

Layer 6: AGENT PROVIDER POLICY
  → Per-agent, per-provider policy

Layer 7: GROUP POLICY
  → Channel/group-level policy for multi-user contexts

Layer 8: SANDBOX / SUBAGENT POLICY
  → Sandbox: restrict tools in container environments
  → Subagent: auto-deny dangerous tools (gateway, cron, memory, etc.)
```

Each layer can **allow** specific tools/groups, **deny** specific tools/groups, or use **alsoAllow** to add to the parent's allowlist without replacing it.

### Tool Groups

Tools are organized into named groups for easier policy management:

| Group | Tools |
|-------|-------|
| `group:fs` | read, write, edit, apply_patch |
| `group:runtime` | exec, process |
| `group:memory` | memory_search, memory_get |
| `group:web` | web_search, web_fetch |
| `group:sessions` | sessions_list, sessions_history, sessions_send, sessions_spawn, subagents, session_status |
| `group:ui` | browser, canvas |
| `group:automation` | cron, gateway |
| `group:messaging` | message |
| `group:nodes` | nodes |
| `group:openclaw` | All OpenClaw-native tools (excludes plugin tools) |

### Tool Hook Wrapping

After policy filtering, every tool gets wrapped with:
1. **before_tool_call hook** — plugins can modify parameters or block execution
2. **AbortSignal** — cancellation on timeout or user abort
3. **after_tool_call hook** — fire-and-forget logging/side-effects

---

## Streaming Architecture

### Event Types

During an agent turn, events stream through the subscription system:

| Event | When | What It Contains |
|-------|------|-----------------|
| `agent_start` | Run begins | Run metadata |
| `message_start` | Model begins a new response | Turn number |
| `message_update` | Text tokens arrive | Incremental text delta |
| `tool_execution_start` | Tool call initiated | Tool name, arguments |
| `tool_execution_update` | Tool produces output | Streaming output |
| `tool_execution_end` | Tool completes | Result, status |
| `message_end` | Model finishes response | Complete text |
| `agent_end` | Run completes | Final state |
| `auto_compaction_start` | Context window cleanup begins | — |
| `auto_compaction_end` | Context window cleanup ends | — |

### Block Chunking

Raw text deltas arrive token-by-token, which is too granular for most UIs. The **EmbeddedBlockChunker** batches them into smooth chunks:

```
Token stream: "The" " answer" " is" " 42" "." " The" ...
                    ↓ Block Chunker ↓
Chunks:       "The answer is 42." | "The next part..."
```

- Configurable chunk size (e.g., 200 characters)
- Breaks at natural points: sentence end, paragraph end, text end
- Prevents mid-word or mid-sentence cuts

### Tag Stripping

The model may emit special tags that need processing:

| Tag | Purpose | Handling |
|-----|---------|---------|
| `<think>...</think>` | Extended reasoning | Stripped from output (optionally streamed separately) |
| `<final>...</final>` | Final answer (when reasoning is on) | Only content inside is emitted |
| `[[reply_to_current]]` | Reply targeting | Extracted as directive, removed from text |
| `[[media:url]]` | Media attachments | Extracted as media reference |
| `[[audio_as_voice]]` | Voice output flag | Extracted as capability flag |

Tag stripping is **stateful across chunks** — a `<think>` tag opened in one chunk can be closed in a later one.

### Reasoning Modes

| Mode | Behavior |
|------|----------|
| `off` | No `<think>` tags, no extended reasoning |
| `on` | Reasoning happens inside `<think>` tags, stripped from output |
| `stream` | Reasoning streamed to a separate channel (`[Thinking]\n...`) |

---

## Context Management

### The Problem

AI models have finite context windows. Long conversations, large tool outputs, and many turns eventually exceed the limit. OpenClaw handles this with **4 mechanisms**:

### 1. History Limiting

Before sending context to the model, old turns are dropped:
- Configurable max turns per session type (e.g., 50 for DMs)
- Most recent turns kept, oldest dropped
- Applied before each model request

### 2. Tool Result Truncation

Large tool outputs are trimmed:
- Configurable max characters per result (e.g., 100k)
- JSON results pretty-printed then truncated
- Binary data excluded or summarized

### 3. Context Pruning (Opt-in)

An optional microcompact-style system that prunes old tool results:

```
Settings:
  mode: off | cache-ttl | aggressive
  maxCharsPerToolResult: 2000
  toolResultHeadRatio: 0.6        ← Keep 60% head, 40% tail
  pruneAfterTurns: 10             ← Don't prune recent results
```

- Prunes based on age and size
- Keeps head and tail of outputs, discards middle
- Allowlist/denylist per tool (some tools never pruned)

### 4. Auto-Compaction

When the context window approaches its limit, the Pi SDK triggers **auto-compaction**:

```
Context nearing limit
  │
  ├─ 1. MEMORY FLUSH (optional, see doc 10)
  │     → Agent writes important notes before compaction
  │
  ├─ 2. COMPACTION
  │     → Summarizes older conversation turns
  │     → Preserves recent context in full
  │     → Up to 3 retries on failure
  │
  └─ 3. CONTINUE
        → Agent resumes with reduced context
        → If compaction fails/times out → use pre-compaction snapshot
```

The subscription tracks `auto_compaction_start` and `auto_compaction_end` events. If a timeout occurs during compaction, the system falls back to a pre-compaction snapshot to prevent corrupted state.

---

## Hook / Extension Points

Plugins can extend agent behavior through **15 hook points** without modifying core code:

### Agent Lifecycle Hooks

| Hook | When | Can Modify? |
|------|------|-------------|
| `before_agent_start` | Before session.prompt() | Yes — inject systemPrompt, prependContext |
| `agent_end` | After run completes | No (fire-and-forget) |
| `before_compaction` | Before auto-compaction | No (fire-and-forget) |
| `after_compaction` | After compaction finishes | No (fire-and-forget) |
| `before_reset` | Before /new or /reset clears session | No (fire-and-forget) |

### Tool Lifecycle Hooks

| Hook | When | Can Modify? |
|------|------|-------------|
| `before_tool_call` | Before tool executes | Yes — modify params, block execution |
| `after_tool_call` | After tool completes | No (fire-and-forget) |
| `tool_result_persist` | When result is saved to transcript | Yes — modify message (synchronous only) |

### Message Lifecycle Hooks

| Hook | When | Can Modify? |
|------|------|-------------|
| `message_received` | User message arrives | No (fire-and-forget) |
| `message_sending` | Before reply is sent | Yes — modify content, cancel send |
| `message_sent` | After reply delivered | No (fire-and-forget) |

### Session & Gateway Hooks

| Hook | When |
|------|------|
| `session_start` | New session initialized |
| `session_end` | Session closed/deleted |
| `gateway_start` | Gateway daemon starts |
| `gateway_stop` | Gateway daemon stops |

### Execution Model

- **Modifying hooks** run **sequentially** in priority order (higher priority first). Results are merged across handlers.
- **Fire-and-forget hooks** run **in parallel** for performance. Errors are caught and logged.
- Plugins can set their **priority** to control execution order.

---

## Sub-Agent Spawning

### How It Works

The main agent can spawn child agents via the `sessions_spawn` tool:

```
Main agent decides it needs help
  │
  ├─ 1. DEPTH CHECK
  │     → Current depth < maxSpawnDepth? (default: 1)
  │     → Leaf agents cannot spawn children
  │
  ├─ 2. CHILD LIMIT CHECK
  │     → Active children < maxChildrenPerAgent? (default: 5)
  │
  ├─ 3. AGENT ALLOWLIST CHECK
  │     → Can this agent spawn the requested agent ID?
  │
  ├─ 4. SESSION CREATION
  │     → New session key: "agent:{targetId}:subagent:{uuid}"
  │     → Model selection (param > agent config > global > default)
  │     → Thinking level override (optional)
  │
  ├─ 5. SYSTEM PROMPT
  │     → "minimal" mode prompt (tooling + workspace + runtime)
  │     → Plus: requester context, task description, label
  │
  ├─ 6. LAUNCH
  │     → Runs in AGENT_LANE_SUBAGENT priority queue
  │     → deliver: false (no user notification during execution)
  │     → spawnedBy: parent session key
  │
  └─ 7. REGISTRY
        → Tracked in SubagentRunRecord
        → Enables list, steer, kill operations
```

### Parent-Child Relationship

```
Main Agent (depth 0)
  ├─ Sub-agent A (depth 1)
  │     → Cannot spawn children (at maxSpawnDepth)
  │     → Has reduced tool set
  │     → Announces results back to parent on completion
  │
  └─ Sub-agent B (depth 1)
        → Same constraints
        → Independent execution from A
```

### Orchestration Tools

The parent agent manages sub-agents through the `subagents` tool:

| Action | What It Does |
|--------|-------------|
| `list` | Shows active + recent runs (last 30 min), with label, model, runtime, token usage, status |
| `kill` | Aborts a running sub-agent (cascades to descendants), clears queue, marks terminated |
| `steer` | Interrupts current work, injects new instructions, restarts as new run (preserves history) |

### Depth-Aware Tool Policy

| Role | Can Spawn? | Allowed Tools |
|------|-----------|---------------|
| **Orchestrator** (depth < max) | Yes | sessions_spawn, subagents, sessions_list, sessions_history |
| **Leaf** (depth >= max) | No | subagents (read-only), denied: sessions_spawn, sessions_list |
| **All sub-agents** (any depth) | — | Denied: gateway, agents_list, cron, memory_search, memory_get, sessions_send |

---

## Extended Thinking

When enabled, the agent has a "thinking" phase where it reasons before responding:

```
Model receives prompt
  │
  ├─ <think> reasoning phase
  │     → Internal deliberation, planning, analysis
  │     → Not shown to user (unless mode = "stream")
  │     → Tag-stripped from final output
  │
  └─ <final> response phase
        → Actual reply to the user
        → Only this content is delivered
```

### Thinking Budget

Configurable token budget for thinking:

```json5
{
  "agents": {
    "defaults": {
      "thinking": {
        "level": "medium",    // "off" | "low" | "medium" | "high"
        "budget": {
          "low": 2048,
          "medium": 8192,
          "high": 32768
        }
      }
    }
  }
}
```

---

## The Complete Picture

```
┌────────────────────────────────────────────────────────────────┐
│                        USER MESSAGE                              │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│ GATEWAY                                                          │
│  ├─ Route to agent session (by session key)                      │
│  ├─ Queue in lane (serialize per session)                        │
│  └─ Resolve model, tools, system prompt                          │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│ PRE-PROMPT                                                       │
│  ├─ before_agent_start hooks                                     │
│  ├─ Image detection & loading                                    │
│  ├─ System prompt assembly (22 sections)                         │
│  └─ Tool policy pipeline (8 layers)                              │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│ AGENT LOOP (Think → Act → Observe)                               │
│                                                                  │
│  ┌─── THINK ───────────────────────────────────────────────┐    │
│  │ Model processes system prompt + conversation history       │    │
│  │ Optional extended thinking (<think> tags)                  │    │
│  │ Generates text and/or tool calls                          │    │
│  └──────────────────────────────────────────────────────────┘    │
│           │                              │                        │
│      Text only                    Tool calls                      │
│           │                              │                        │
│           │                    ┌─── ACT ──────────────────┐      │
│           │                    │ before_tool_call hook      │      │
│           │                    │ Execute tool               │      │
│           │                    │ after_tool_call hook       │      │
│           │                    │ Inject result              │      │
│           │                    └────────────┬───────────────┘      │
│           │                                 │                      │
│           │                    ┌─── OBSERVE ─────────────┐        │
│           │                    │ Tool result in context    │        │
│           │                    │ Model gets another turn   │        │
│           │                    └────────────┬─────────────┘        │
│           │                                 │                      │
│           │                          (loop back to THINK)          │
│           │                                                        │
│           ▼                                                        │
│  ┌─── DONE ────────────────────────────────────────────────┐    │
│  │ Final text response (no more tool calls)                  │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────┐
│ POST-PROMPT                                                      │
│  ├─ Wait for compaction (if triggered)                           │
│  ├─ agent_end event                                              │
│  ├─ message_sending hook (can modify or cancel)                  │
│  ├─ Deliver to channel (Web UI, Slack, Telegram, etc.)           │
│  └─ message_sent hook                                            │
└────────────────────────────────────────────────────────────────┘
```

---

## For Your Multi-Agent System

The agent design translates to your Python/TS testing system:

1. **Think → Act → Observe loop** — your agents need the same pattern. The model reasons, calls tools (SSH, API, test runners), gets results, and decides what to do next.

2. **Dynamic system prompt** — assemble the prompt from sections based on agent role. Your topology agent gets topology-specific sections; your test runner gets execution-specific ones.

3. **Tool policy pipeline** — restrict what each agent can do. Your helper agent shouldn't have SSH access; your action agent shouldn't create test suites.

4. **Streaming events** — expose tool execution progress to your UI. When an agent runs a test, stream the output in real time.

5. **Sub-agent orchestration** — your orchestrator spawns specialized agents, monitors them via `list`, redirects via `steer`, and cancels via `kill`. Depth limits prevent runaway spawning.

6. **Context management** — long test runs generate massive output. Use tool result truncation (keep head + tail) and compaction to stay within limits.

7. **Hook points** — extensibility without modifying core. Add logging, metrics, or approval gates at `before_tool_call` or `message_sending`.

8. **Lane-based serialization** — one test run per topology at a time, but different topologies in parallel. Exactly matches the queue lane pattern.
