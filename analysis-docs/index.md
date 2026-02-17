# OpenClaw Analysis Documentation

**Generated**: 2026-02-15
**Author**: AI-assisted analysis for Amol
**Source**: Full codebase scan of openclaw@2026.2.15 (main branch)

---

## Documents

### Project Understanding

| # | Document | Description |
|---|---|---|
| 01 | [Project Overview](01-project-overview.md) | What OpenClaw is, technology stack, capabilities, project scale |
| 02 | [Architecture](02-architecture.md) | High-level architecture diagram, component deep dive, data flow, security |
| 03 | [Source Tree Analysis](03-source-tree-analysis.md) | Annotated directory structure, entry points, build output |
| 04 | [Component Inventory](04-component-inventory.md) | Complete inventory of all gateway, agent, channel, UI, and infrastructure components |

### Deep Dives

| # | Document | Description |
|---|---|---|
| 05 | [Agent & Event Queue Deep Dive](05-agent-deep-dive.md) | How Claude is connected, event queues, agent execution pipeline, streaming, sessions |
| 06 | [UI-Agent Interaction](06-ui-agent-interaction.md) | How the Control UI works with agents, WebSocket protocol, chat flow, tool cards |

### Architectural Guidance

| # | Document | Description |
|---|---|---|
| 07 | [Multi-Agent Testing System Guide](07-architectural-guidance-multi-agent-testing.md) | Architecture for Amol's 6-agent testing system: orchestrator, pipeline, discovery queue |

---

## Quick Reference

**OpenClaw in one sentence**: A self-hosted multi-channel gateway that connects chat apps (WhatsApp, Telegram, Discord, etc.) to AI coding agents via a WebSocket control plane.

**Key architectural patterns used by OpenClaw**:
- Lane-based command queue (serialize per-session, parallelize across sessions)
- Pi SDK embedded agent (not subprocess — direct library import)
- WebSocket RPC + event streaming (no REST)
- JSONL session persistence with compaction
- Plugin-based channel system with standardized adapters
- Dynamic system prompt assembly from multiple sections
- Subagent spawning with depth/count limits
- Tool policy enforcement (per-profile, per-sandbox)

**Patterns recommended for Amol's multi-agent testing system**:
- Orchestrator + specialized agents (like OpenClaw's Gateway + agents)
- Lane-based queue (one lane per task, serialize phases)
- Discovery queue for runtime-found test cases
- JSONL state persistence for task lifecycle
- Event streaming for progress monitoring
- Agent-specific tool registries
- Retry loops with validation (topology → validate → iterate)
