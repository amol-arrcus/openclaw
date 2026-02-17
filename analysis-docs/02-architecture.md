# OpenClaw - Architecture Document

**Generated**: 2026-02-15
**Source**: Deep codebase scan of openclaw@2026.2.15

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CHAT CHANNELS (Inbound)                      │
│  WhatsApp  Telegram  Discord  Slack  Signal  iMessage  IRC  ...     │
└──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────────────────┘
       │      │      │      │      │      │      │
       ▼      ▼      ▼      ▼      ▼      ▼      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CHANNEL PLUGIN LAYER                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────────────┐  │
│  │ Monitor  │  │Normalize │  │ Outbound Adapters (send/media)   │  │
│  │(inbound) │  │(unified) │  │ Chunking, threading, formatting  │  │
│  └────┬─────┘  └────┬─────┘  └──────────────────────────────────┘  │
│       │              │                                               │
│       ▼              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │           Plugin Registry & Channel Manager                  │    │
│  │   Lifecycle (start/stop), login flows, status probes         │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      GATEWAY (Control Plane)                        │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   HTTP Server     │  │  WebSocket RPC   │  │  Event Broadcast │  │
│  │ (hooks, webhooks, │  │  (protocol v3)   │  │  (to all clients)│  │
│  │  control UI, API) │  │  40+ methods     │  │                  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                      │                      │           │
│  ┌────────┴──────────────────────┴──────────────────────┴────────┐  │
│  │                   RPC Method Dispatcher                        │  │
│  │  agent | chat.send | sessions.* | config.* | channels.* | ... │  │
│  └────────────────────────────┬──────────────────────────────────┘  │
│                               │                                     │
│  ┌────────────────────────────┴──────────────────────────────────┐  │
│  │                 Routing & Session Layer                        │  │
│  │  resolveAgentRoute() → agentId + sessionKey                   │  │
│  │  Bindings: channel + peer + guild + roles → agent             │  │
│  │  Session store: JSONL transcripts per agent/session            │  │
│  └────────────────────────────┬──────────────────────────────────┘  │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     AGENT EXECUTION ENGINE                          │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              Auto-Reply / Dispatch System                      │ │
│  │  dispatchInboundMessage() → resolveRoute() → enqueue          │ │
│  │  FollowupRun queue → runReplyAgent()                          │ │
│  └───────────────────────────┬────────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────┴────────────────────────────────────┐ │
│  │           Pi Embedded Runner (Agent Core)                      │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐  │ │
│  │  │ Session Mgr  │  │ Model Select │  │ System Prompt Builder│  │ │
│  │  │ (JSONL store)│  │ (fallback)   │  │ (dynamic assembly)   │  │ │
│  │  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘  │ │
│  │         │                │                      │              │ │
│  │         ▼                ▼                      ▼              │ │
│  │  ┌───────────────────────────────────────────────────────────┐ │ │
│  │  │              createAgentSession() (Pi SDK)                │ │ │
│  │  │  session.prompt() → streaming agent loop                  │ │ │
│  │  │  Tool execution → result collection → next turn           │ │ │
│  │  └──────────────────────────┬────────────────────────────────┘ │ │
│  │                             │                                  │ │
│  │  ┌──────────────────────────┴────────────────────────────────┐ │ │
│  │  │               Tool System (60+ tools)                     │ │ │
│  │  │  File ops | Bash | Web search | Browser | Messaging       │ │ │
│  │  │  Sessions spawn | Cron | Gateway control | Channel tools  │ │ │
│  │  └───────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │          Lane-Based Command Queue                              │ │
│  │  Per-session concurrency lanes → serialized agent turns        │ │
│  │  Parallel across sessions                                      │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │          Subagent System                                       │ │
│  │  sessions_spawn → child agents with depth/count limits         │ │
│  │  Registry tracking, announce, steering, kill                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     DELIVERY LAYER (Outbound)                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Reply Dispatcher → Block Reply Pipeline → Channel Outbound  │   │
│  │  Text chunking → media extraction → channel-specific format  │   │
│  │  Mirror to WebChat if connected                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                      │
│  ┌─────────┐ ┌───────────┐ ┌──────────┐ ┌──────┐ ┌─────────────┐  │
│  │ Web UI  │ │ CLI       │ │ macOS    │ │ iOS  │ │ Android     │  │
│  │ (Lit)   │ │ (Commander│ │ (SwiftUI)│ │(node)│ │ (node)      │  │
│  │         │ │  + WS)    │ │          │ │      │ │             │  │
│  └─────────┘ └───────────┘ └──────────┘ └──────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Architecture Deep Dive

### 1. Gateway (Control Plane)

The Gateway is the central orchestrator. It runs as a single process that:

- **Serves HTTP**: Webhooks, plugin endpoints, Control UI static files, OpenAI-compatible API
- **Manages WebSocket connections**: Custom RPC protocol (v3) with TypeBox validation
- **Broadcasts events**: Real-time updates to all connected clients
- **Routes messages**: Channel + peer + guild + roles → agent mapping

**Key files:**
- `src/gateway/server-http.ts` — HTTP server factory
- `src/gateway/server/ws-connection.ts` — WebSocket lifecycle
- `src/gateway/server/ws-connection/message-handler.ts` — RPC routing
- `src/gateway/server-methods.ts` — Handler dispatcher (40+ methods)
- `src/gateway/auth.ts` — Device identity + token/password auth

**WebSocket Protocol:**
```
Client → Gateway:  { type: "req", id: uuid, method: "agent", params: {...} }
Gateway → Client:  { type: "res", id: uuid, ok: true, payload: {...} }
Gateway → Client:  { type: "event", event: "agent.event", payload: {...} }
```

### 2. Channel Plugin System

Every messaging platform is a **ChannelPlugin** implementing a standardized contract:

```typescript
ChannelPlugin = {
  id: ChannelId,
  meta: ChannelMeta,              // Display info
  capabilities: ChannelCapabilities, // Feature matrix
  config: ChannelConfigAdapter,   // Account management
  outbound: ChannelOutboundAdapter, // Send text/media
  gateway: ChannelGatewayAdapter, // Lifecycle (start/stop/login)
  security: ChannelSecurityAdapter, // DM policy, allowlists
  status: ChannelStatusAdapter,   // Health probes
  // + streaming, threading, messaging, actions, etc.
}
```

**Delivery modes:**
- `direct` — Plugin calls channel API directly (Telegram, Discord, iMessage)
- `gateway` — Via gateway service (WhatsApp Web)
- `hybrid` — Supports both

**Core channels** (in `src/`): telegram, whatsapp, discord, slack, signal, imessage, line
**Extension channels** (in `extensions/`): matrix, msteams, googlechat, irc, feishu, nostr, bluebubbles, mattermost, zalo, twitch, nextcloud-talk

### 3. Routing & Session Layer

**Routing** maps inbound messages to agents:

```
resolveAgentRoute({channel, peer, accountId, guildId, roles})
  → { agentId, sessionKey, matchedBy }
```

Matching priority (first wins):
1. Binding: channel + peer + parent-peer
2. Binding: channel + guild + roles (Discord)
3. Binding: channel + guild
4. Binding: channel + team (Slack)
5. Binding: channel + accountId
6. Binding: channel match
7. Default agent

**Session key format:** `agent:{agentId}:{scope}`
- `agent:main:main` — default session
- `agent:main:direct:{peerId}` — DM with specific user
- `agent:coding:slack:channel:{id}` — Slack channel session

**Session storage:** JSONL files at `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

### 4. Agent Execution Engine

**Execution flow:**

```
Inbound message / RPC request
  → dispatchInboundMessage() or gateway RPC handler
    → resolveAgentRoute()
      → enqueue FollowupRun in command queue
        → runReplyAgent()
          → runAgentTurnWithFallback()
            → runEmbeddedPiAgent()
              → createAgentSession() (Pi SDK)
                → session.prompt() (streaming agent loop)
                  → Tool execution loop
                    → Block reply pipeline
                      → Channel delivery
```

**Concurrency model:**
- **Per-session lanes**: One agent turn at a time per session (serialized)
- **Cross-session**: Multiple sessions execute in parallel
- **Subagents**: Max depth 1, max 5 children per agent (configurable)

### 5. Tool System

60+ tools organized by category:

| Category | Tools |
|---|---|
| File operations | read, write, edit, apply_patch, grep, find, ls |
| Execution | exec, process (bash/shell) |
| Web | web_search, web_fetch, browser (Playwright) |
| Messaging | message (send to channels) |
| Session mgmt | sessions_spawn, sessions_send, sessions_list, sessions_history |
| Scheduling | cron (reminders, wake events) |
| Subagents | list, steer, kill sub-agent runs |
| Gateway | self-management |
| Channel-specific | Discord actions, Telegram actions, Slack actions |

**Tool policy system**: Per-profile allowlists/denylists, sandbox policies, group/channel policies, authorization checks.

### 6. UI Layer (Control UI)

- **Framework**: Lit 3.3.2 (Web Components) + Vite 7.3.1
- **Single component**: `OpenClawApp` with `@state()` decorators for all state
- **Communication**: WebSocket RPC exclusively (no REST)
- **14 tabs**: Chat, Overview, Channels, Instances, Sessions, Usage, Cron, Agents, Skills, Nodes, Config, Debug, Logs

**Chat message flow:**
1. User types → `handleSendChat()` → enqueues with UUID
2. API call `chat.send` over WebSocket
3. Optimistic render in chat history
4. Streaming responses via `agent.event` WebSocket frames
5. Block chunking, markdown rendering, tool card display

### 7. Configuration System

Config lives at `~/.openclaw/openclaw.json` (or YAML). Key sections:

```json5
{
  agents: {
    default: { name: "...", workspace: "...", model: "..." },
    defaults: { models: {...}, heartbeat: {...}, subagents: {...} }
  },
  channels: {
    telegram: { token: "...", allowFrom: [...] },
    discord: { token: "...", replyToMode: "threads" },
    whatsapp: { allowFrom: ["+1..."] }
  },
  bindings: [
    { match: { channel: "discord", guildId: "..." }, agent: "moderator" }
  ],
  gateway: { port: 9001, auth: { mode: "password" } }
}
```

### 8. Infrastructure Services

| Service | Purpose | Key files |
|---|---|---|
| **Daemon** | Background gateway process | `src/daemon/` |
| **Cron** | Scheduled agent tasks | `src/cron/` |
| **Memory** | Persistent agent memory (MEMORY.md) | `src/memory/` |
| **Media** | Image/audio/video pipeline | `src/media/` |
| **Browser** | Playwright automation | `src/browser/` |
| **Skills** | Remote skill execution on nodes | `src/infra/skills-remote.ts` |
| **Pairing** | Device pairing with approval | `src/pairing/` |
| **Logging** | Structured subsystem logging | `src/logging/` |
| **Security** | Auth, rate limiting, device identity | `src/security/` |

## Data Flow Diagram

```
USER sends "what's the weather?" via Telegram
  │
  ├─1→ Telegram Bot receives update
  ├─2→ TelegramMonitor parses & normalizes → MessageInput
  ├─3→ dispatchInboundMessage()
  │      ├─ resolveAgentRoute() → agentId="main", sessionKey="agent:main:telegram:direct:123"
  │      └─ enqueue FollowupRun
  ├─4→ runReplyAgent() picks up from queue
  │      ├─ Load session transcript (JSONL)
  │      ├─ Append user message
  │      ├─ runEmbeddedPiAgent() → Anthropic API call
  │      ├─ Agent uses tools: web_search("weather NYC")
  │      ├─ Agent generates text response
  │      └─ createReplyDispatcher().dispatch()
  ├─5→ Block reply pipeline
  │      ├─ Text chunking (4000 char limit for Telegram)
  │      ├─ HTML formatting
  │      └─ Send via grammy Bot API
  └─6→ Update transcript, emit agent events to WebSocket clients
```

## Security Architecture

| Layer | Mechanism |
|---|---|
| **Device identity** | Ed25519 keypair, challenge-response |
| **Authentication** | Token or password auth |
| **Device pairing** | Approval workflow |
| **Rate limiting** | Per-IP + scope |
| **Channel allowlists** | Per-channel sender whitelists |
| **Tool policy** | Per-profile, per-sandbox allowlists |
| **Origin check** | CORS for Control UI |
| **Command gating** | Elevated command authorization |
