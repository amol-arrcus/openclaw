# OpenClaw - Project Overview

**Generated**: 2026-02-15
**Source**: Codebase scan of openclaw@2026.2.15 (main branch)

---

## What is OpenClaw?

OpenClaw is a **self-hosted, multi-channel gateway** that connects messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, and 13+ more) to AI coding agents. It acts as a unified bridge — you run a single Gateway process on your machine or server, and it becomes an always-available AI assistant reachable from any of your chat apps.

## Who is it for?

Developers and power users who want a personal AI assistant they can message from anywhere, without giving up control of their data or relying on a hosted service.

## Key Capabilities

| Capability | Description |
|---|---|
| **Multi-channel gateway** | WhatsApp, Telegram, Discord, iMessage, Slack, Signal, IRC, Google Chat, Matrix, MS Teams, LINE, Nostr, and more — all from one process |
| **Plugin channels** | Add channels via extension packages (36 extensions available) |
| **Multi-agent routing** | Multiple AI agents with isolated sessions per agent, workspace, or sender |
| **Media support** | Send/receive images, audio, documents across all channels |
| **Web Control UI** | Browser dashboard (Lit + Vite) for chat, config, sessions, nodes |
| **Mobile nodes** | iOS and Android companion apps with Canvas support |
| **macOS app** | Menu bar app with Voice Wake, PTT, Talk Mode |
| **Skills system** | 50+ bundled skills (GitHub, Notion, Slack, 1Password, Spotify, etc.) |
| **Cron/scheduling** | Scheduled agent tasks and heartbeat system |
| **Tool use** | 60+ tools including bash, web search, browser automation, file ops |

## Technology Stack

| Layer | Technology |
|---|---|
| **Language** | TypeScript (ES2023, ESM) |
| **Runtime** | Node.js >= 22.12.0 |
| **Package manager** | pnpm 10.23.0 (workspaces) |
| **AI SDK** | `@mariozechner/pi-agent-core`, `pi-ai`, `pi-coding-agent` (v0.49.3) |
| **Build** | tsdown (Rolldown-based bundler) |
| **Test** | Vitest 4.0.18 with V8 coverage |
| **Lint/Format** | Oxlint + Oxfmt |
| **UI framework** | Lit 3.3.2 (Web Components) |
| **UI build** | Vite 7.3.1 |
| **Channels** | grammy (Telegram), discord.js, @slack/bolt, baileys (WhatsApp), signal-cli |
| **Browser automation** | Playwright |
| **Validation** | TypeBox (runtime), Zod (config) |
| **Docs** | Mintlify (docs.openclaw.ai) |

## Project Scale

| Metric | Count |
|---|---|
| Source directories in `src/` | 71 |
| Extension packages | 36 |
| Bundled skills | 50+ |
| Supported channels | 20+ (core + extensions) |
| UI component directories | 59 |
| Documentation directories | 44 |
| Native apps | 3 (macOS, iOS, Android) |

## Versioning

Uses calendar-based versioning: `YYYY.M.D` (e.g., `2026.2.15`)

Release channels: **stable** (npm `latest`), **beta** (`vYYYY.M.D-beta.N`), **dev** (main head)

## Monorepo Structure

```
pnpm-workspace.yaml:
  - . (root - main CLI + gateway)
  - ui (Control UI web app)
  - packages/* (clawdbot, moltbot workspaces)
  - extensions/* (36 plugin packages)
```

## How to Run

```bash
npm install -g openclaw@latest     # Install globally
openclaw onboard --install-daemon  # Guided setup
openclaw channels login            # Pair WhatsApp/Telegram/etc.
openclaw gateway --port 18789      # Start the gateway
```

## License

MIT — open source, community-driven.
