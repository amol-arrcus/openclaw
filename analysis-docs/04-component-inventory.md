# OpenClaw - Component Inventory

**Generated**: 2026-02-15
**Source**: Deep codebase scan of openclaw@2026.2.15

---

## Core Components

### Gateway Server

| Component | Location | Purpose |
|---|---|---|
| HTTP Server Factory | `src/gateway/server-http.ts` | Creates HTTP/HTTPS server, routes requests |
| WebSocket Handler | `src/gateway/server/ws-connection.ts` | Connection lifecycle, upgrade handling |
| Message Handler | `src/gateway/server/ws-connection/message-handler.ts` | RPC message routing |
| RPC Dispatcher | `src/gateway/server-methods.ts` | Routes methods to handlers |
| Method Registry | `src/gateway/server-methods-list.ts` | Available methods/events list |
| Auth | `src/gateway/auth.ts` | Device identity + token/password |
| Rate Limiter | `src/gateway/auth-rate-limit.ts` | Per-IP request limiting |
| Event Broadcast | `src/gateway/server-broadcast.ts` | Push events to WebSocket clients |
| Health State | `src/gateway/server/health-state.ts` | Cached health snapshots |
| Channel Manager | `src/gateway/server-channels.ts` | Start/stop channels, runtime state |
| Chat Registry | `src/gateway/server-chat-registry.ts` | In-flight chat run tracking |
| Chat State | `src/gateway/server-chat.ts` | Chat run state management |
| Boot Runner | `src/gateway/boot.ts` | Execute BOOT.md on startup |

### RPC Handlers (40+)

| Handler | File | Key Methods |
|---|---|---|
| Agent | `server-methods/agent.ts` | `agent`, `agent.identity.get`, `agent.wait` |
| Agents CRUD | `server-methods/agents.ts` | `agents.list/create/update/delete`, `agents.files.*` |
| Chat | `server-methods/chat.ts` | `chat.send`, `chat.history`, `chat.abort`, `chat.inject` |
| Send | `server-methods/send.ts` | `send` (direct message delivery) |
| Sessions | `server-methods/sessions.ts` | `sessions.list/patch/delete/reset/compact/usage` |
| Channels | `server-methods/channels.ts` | `channels.status/logout` |
| Config | `server-methods/config.ts` | `config.get/set/patch/apply/schema` |
| Connect | `server-methods/connect.ts` | WebSocket handshake |
| Health | `server-methods/health.ts` | `health`, `status` |
| Device | (various) | `device.pair.list/approve/reject`, `device.token.rotate` |
| Cron | (various) | `cron.list/add/update/remove/run` |
| Skills | (various) | `skills.status/update/bins` |
| Nodes | (various) | `node.list/invoke/pair.*` |
| Logs | (various) | `logs.tail` |
| Models | (various) | `models.list` |

### Protocol Layer

| Component | Location | Purpose |
|---|---|---|
| Protocol Validator | `src/gateway/protocol/index.ts` | TypeBox-based frame validation |
| Agent Schema | `src/gateway/protocol/schema/agent.ts` | AgentParams, AgentWait schemas |
| Frame Schema | `src/gateway/protocol/schema/frames.ts` | Request/Response/Event frames |
| Error Codes | `src/gateway/protocol/schema/error-codes.ts` | Standardized error codes |

---

## Agent Runtime Components

### Pi Embedded Runner (Core Execution)

| Component | Location | Purpose |
|---|---|---|
| Run Lifecycle | `src/agents/pi-embedded-runner/run.ts` | Main agent run management |
| Run Attempt | `src/agents/pi-embedded-runner/run/attempt.ts` | `createAgentSession()` invocation |
| Active Runs | `src/agents/pi-embedded-runner/runs.ts` | `ACTIVE_EMBEDDED_RUNS` Map tracking |
| Concurrency Lanes | `src/agents/pi-embedded-runner/lanes.ts` | Per-session serialization |
| Session Compaction | `src/agents/pi-embedded-runner/compact.ts` | Context overflow recovery |
| History Manager | `src/agents/pi-embedded-runner/history.ts` | Context window limits |
| Model Resolver | `src/agents/pi-embedded-runner/model.ts` | Provider + model selection |
| SM Cache | `src/agents/pi-embedded-runner/session-manager-cache.ts` | SessionManager instances |

### Event Subscription & Streaming

| Component | Location | Purpose |
|---|---|---|
| Event Subscriber | `src/agents/pi-embedded-subscribe.ts` | Pi session event → OpenClaw callbacks (700 lines) |
| Block Chunker | `src/agents/pi-embedded-block-chunker.ts` | Text accumulation + paragraph-aware flushing |
| Tool Stream | (UI) `ui/src/ui/app-tool-stream.ts` | Tool output streaming to UI |

### Tool System

| Component | Location | Purpose |
|---|---|---|
| Tool Creator | `src/agents/pi-tools.ts` | `createOpenClawCodingTools()` — 60+ tools (465 lines) |
| Tool Adapter | `src/agents/pi-tool-definition-adapter.ts` | Bridge pi-agent-core ↔ pi-coding-agent |
| Tool Policy | `src/agents/tool-policy.ts` | Allowlists, denylists, sandbox policies |
| Tool Display | `src/agents/tool-display-common.ts` | Tool output formatting |
| Bash Tools | `src/agents/bash-tools.ts` | Shell command execution |
| Channel Tools | `src/agents/channel-tools.ts` | Channel-specific agent tools |
| Sessions Spawn | `src/agents/tools/sessions-spawn-tool.ts` | Subagent creation |

### Agent Configuration & Identity

| Component | Location | Purpose |
|---|---|---|
| System Prompt | `src/agents/system-prompt.ts` | Dynamic prompt assembly (673 lines) |
| Model Selection | `src/agents/model-selection.ts` | Model resolution + fallback |
| Agent Scope | `src/agents/agent-scope.ts` | Agent ID resolution |
| Agent Paths | `src/agents/agent-paths.ts` | Workspace/session file paths |
| Identity | `src/agents/identity.ts` | Name, emoji, avatar |
| Auth Profiles | `src/agents/auth-profiles.ts` | OAuth/API key management |
| Workspace Templates | `src/agents/workspace-templates.ts` | Bootstrap files (AGENTS.md, etc.) |

### Subagent System

| Component | Location | Purpose |
|---|---|---|
| Subagent Registry | `src/agents/subagent-registry.ts` | Active child tracking |
| Subagent Announce | `src/agents/subagent-announce.ts` | Child notifications |
| Spawn Tool | `src/agents/tools/sessions-spawn-tool.ts` | `sessions_spawn` tool impl |

---

## Reply Engine Components

| Component | Location | Purpose |
|---|---|---|
| Inbound Dispatch | `src/auto-reply/dispatch.ts` | Route inbound messages to agents |
| FollowupRun Queue | `src/auto-reply/queue.ts` | Queue agent runs |
| Agent Runner | `src/auto-reply/reply/agent-runner.ts` | Main reply driver |
| Execution Loop | `src/auto-reply/reply/agent-runner-execution.ts` | `runAgentTurnWithFallback()` |
| Memory Loader | `src/auto-reply/reply/agent-runner-memory.ts` | Load MEMORY.md |
| Abort Handler | `src/auto-reply/reply/abort.ts` | Cancel in-flight runs |
| Block Pipeline | `src/auto-reply/reply/block-reply-pipeline.ts` | Coalesce streaming blocks |
| Stream Processor | `src/auto-reply/reply/block-streaming.ts` | Process text_delta events |
| Reply Dispatcher | `src/auto-reply/reply/reply-dispatcher.ts` | Route replies to channels |
| Template Context | `src/auto-reply/templating.ts` | MsgContext (100+ fields) |

---

## Routing & Sessions

| Component | Location | Purpose |
|---|---|---|
| Route Resolver | `src/routing/resolve-route.ts` | Channel + peer → agentId + sessionKey |
| Session Key Builder | `src/routing/session-key.ts` | Format: `agent:{id}:{scope}` |
| Bindings | `src/routing/bindings.ts` | Config binding match cache |
| Session Store | `src/config/sessions.ts` | JSONL file management |
| Main Session | `src/config/sessions/main-session.ts` | Default session resolution |
| Input Provenance | `src/sessions/input-provenance.ts` | Message origin tracking |
| Send Policy | `src/sessions/send-policy.ts` | Delivery policy |
| Command Queue | `src/process/command-queue.ts` | Lane-based concurrency (150 lines) |

---

## Channel Plugin Components

### Core Abstractions

| Component | Location | Purpose |
|---|---|---|
| Plugin Interface | `src/channels/plugins/types.plugin.ts` | Full ChannelPlugin contract |
| Core Types | `src/channels/plugins/types.core.ts` | Base types |
| Adapter Types | `src/channels/plugins/types.adapters.ts` | All adapter contracts |
| Plugin Catalog | `src/channels/plugins/catalog.ts` | Available plugins |
| Plugin Loader | `src/channels/plugins/load.ts` | Lazy loading with cache |
| Channel Registry | `src/channels/registry.ts` | Core 8 channels |
| Channel Dock | `src/channels/dock.ts` | Metadata & adapters |

### Per-Channel Implementations

| Channel | Core Location | Extension | Files |
|---|---|---|---|
| Telegram | `src/telegram/` | `extensions/telegram/` | 97 |
| Discord | `src/discord/` | `extensions/discord/` | 49 |
| Slack | `src/slack/` | `extensions/slack/` | 37 |
| Signal | `src/signal/` | `extensions/signal/` | 29 |
| LINE | `src/channels/line/` | `extensions/line/` | 43 |
| iMessage | `src/imessage/` | `extensions/imessage/` | 18 |
| WhatsApp | `src/whatsapp/` | `extensions/whatsapp/` | 6 |
| Matrix | — | `extensions/matrix/` | — |
| MS Teams | — | `extensions/msteams/` | — |
| Google Chat | — | `extensions/googlechat/` | — |
| IRC | — | `extensions/irc/` | — |
| Feishu | — | `extensions/feishu/` | — |
| Mattermost | — | `extensions/mattermost/` | — |

### Channel Infrastructure

| Component | Location | Purpose |
|---|---|---|
| Normalizers | `src/channels/plugins/normalize/*.ts` | Target ID normalization per channel |
| Onboarding | `src/channels/plugins/onboarding/*.ts` | Setup wizard per channel |
| Outbound | `src/channels/plugins/outbound/` | Send pipeline |
| Actions | `src/channels/plugins/actions/` | Message actions per channel |
| Agent Tools | `src/channels/plugins/agent-tools/` | Channel tools for agents |
| Status Issues | `src/channels/plugins/status-issues/` | Diagnostic checks |
| Group Mentions | `src/channels/plugins/group-mentions.ts` | Mention handling |

---

## UI Components (Control UI)

### Core Application

| Component | Location | Purpose |
|---|---|---|
| App Component | `ui/src/ui/app.ts` | `OpenClawApp` (LitElement, all state) |
| Renderer | `ui/src/ui/app-render.ts` | Page rendering dispatch |
| Gateway Client | `ui/src/ui/gateway.ts` | `GatewayBrowserClient` (WebSocket RPC) |
| Storage | `ui/src/ui/storage.ts` | localStorage persistence |
| Navigation | `ui/src/ui/navigation.ts` | URL ↔ tab routing |
| Theme | `ui/src/ui/theme.ts` | Light/dark/system |

### Views (14 tabs)

| View | File | Description |
|---|---|---|
| Chat | `ui/src/ui/views/chat.ts` | Message interface |
| Overview | `ui/src/ui/views/overview.ts` | Dashboard status |
| Channels | `ui/src/ui/views/channels.ts` | Channel config |
| Instances | `ui/src/ui/views/instances.ts` | Presence monitoring |
| Sessions | `ui/src/ui/views/sessions.ts` | Conversation management |
| Usage | `ui/src/ui/views/usage.ts` | Token/cost analytics |
| Cron | `ui/src/ui/views/cron.ts` | Scheduled tasks |
| Agents | `ui/src/ui/views/agents.ts` | Agent management |
| Skills | `ui/src/ui/views/skills.ts` | Skill management |
| Nodes | `ui/src/ui/views/nodes.ts` | Compute nodes |
| Config | `ui/src/ui/views/config.ts` | YAML config editor |
| Debug | `ui/src/ui/views/debug.ts` | Debug utilities |
| Logs | `ui/src/ui/views/logs.ts` | Log viewer |

### Controllers (API + State)

| Controller | File | Purpose |
|---|---|---|
| Chat | `ui/src/ui/controllers/chat.ts` | Send/receive messages |
| Sessions | `ui/src/ui/controllers/sessions.ts` | Session CRUD |
| Config | `ui/src/ui/controllers/config.ts` | Config read/write |
| Devices | `ui/src/ui/controllers/devices.ts` | Pairing management |
| Channels | `ui/src/ui/controllers/channels.ts` | Channel status |
| Agents | `ui/src/ui/controllers/agents.ts` | Agent CRUD |
| Skills | `ui/src/ui/controllers/skills.ts` | Skill management |
| Cron | `ui/src/ui/controllers/cron.ts` | Cron jobs |
| Nodes | `ui/src/ui/controllers/nodes.ts` | Node management |
| Debug | `ui/src/ui/controllers/debug.ts` | Debug APIs |
| Logs | `ui/src/ui/controllers/logs.ts` | Log streaming |

### Chat Subsystem

| Component | File | Purpose |
|---|---|---|
| Message Normalizer | `ui/src/ui/chat/message-normalizer.ts` | Normalize message format |
| Grouped Render | `ui/src/ui/chat/grouped-render.ts` | Group by role |
| Message Extract | `ui/src/ui/chat/message-extract.ts` | Extract text content |
| Tool Cards | `ui/src/ui/chat/tool-cards.ts` | Tool output rendering |
| Markdown | `ui/src/ui/markdown.ts` | marked + dompurify |

---

## Infrastructure Components

| Component | Location | Purpose |
|---|---|---|
| Agent Events | `src/infra/agent-events.ts` | `emitAgentEvent()` for monitoring |
| Outbound Delivery | `src/infra/outbound/deliver.ts` | Payload delivery orchestration |
| Delivery Planning | `src/infra/outbound/agent-delivery.ts` | Target resolution |
| Retry Logic | `src/infra/retry.ts` | Exponential backoff with jitter |
| Retry Policies | `src/infra/retry-policy.ts` | Per-channel retry configs |
| Device Identity | `src/infra/device-identity.ts` | Ed25519 key management |
| Device Pairing | `src/infra/device-pairing.ts` | Pairing protocol |
| Skills Remote | `src/infra/skills-remote.ts` | Remote node execution |
| System Presence | `src/infra/system-presence.ts` | Instance tracking |
| Gateway Lock | `src/infra/gateway-lock.ts` | Singleton enforcement |
| JSON File | `src/infra/json-file.ts` | Async JSON helpers |

---

## Extension Categories

### Channel Extensions (20)
bluebubbles, discord, feishu, googlechat, imessage, irc, line, matrix, mattermost, msteams, nextcloud-talk, nostr, signal, slack, telegram, twitch, whatsapp, zalo, zalouser

### Auth Provider Extensions (5)
copilot-proxy, google-antigravity-auth, google-gemini-cli-auth, minimax-portal-auth, qwen-portal-auth

### Feature Extensions (11)
device-pair, diagnostics-otel, llm-task, lobster, memory-core, memory-lancedb, open-prose, phone-control, talk-voice, thread-ownership, voice-call

---

## Native App Components

| App | Location | Framework | Features |
|---|---|---|---|
| macOS | `apps/macos/` | SwiftUI | Menu bar, Voice Wake, PTT, Talk Mode overlay, WebChat |
| iOS | `apps/ios/` | Swift/UIKit | Canvas, Voice Wake, Talk Mode, camera, Bonjour pairing |
| Android | `apps/android/` | Kotlin/Gradle | Canvas, Talk Mode, camera, screen recording, SMS |
| Shared | `apps/shared/` | Swift | OpenClawKit shared library |
