# OpenClaw - How Agents Work with the UI

**Generated**: 2026-02-15
**Source**: Deep scan of UI components, gateway RPC, and event streaming

---

## Overview

The UI-Agent interaction in OpenClaw follows a **WebSocket RPC + Event Streaming** pattern:

```
┌─────────────────────────────────────────────────────┐
│                   Control UI (Browser)               │
│  ┌──────────────────────────────────────────────┐   │
│  │ GatewayBrowserClient (WebSocket)             │   │
│  │  ├── send RPC requests (chat.send, agent)    │   │
│  │  ├── receive RPC responses                   │   │
│  │  └── receive event streams (agent.event)     │   │
│  └───────────────────────┬──────────────────────┘   │
│          ┌───────────────┼───────────────┐          │
│  ┌───────▼───────┐ ┌────▼────┐ ┌────────▼────────┐ │
│  │ Chat View     │ │Sessions │ │ Tool Stream     │ │
│  │ (messages,    │ │ View    │ │ (live tool      │ │
│  │  attachments) │ │         │ │  output cards)  │ │
│  └───────────────┘ └─────────┘ └─────────────────┘ │
└────────────────────────┬────────────────────────────┘
                         │ WebSocket
                         ▼
┌─────────────────────────────────────────────────────┐
│                   Gateway Server                     │
│  ┌──────────────────────────────────────────────┐   │
│  │ WebSocket Connection Handler                  │   │
│  │  ├── Authenticate (device identity + token)   │   │
│  │  ├── Route RPC to method handlers             │   │
│  │  └── Broadcast events to all clients          │   │
│  └───────────────────────┬──────────────────────┘   │
│                          │                           │
│  ┌───────────────────────▼──────────────────────┐   │
│  │ Agent Execution Engine                        │   │
│  │  ├── Pi SDK (createAgentSession)              │   │
│  │  ├── Tool execution loop                      │   │
│  │  ├── Streaming text/tool/thinking events      │   │
│  │  └── Reply dispatcher → channel delivery      │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## Chat Message Flow (UI → Agent → UI)

### 1. User Sends a Message

**UI Side** (`ui/src/ui/controllers/chat.ts`):

```
User types message → clicks Send
  → handleSendChat()
    → Enqueue message with UUID in chatQueue
    → Optimistic render in chat history (shows immediately)
    → WebSocket RPC: chat.send({
        message: "what's the weather?",
        sessionKey: "agent:main:main",
        attachments: [],     // Optional image/audio
        idempotencyKey: uuid
      })
```

### 2. Gateway Processes the Request

**Gateway Side** (`src/gateway/server-methods/chat.ts`):

```
chat.send handler receives RPC
  → Validate ChatSendParams
  → Resolve or create SessionEntry
  → Append user message to transcript
  → Register ChatRunState in ChatRunRegistry
  → Start agent execution (non-blocking)
  → Immediately respond: { ok: true, runId: "run_xyz" }
```

### 3. Agent Executes (Streaming)

```
runEmbeddedPiAgent()
  → session.prompt(message)
    → Anthropic API call (streaming)
      → text_delta events → onBlockReply()
      → tool_use events → onToolResult()
      → thinking events → onReasoningStream()
```

### 4. Events Stream Back to UI

For each streaming event, the gateway broadcasts a WebSocket event frame:

```json
{
  "type": "event",
  "event": "agent.event",
  "payload": {
    "runId": "run_xyz",
    "type": "text_delta",
    "text": "The weather in "
  }
}
```

```json
{
  "type": "event",
  "event": "agent.event",
  "payload": {
    "runId": "run_xyz",
    "type": "tool_execution_start",
    "toolName": "web_search",
    "toolInput": {"query": "weather NYC today"}
  }
}
```

### 5. UI Renders in Real-Time

**UI Side** (`ui/src/ui/app-chat.ts` + `app-tool-stream.ts`):

```
WebSocket onEvent("agent.event")
  → Route by payload.type:
    text_delta → Append to streaming message bubble
    tool_execution_start → Create tool card (collapsible)
    tool_execution_update → Update tool card output
    tool_execution_end → Close tool card
    message_end → Finalize message, update transcript
```

---

## WebSocket Connection Lifecycle

### Connection Setup

**File**: `ui/src/ui/gateway.ts` — `GatewayBrowserClient`

```
1. Client opens WebSocket to gateway URL
2. Gateway sends: connect.challenge { nonce: "abc123" }
3. Client responds: connect {
     clientName: "openclaw-control-ui",
     clientMode: "webchat",
     token: "...",         // or password
     devicePublicKey: "...",  // Ed25519
     signature: "...",       // Signed nonce
   }
4. Gateway validates, responds: hello-ok {
     protocol: 3,
     features: [...],
     snapshot: { health, channels, presence },
     deviceToken: "..."
   }
5. Connection established → RPC + Events active
```

### Auto-Reconnect

```
Disconnect detected
  → Wait 800ms (exponential backoff up to 15s)
  → Reconnect
  → Re-authenticate
  → Re-subscribe to events
  → Refresh UI state (sessions, channels, etc.)
```

---

## Chat UI Architecture

### State Model

All state lives on the `OpenClawApp` component via `@state()` decorators:

```typescript
// Chat state
@state() chatMessage = "";           // Current input text
@state() chatMessages: ChatMessage[] = []; // Conversation history
@state() chatStream = "";            // Currently streaming text
@state() chatRunId: string | null;   // Active run ID
@state() chatSending = false;        // Send in progress
@state() chatQueue: QueuedMessage[]; // Pending messages
@state() chatSessionKey = "";        // Active session
@state() chatAttachments: ChatAttachment[]; // File attachments
```

### Message Rendering Pipeline

```
Raw messages from chat.history RPC
  → message-normalizer.ts: Normalize content items
  → grouped-render.ts: Group consecutive same-role messages
  → For each group:
    ├── Avatar + timestamp header
    ├── For each message in group:
    │   ├── Text content → markdown.ts → marked → dompurify → HTML
    │   ├── Tool calls → tool-cards.ts → collapsible cards
    │   └── Images → inline display
    └── Streaming indicator (if active run)
```

### Tool Cards

When the agent uses a tool, the UI renders a collapsible card:

```
┌─ 🔧 web_search ────────────────────────────┐
│ Input: {"query": "weather NYC today"}        │
│ ─────────────────────────────────────────── │
│ Output:                                      │
│ Temperature: 72°F, Sunny                     │
│ Humidity: 45%                                │
│ Wind: 8 mph NW                               │
└──────────────────────────────────────────────┘
```

Tool output is streamed via `app-tool-stream.ts`:
- Limited to 50 concurrent tool streams
- Truncated at 120KB per stream
- Synced to chat UI asynchronously

### Session Switching

```
User selects different session from Sessions tab
  → setSessionKey(newKey)
    → chatMessages = []
    → Load chat.history for new session
    → Update URL hash
    → Refresh session metadata
```

---

## Multi-Channel Message Delivery

When an agent responds, the reply goes to the **source channel** AND optionally mirrors to the Web UI:

```
Agent response
  → reply-dispatcher.ts
    ├── Channel delivery:
    │   ├── Telegram: grammy Bot API → HTML formatted
    │   ├── Discord: discord.js → Discord markdown
    │   ├── WhatsApp: baileys → WhatsApp formatted
    │   ├── Slack: @slack/bolt → Slack blocks
    │   └── iMessage: imsg CLI → plain text
    │
    └── WebChat mirror:
        └── Broadcast agent.event to all WebSocket clients
```

### Channel-Specific Formatting

| Channel | Text Limit | Format | Threading |
|---|---|---|---|
| Telegram | 4000 chars | HTML | reply-to, forum topics |
| Discord | 2000 chars | Discord markdown | threads |
| WhatsApp | 4000 chars | WhatsApp markdown | reply-to |
| Slack | (large) | Blocks/mrkdwn | threads |
| iMessage | 4000 chars | Plain text | — |
| Web UI | unlimited | Markdown (marked) | session-based |

---

## Agent Identity in the UI

Each agent has an identity displayed in the UI:

```typescript
// Fetched via agent.identity.get RPC
{
  agentId: "main",
  name: "Pi",
  avatar: "🦞",     // Or image URL
  emoji: "🦞"
}
```

The UI shows the avatar next to assistant messages, in the header, and in session lists.

---

## Configuration UI → Agent

The Config tab provides two modes:

**Form Mode**: Structured fields with JSON Schema validation
```
Gateway Port: [18789]
Telegram Bot Token: [●●●●●●●]  (sensitive)
Agent Model: [claude-opus-4-6 ▼]
AllowFrom: [+1555..., +1666...]
```

**Raw YAML Mode**: Direct YAML editing with syntax highlighting

Changes flow: `config.set` RPC → Gateway reloads → Agent picks up new config on next run.

---

## Real-Time Updates Summary

| Mechanism | Used For | Frequency |
|---|---|---|
| WebSocket events | Agent streaming, chat, channel status | Real-time |
| WebSocket RPC | All API calls (chat, config, sessions, etc.) | On-demand |
| Polling (5s) | Nodes tab | While visible |
| Polling (2s) | Logs tab | While visible |
| Polling (3s) | Debug tab | While visible |

No REST API or Server-Sent Events — everything goes through the WebSocket.

---

## Key Takeaways for Building a Similar UI

1. **WebSocket-first architecture** — Single connection for both RPC and streaming events
2. **Optimistic UI** — Show user messages immediately, don't wait for server
3. **Event-driven streaming** — Stream agent output as it arrives, don't batch
4. **Tool cards** — Show tool execution as visual cards, not raw JSON
5. **Session-based navigation** — Each conversation is a switchable session
6. **State on component** — All UI state on one root component (simple, no Redux)
7. **Markdown rendering** — Use `marked` + `dompurify` for safe HTML
8. **Auto-reconnect** — WebSocket disconnects are normal, handle gracefully
