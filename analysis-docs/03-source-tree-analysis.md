# OpenClaw - Source Tree Analysis

**Generated**: 2026-02-15
**Source**: Full codebase scan of openclaw@2026.2.15

---

## Top-Level Structure

```
openclaw/
├── src/                          # Main TypeScript source (71 subdirectories)
│   ├── entry.ts                  # CLI entry (respawn guard, profile parsing)
│   ├── index.ts                  # Main exports
│   ├── extensionAPI.ts           # Extension API surface
│   ├── utils.ts                  # Core utilities
│   │
│   ├── cli/                      # CLI framework & commands
│   │   ├── program.ts            # Re-exports buildProgram
│   │   ├── program/
│   │   │   └── build-program.ts  # Commander.js program builder
│   │   ├── run-main.ts           # Main CLI dispatch
│   │   ├── gateway-cli/          # `openclaw gateway` command
│   │   ├── daemon-cli/           # `openclaw daemon` commands
│   │   ├── node-cli/             # `openclaw node` commands
│   │   └── ports.ts              # Port management
│   │
│   ├── gateway/                  # ★ GATEWAY (Control Plane)
│   │   ├── server-http.ts        # HTTP/HTTPS server factory
│   │   ├── server/
│   │   │   ├── ws-connection.ts        # WebSocket lifecycle
│   │   │   ├── ws-connection/
│   │   │   │   ├── message-handler.ts  # RPC message routing
│   │   │   │   └── auth-messages.ts    # Auth frame handling
│   │   │   ├── http-listen.ts          # HTTP binding
│   │   │   ├── health-state.ts         # Health snapshot
│   │   │   └── plugins-http.ts         # Plugin HTTP endpoints
│   │   ├── server-methods.ts     # RPC handler dispatcher
│   │   ├── server-methods-list.ts # Available methods registry
│   │   ├── server-methods/       # ★ 40+ RPC handlers
│   │   │   ├── agent.ts          # Agent execution (692 lines)
│   │   │   ├── agents.ts         # Agent CRUD (530 lines)
│   │   │   ├── chat.ts           # Chat transcript & streaming
│   │   │   ├── send.ts           # Message delivery
│   │   │   ├── channels.ts       # Channel management
│   │   │   ├── sessions.ts       # Session lifecycle
│   │   │   ├── config.ts         # Config read/write
│   │   │   ├── connect.ts        # Connection handshake
│   │   │   ├── health.ts         # Health probes
│   │   │   ├── agent-job.ts      # Job status tracking
│   │   │   └── types.ts          # GatewayRequestContext
│   │   ├── protocol/
│   │   │   ├── index.ts          # Protocol validation
│   │   │   ├── schema/
│   │   │   │   ├── agent.ts      # Agent RPC schemas
│   │   │   │   ├── frames.ts     # Protocol frame schemas
│   │   │   │   └── error-codes.ts
│   │   │   └── client-info.ts
│   │   ├── auth.ts               # Authentication logic
│   │   ├── auth-rate-limit.ts    # Rate limiting
│   │   ├── boot.ts               # Startup execution (BOOT.md)
│   │   ├── server-broadcast.ts   # Event broadcasting
│   │   ├── server-chat.ts        # ChatRunState management
│   │   ├── server-chat-registry.ts # In-flight chat runs
│   │   ├── server-channels.ts    # Channel manager
│   │   ├── server-health.ts      # Health RPC
│   │   ├── session-utils.ts      # Session helpers
│   │   └── agent-prompt.ts       # Agent prompt resolution
│   │
│   ├── agents/                   # ★ AI AGENT RUNTIME (368 files)
│   │   ├── pi-embedded.ts        # Pi embedded runner exports
│   │   ├── pi-embedded-runner/   # ★ Core agent execution
│   │   │   ├── run.ts            # Main run lifecycle
│   │   │   ├── run/
│   │   │   │   └── attempt.ts    # createAgentSession() call
│   │   │   ├── runs.ts           # Concurrent run tracking (ACTIVE_EMBEDDED_RUNS)
│   │   │   ├── compact.ts        # Session compaction
│   │   │   ├── history.ts        # Context window management
│   │   │   ├── lanes.ts          # Concurrency lanes (per-session)
│   │   │   ├── model.ts          # Model resolution
│   │   │   ├── tool-split.ts     # Tool result splitting
│   │   │   ├── system-prompt.ts  # Prompt sections
│   │   │   ├── extra-params.ts   # Custom params
│   │   │   ├── sandbox-info.ts   # Sandbox metadata
│   │   │   ├── session-manager-cache.ts # SM instance cache
│   │   │   └── types.ts          # EmbeddedPiRunResult, RunMeta
│   │   ├── pi-embedded-subscribe.ts  # ★ Streaming event handler (700 lines)
│   │   ├── pi-embedded-helpers.ts    # Helper utilities
│   │   ├── pi-embedded-block-chunker.ts # Text chunking
│   │   ├── pi-tools.ts           # ★ Tool creation (465 lines)
│   │   ├── pi-tool-definition-adapter.ts # Tool bridge
│   │   ├── tools/                # Individual tool implementations
│   │   │   ├── sessions-spawn-tool.ts  # Subagent spawning
│   │   │   └── ...
│   │   ├── system-prompt.ts      # ★ Dynamic prompt builder (673 lines)
│   │   ├── model-selection.ts    # Model selection logic
│   │   ├── agent-scope.ts        # Agent resolution
│   │   ├── agent-paths.ts        # Agent file paths
│   │   ├── auth-profiles.ts      # OAuth/API key management
│   │   ├── bash-tools.ts         # Bash command execution
│   │   ├── channel-tools.ts      # Channel-specific tools
│   │   ├── tool-policy.ts        # Tool allowlist/denylist
│   │   ├── tool-display-common.ts # Tool display formatting
│   │   ├── cli-runner.ts         # CLI agent execution
│   │   ├── cli-session.ts        # CLI session management
│   │   ├── workspace-templates.ts # Workspace bootstrapping
│   │   ├── subagent-registry.ts  # Active subagent tracking
│   │   ├── subagent-announce.ts  # Subagent announcements
│   │   ├── session-write-lock.ts # Concurrent write prevention
│   │   ├── session-tool-result-guard.ts # Output sanitization
│   │   ├── anthropic-payload-log.ts # API logging
│   │   ├── pi-extensions/        # Pi SDK extensions
│   │   │   ├── compaction-safeguard.ts
│   │   │   └── context-pruning.ts
│   │   └── identity.ts           # Agent identity (name, emoji, avatar)
│   │
│   ├── routing/                  # ★ MESSAGE ROUTING
│   │   ├── resolve-route.ts      # Agent route resolution
│   │   ├── session-key.ts        # Session key building
│   │   └── bindings.ts           # Config binding matching
│   │
│   ├── auto-reply/               # ★ REPLY ENGINE (79 files)
│   │   ├── dispatch.ts           # Inbound message dispatcher
│   │   ├── queue.ts              # FollowupRun queue
│   │   ├── templating.ts         # Template context (MsgContext)
│   │   └── reply/                # Reply execution
│   │       ├── agent-runner.ts           # Main reply driver
│   │       ├── agent-runner-execution.ts # Agent turn loop
│   │       ├── agent-runner-memory.ts    # Memory loading
│   │       ├── abort.ts                  # Run abort logic
│   │       ├── block-reply-pipeline.ts   # Block coalescing
│   │       ├── block-streaming.ts        # Stream processing
│   │       └── reply-dispatcher.ts       # Delivery dispatch
│   │
│   ├── channels/                 # ★ CHANNEL ABSTRACTION
│   │   ├── registry.ts           # Channel catalog (8 core channels)
│   │   ├── dock.ts               # Channel adapters & metadata
│   │   ├── chat-type.ts          # direct/group/channel/thread
│   │   ├── plugins/
│   │   │   ├── types.core.ts     # Core channel interfaces
│   │   │   ├── types.adapters.ts # Adapter contracts
│   │   │   ├── types.plugin.ts   # Full plugin interface
│   │   │   ├── catalog.ts        # Plugin catalog
│   │   │   ├── index.ts          # Registry
│   │   │   ├── load.ts           # Plugin loading
│   │   │   ├── normalize/        # Per-channel normalizers
│   │   │   ├── onboarding/       # Per-channel setup wizards
│   │   │   ├── outbound/         # Send pipeline
│   │   │   ├── actions/          # Message actions
│   │   │   ├── agent-tools/      # Channel tools for agents
│   │   │   ├── status-issues/    # Diagnostic checks
│   │   │   └── group-mentions.ts # Mention handling
│   │   ├── telegram/             # Telegram (97 files)
│   │   ├── discord/              # Discord (49 files)
│   │   ├── slack/                # Slack (37 files)
│   │   ├── signal/               # Signal (29 files)
│   │   ├── imessage/             # iMessage (18 files)
│   │   ├── whatsapp/             # WhatsApp (6 files)
│   │   └── line/                 # LINE (43 files)
│   │
│   ├── process/                  # ★ CONCURRENCY
│   │   └── command-queue.ts      # Lane-based command queuing
│   │
│   ├── sessions/                 # Session model
│   │   ├── input-provenance.ts   # Message source tracking
│   │   ├── send-policy.ts        # Delivery policy
│   │   ├── session-key-utils.ts  # Key utilities
│   │   ├── session-label.ts      # Labels
│   │   └── transcript-events.ts  # Event types
│   │
│   ├── config/                   # Configuration (152 files)
│   │   ├── config.ts             # Config loading
│   │   ├── types.ts              # OpenClawConfig type
│   │   ├── validation.ts         # Config validation
│   │   ├── zod-schema.ts         # Zod schema
│   │   ├── paths.ts              # Path resolution
│   │   ├── sessions/             # Session storage API
│   │   │   └── main-session.ts   # Main session resolution
│   │   └── profiles/             # Profile management
│   │
│   ├── infra/                    # Infrastructure (182 files)
│   │   ├── agent-events.ts       # Event emission
│   │   ├── outbound/
│   │   │   ├── agent-delivery.ts # Delivery planning
│   │   │   ├── deliver.ts        # Payload delivery
│   │   │   ├── outbound-session.ts
│   │   │   ├── targets.ts
│   │   │   └── payloads.ts
│   │   ├── retry-policy.ts       # Retry configs
│   │   ├── retry.ts              # Retry with backoff
│   │   ├── device-identity.ts    # Device crypto
│   │   ├── device-pairing.ts     # Pairing protocol
│   │   ├── skills-remote.ts      # Remote node skills
│   │   ├── system-presence.ts    # Instance presence
│   │   ├── gateway-lock.ts       # Singleton lock
│   │   └── json-file.ts          # JSON file helpers
│   │
│   ├── media/                    # Media pipeline (30 files)
│   │   ├── local-roots.ts        # Sandboxed paths
│   │   └── ...                   # Image/audio/video processing
│   │
│   ├── browser/                  # Browser automation (90 files)
│   │   └── ...                   # Playwright integration
│   │
│   ├── memory/                   # Persistent memory (63 files)
│   ├── media-understanding/      # Image/video analysis (30 files)
│   ├── auto-reply/               # Auto-reply system (79 files)
│   ├── cron/                     # Scheduled tasks (44 files)
│   ├── daemon/                   # Daemon lifecycle (38 files)
│   ├── plugins/                  # Plugin system (48 files)
│   ├── web/                      # Web provider & control (46 files)
│   ├── tui/                      # Terminal UI (30 files)
│   ├── terminal/                 # Terminal helpers (17 files)
│   ├── logging/                  # Structured logging (22 files)
│   ├── security/                 # Auth & security (24 files)
│   ├── utils/                    # Shared utilities (24 files)
│   ├── shared/                   # Shared types (21 files)
│   ├── types/                    # Type definitions (11 files)
│   ├── providers/                # Auth providers (11 files)
│   ├── acp/                      # ACP integration (16 files)
│   ├── commands/                 # Command implementations
│   ├── wizard/                   # Onboarding wizard (14 files)
│   ├── node-host/                # Node integration (9 files)
│   ├── pairing/                  # Device pairing (7 files)
│   ├── macos/                    # macOS integration (6 files)
│   ├── canvas-host/              # Canvas rendering (8 files)
│   ├── hooks/                    # Hook system
│   │   └── bundled/              # Built-in hooks
│   └── plugin-sdk/               # ★ PUBLIC PLUGIN SDK
│       ├── index.ts              # SDK exports
│       └── account-id.ts         # Account ID helpers
│
├── ui/                           # ★ CONTROL UI
│   ├── src/
│   │   ├── main.ts               # Entry point
│   │   ├── styles.css             # Global styles
│   │   └── ui/
│   │       ├── app.ts             # OpenClawApp component
│   │       ├── app-render.ts      # Page rendering
│   │       ├── app-chat.ts        # Chat logic
│   │       ├── app-gateway.ts     # Gateway connection
│   │       ├── app-tool-stream.ts # Tool streaming
│   │       ├── app-polling.ts     # Tab-specific polling
│   │       ├── gateway.ts         # GatewayBrowserClient (WebSocket)
│   │       ├── storage.ts         # localStorage persistence
│   │       ├── navigation.ts      # URL routing
│   │       ├── views/             # 14 tab views
│   │       ├── controllers/       # State + API controllers
│   │       ├── chat/              # Chat utilities
│   │       ├── components/        # Reusable components
│   │       └── types/             # Type definitions
│   ├── index.html
│   ├── package.json               # Lit 3.3.2, Vite 7.3.1
│   └── vite.config.ts
│
├── extensions/                   # ★ 36 PLUGIN PACKAGES
│   ├── bluebubbles/              # iMessage via BlueBubbles
│   ├── copilot-proxy/            # GitHub Copilot proxy auth
│   ├── device-pair/              # Device pairing
│   ├── diagnostics-otel/         # OpenTelemetry diagnostics
│   ├── discord/                  # Discord channel
│   ├── feishu/                   # Feishu/Lark
│   ├── googlechat/               # Google Chat
│   ├── imessage/                 # iMessage
│   ├── irc/                      # IRC
│   ├── line/                     # LINE
│   ├── llm-task/                 # LLM task tool
│   ├── lobster/                  # Lobster integration
│   ├── matrix/                   # Matrix protocol
│   ├── mattermost/               # Mattermost
│   ├── memory-core/              # Core memory system
│   ├── memory-lancedb/           # LanceDB memory backend
│   ├── msteams/                  # Microsoft Teams
│   ├── nextcloud-talk/           # Nextcloud Talk
│   ├── nostr/                    # Nostr protocol
│   ├── open-prose/               # Prose writing tool
│   ├── phone-control/            # Phone control
│   ├── signal/                   # Signal
│   ├── slack/                    # Slack
│   ├── talk-voice/               # Voice calling
│   ├── telegram/                 # Telegram
│   ├── thread-ownership/         # Thread management
│   ├── twitch/                   # Twitch
│   ├── voice-call/               # Voice calls
│   ├── whatsapp/                 # WhatsApp
│   ├── zalo/                     # Zalo (OA)
│   └── zalouser/                 # Zalo (User)
│
├── skills/                       # ★ 50+ BUNDLED SKILLS
│   ├── coding-agent/             # Coding assistant
│   ├── github/                   # GitHub integration
│   ├── 1password/                # 1Password
│   ├── notion/                   # Notion
│   ├── spotify-player/           # Spotify
│   ├── slack/                    # Slack skill
│   └── ...                       # 44 more skills
│
├── apps/                         # ★ NATIVE APPS
│   ├── macos/                    # macOS menu bar (SwiftUI)
│   ├── ios/                      # iOS companion (Swift)
│   ├── android/                  # Android companion (Kotlin)
│   └── shared/                   # Shared OpenClawKit
│
├── packages/                     # Bot workspaces
│   ├── clawdbot/                 # ClawdBot
│   └── moltbot/                  # MoltBot
│
├── scripts/                      # Build & utility scripts (79 files)
│   ├── bundle-a2ui.sh            # Canvas UI bundling
│   ├── protocol-gen.ts           # Protocol code gen
│   ├── protocol-gen-swift.ts     # Swift protocol gen
│   └── ...
│
├── docs/                         # Mintlify documentation (44 dirs)
│   ├── channels/                 # Channel guides
│   ├── cli/                      # CLI reference
│   ├── concepts/                 # Architecture concepts
│   ├── gateway/                  # Gateway ops
│   └── ...
│
├── test/                         # Test infrastructure
│   ├── setup.ts                  # Global setup
│   ├── helpers/                  # Test utilities
│   ├── fixtures/                 # Test data
│   └── mocks/                    # Mock implementations
│
├── openclaw.mjs                  # ★ CLI entry point (shim)
├── package.json                  # Root workspace config
├── pnpm-workspace.yaml           # Workspace definition
├── tsconfig.json                 # TypeScript config (ES2023, NodeNext)
├── tsdown.config.ts              # Build bundler config
├── vitest.config.ts              # Main test config
├── AGENTS.md → CLAUDE.md         # Agent guidelines (symlink)
├── CONTRIBUTING.md               # Contribution guide
├── CHANGELOG.md                  # Release history
└── Dockerfile                    # Container build
```

## Entry Point Chain

```
openclaw.mjs (shim)
  └─→ dist/entry.js
    └─→ src/entry.ts (respawn guard, profile parsing)
      └─→ src/cli/run-main.ts (Commander.js program dispatch)
        ├─→ openclaw gateway  → src/cli/gateway-cli/start.ts
        ├─→ openclaw agent    → src/commands/agent.ts
        ├─→ openclaw send     → message delivery
        ├─→ openclaw wizard   → src/wizard/
        └─→ openclaw ...      → 20+ more commands
```

## Build Output

```
dist/
├── index.js              # Main library export
├── entry.js              # CLI entry
├── plugin-sdk/
│   ├── index.js          # Public SDK
│   └── account-id.js     # Account ID helpers
└── control-ui/           # Built Control UI (from ui/)
```
