# OpenClaw - Channel Adapters: Web UI & Slack

**Generated**: 2026-02-15
**Focus**: How the Web UI and Slack channel adapters work (conceptual, not code-level)

---

## Overview

OpenClaw uses a **channel plugin system** where each messaging platform implements a standardized contract. This doc covers only the **Web UI** (Control UI / WebChat) and **Slack** adapters.

---

## Web UI (Control UI)

### What It Is

A browser-based dashboard and chat interface built with **Lit web components**. It connects to the OpenClaw gateway via WebSocket and provides:
- Real-time chat with agents
- Session management
- Agent configuration
- Tool execution monitoring
- System status dashboards

### Architecture

```
Browser (Lit Web Components)
  │
  │  WebSocket (persistent)
  │
  ▼
Gateway Server
  │
  │  RPC Protocol v3
  │
  ▼
Agent Engine
```

**Key pattern**: The UI is a **single Lit component** (`OpenClawApp`) with reactive `@state` decorators for all state. No Redux or external state management — Lit's built-in reactivity handles everything.

### Connection Lifecycle

```
1. User opens UI → connect() called
2. WebSocket created to gateway URL
3. Auth payload sent (device identity + token/password)
4. Gateway responds with "hello-ok" frame:
   - Protocol version
   - Features list
   - Initial state snapshot (sessions, agents, channels)
5. UI subscribes to server events:
   - Chat updates (streaming text, tool calls)
   - Agent events (lifecycle, errors)
   - Presence updates
   - Device pairing
6. Connection maintained with reconnection:
   - Exponential backoff: starts 800ms, caps at 15s
   - Event gap detection (alerts user if messages lost)
```

### How Messages Flow: User → Agent → UI

**Sending a message:**

```
1. User types in textarea, presses Send
2. handleSendChat() validates:
   - Detects /stop, /reset, /new commands
   - If chat busy → queues message
   - Supports image attachments (base64)
3. RPC call: chat.send → gateway
   - Payload: sessionKey, message text, attachments, idempotencyKey (UUID)
4. Message added to local UI immediately (optimistic render)
5. Gateway returns runId (UUID for this agent turn)
```

**Receiving the response (streaming):**

```
6. Gateway streams "chat" events via WebSocket:
   - state: "delta" → accumulate text, render incrementally
   - state: "delta" (tool) → show tool card with live output
   - state: "final" → mark complete, add to history
   - state: "error" → show error
   - state: "aborted" → handle gracefully
7. UI renders streaming text with markdown parsing
8. Tool executions shown as interactive cards
9. Queue drained: next queued message sent (if any)
```

### How Tool Executions Are Displayed

When the agent calls a tool (bash, file read, web search, etc.), the UI shows it as an interactive **tool card**:

```
┌─────────────────────────────────┐
│ 🔧 bash                         │
│ Arguments: ls -la /tmp           │
│ ─────────────────────────────── │
│ Output:                          │
│ total 48                         │
│ drwxrwxrwt  12 root ...         │
│ ✓ Completed                      │
└─────────────────────────────────┘
```

- Live streaming of tool output (throttled at 80ms to avoid UI thrashing)
- Large outputs open in a **resizable sidebar panel**
- Up to 50 concurrent tool streams tracked
- Output truncated at 120k chars

### Markdown Rendering

- **Library**: `marked.js` with GitHub Flavored Markdown
- **Safety**: DOMPurify sanitizes all HTML output
- **Performance**: LRU cache of 200 parsed results (avoids re-parsing)
- **Limits**: Messages > 140k chars truncated; > 40k chars rendered as `<pre>` without parsing
- **Code blocks**: Syntax-highlighted with CSS classes

### Session Management

- Each chat has a `sessionKey` identifying the conversation + agent scope
- Sessions persist across browser refreshes (state lives on the gateway)
- `/reset` or `/new` starts a fresh session
- Session dropdown for switching between recent conversations
- Last active session stored in localStorage

### UI Structure

```
OpenClawApp (root Lit component)
  ├── app-gateway.ts     → WebSocket lifecycle, event routing
  ├── app-chat.ts        → Send/queue/abort logic
  ├── app-render.ts      → Layout with tabs
  ├── views/
  │   ├── chat.ts        → Chat interface
  │   ├── agents.ts      → Agent management
  │   ├── sessions.ts    → Session browser
  │   ├── config.ts      → YAML editor
  │   ├── channels.ts    → Channel status
  │   ├── usage.ts       → Token analytics
  │   ├── cron.ts        → Scheduled tasks
  │   ├── skills.ts      → Skill management
  │   ├── logs.ts        → Log viewer
  │   └── ...            → 14 tabs total
  ├── controllers/       → RPC calls to gateway
  └── chat/              → Message normalization, tool extraction, markdown
```

**State flow**: User action → controller (RPC) → gateway responds → event handler updates `@state` → Lit re-renders affected components.

---

## Slack Adapter

### What It Is

A bidirectional channel adapter that bridges Slack conversations with OpenClaw agents. The bot listens for messages in Slack channels/DMs, routes them to agents, and delivers replies back.

### Connection Modes

| Mode | How It Works | When to Use |
|------|-------------|-------------|
| **Socket Mode** | Persistent WebSocket to Slack (via app-level token) | Default, always-on |
| **HTTP Mode** | Slack posts events to your webhook endpoint | When behind load balancer |

### Authentication

| Credential | Purpose |
|-----------|---------|
| **Bot Token** (`xoxb-...`) | Send/receive messages |
| **App Token** (`xapp-...`) | Socket Mode connection |
| **Signing Secret** | Validate webhook authenticity (HTTP mode) |

Multi-account support: multiple Slack workspaces via `channels.slack.accounts`.

### How Messages Are Received

```
Slack event arrives (message or app_mention)
  │
  ├─ Filter out: bot messages, deletions, edits, thread broadcasts
  │
  ├─ prepareSlackMessage() enriches the event:
  │   → Resolve channel type (DM, channel, group DM)
  │   → Check allowlist / access policies
  │   → Load thread history (if in a thread)
  │   → Extract media attachments
  │   → Build session key
  │
  ├─ Classify message:
  │   → DM: subject to dmPolicy (open/pairing/disabled)
  │   → Channel: subject to groupPolicy (open/closed/pairing)
  │   → May require @mention to trigger (requireMention config)
  │
  └─ Route to agent for processing
```

### How Replies Are Sent Back

```
Agent generates response
  │
  ├─ markdownToSlackMrkdwnChunks() converts markdown → Slack markup
  │   → Bold, italic, strikethrough, code blocks
  │   → Links formatted as <URL|label>
  │   → Tables converted to formatted text
  │
  ├─ Chunk if needed (Slack limit: ~4000 chars per message)
  │   → Split modes: "length" (default) or "newline"
  │
  ├─ Threading logic:
  │   → "all": all replies threaded
  │   → "first": first reply threaded, rest in channel
  │   → "off": stay in thread if already in one
  │
  └─ sendMessageSlack() → Slack Web API
      → Supports: text, file uploads, metadata blocks
      → Typing indicator shown while processing
```

### Threading Support

Slack threads are fully supported:

```
Channel message → Agent replies in thread (configurable)
  │
  Thread reply → Agent detects thread context
  │   → Loads thread history (up to 20 messages)
  │   → historyScope: "thread" (thread only) or "channel" (full context)
  │   → Agent responds with thread awareness
  │
  └─ Thread lifecycle managed across multiple turns
```

### Slash Commands

Skills can be exposed as Slack slash commands:

```
User types: /opengpt what's the weather?
  │
  ├─ Slack sends "commands" event
  ├─ parseCommandArgs() parses command + arguments
  ├─ Match to agent command registry
  ├─ Agent runs with command context
  └─ Reply sent back (ephemeral or in-channel)
```

Supports interactive argument menus (buttons for selecting options).

### Channel vs DM Handling

| Context | Policy | Access Control |
|---------|--------|---------------|
| **Direct Message** | `dmPolicy`: open / pairing / disabled | Per-user access |
| **Channel** | `groupPolicy`: open / closed / pairing | Per-channel allowlist |
| **Group DM** | `groupDmEnabled` flag | Optional channel list |

**Access control chain**:
1. Check channel allowed by policy + config
2. Check sender allowed (allowlist, pairing)
3. Check command authorized (if slash command)
4. Route to correct agent + session
5. Apply skill filters + system prompts

### Per-Channel Configuration

Each Slack channel can have its own config:

```json5
{
  "channels": {
    "slack": {
      "channels": {
        "C01234567": {
          "skills": ["web-search", "code-review"],  // Skill allowlist
          "systemPrompt": "You are a helpful coding assistant.",
          "requireMention": true,   // Only respond when @mentioned
          "users": {
            "allowlist": ["U01234567"]
          }
        }
      }
    }
  }
}
```

### Rate Limiting

| Constraint | Mitigation |
|-----------|------------|
| ~1 msg/sec/channel | Message chunking + sequential delivery |
| File upload limits | `mediaMaxBytes` (default 20MB) |
| API rate limits | Queue management, typing indicator deferral |
| Duplicate messages | Dedup cache by message ID |

---

## Key Differences: Web UI vs Slack

| Aspect | Web UI | Slack |
|--------|--------|-------|
| **Connection** | WebSocket (persistent, real-time) | Socket Mode WebSocket or HTTP webhook |
| **Streaming** | Character-by-character streaming | Full message delivery (no streaming) |
| **Rendering** | Rich markdown + tool cards + sidebar | Slack mrkdwn (limited formatting) |
| **Threading** | Single linear chat history | Native Slack threads + channel context |
| **Auth** | Device identity + token/password | Bot token + app token |
| **Session state** | Full transcript on gateway | Limited to configured history window |
| **Tool visibility** | Interactive tool cards with output | Text-only tool summaries |
| **File handling** | Base64 image attachments | Slack file uploads |
| **Rate limits** | None (local connection) | Slack API rate limits apply |

---

## For Your Multi-Agent System

Both adapters demonstrate patterns you can reuse:

**From Web UI:**
- WebSocket-based real-time streaming for test progress
- Tool card pattern for showing device commands and their output
- Session management for tracking test runs
- Queue-based message handling (send while busy)

**From Slack:**
- Channel plugin contract (standardized interface for any frontend)
- Threading for organizing test discussions per topology
- Slash commands for triggering test runs (`/run-suite topology-A`)
- Access control for restricting who can trigger expensive test operations
- Message chunking for large test result outputs
