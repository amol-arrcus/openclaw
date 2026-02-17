# OpenClaw - Skill Architecture & Plugin System

**Generated**: 2026-02-15
**Focus**: How the skill system is built, not what individual skills do

---

## What Is a Skill?

A **skill** is a directory with a `SKILL.md` file that teaches the agent how to use a specific capability. It's not code that runs — it's **instructions** the agent reads and follows, combined with optional configuration, scripts, and environment setup.

```
Skills are teaching modules, not executable plugins.
The agent reads the instructions and uses its existing tools to carry them out.
```

---

## Skill File Structure

A typical skill directory:

```
my-skill/
├── SKILL.md           ← Required: metadata + instructions for the agent
├── scripts/           ← Optional: helper scripts the agent can invoke
├── templates/         ← Optional: reference files
├── config/            ← Optional: skill-specific configuration
└── docs/              ← Optional: additional documentation
```

### SKILL.md Anatomy

```yaml
---
name: my-skill                    # Unique identifier (required)
description: One-line summary     # Brief description
user-invocable: true              # Can users trigger via /my-skill?
disable-model-invocation: false   # Exclude from agent's prompt?
command-dispatch: tool            # Optional: bypass model, dispatch to tool directly
command-tool: exec                # Which tool to invoke directly
command-arg-mode: raw             # How to forward arguments
homepage: https://...             # Link shown in UI
metadata: {"openclaw": {          # OpenClaw-specific config (single-line JSON)
  "skillKey": "config-key",
  "primaryEnv": "API_KEY_VAR",
  "emoji": "🔧",
  "os": ["darwin", "linux"],
  "requires": {
    "bins": ["tool1"],
    "env": ["ENV_VAR"],
    "config": ["config.path"]
  },
  "install": [{
    "kind": "brew",
    "formula": "tool1",
    "bins": ["tool1"]
  }]
}}
---

# My Skill

Instructions for the agent on how to use this capability.
The agent reads this and follows the guidance using its available tools.
```

---

## How Skills Are Discovered

Skills are scanned from **6 locations** in precedence order (highest wins):

| Priority | Location | What It Contains |
|----------|----------|-----------------|
| 1 (highest) | `<workspace>/skills/` | Project-specific skills |
| 2 | `<workspace>/.agents/skills/` | Project agent skills |
| 3 | `~/.agents/skills/` | Personal agent skills |
| 4 | `~/.openclaw/skills/` | Managed/shared skills |
| 5 | Plugin-provided skills | From enabled extensions |
| 6 (lowest) | Bundled skills (shipped with package) | Default capabilities |

**Name collisions**: If two skills have the same name, the higher-priority location wins (workspace overrides bundled).

**File watching**: Optional hot-reload when skill files change (`skills.load.watch: true`, debounce 250ms).

---

## How Skills Get Into the Agent's Prompt

### Loading Pipeline

```
Session starts
  │
  ├─ 1. DISCOVER: Scan all 6 skill directories
  │     → Collect SkillEntry objects (name, source, metadata)
  │
  ├─ 2. FILTER: Apply gating rules
  │     → Is it enabled? (skills.entries[name].enabled)
  │     → Passes allowlist? (skills.allowBundled for bundled skills)
  │     → Requirements met? (bins, env vars, config)
  │     → OS compatible? (metadata.openclaw.os)
  │     → Agent allows it? (agents.list[].skills allowlist)
  │
  ├─ 3. SNAPSHOT: Build cached skill set
  │     → SkillSnapshot: { prompt, skills[], resolvedSkills[], version }
  │     → Cached per session, invalidated when skills change
  │
  ├─ 4. INJECT ENV: Apply environment overrides
  │     → Inject skill-specific API keys and env vars
  │     → Scoped to single agent turn (reverted after)
  │
  └─ 5. EMBED IN PROMPT: Insert into system prompt
        → XML-formatted skill list:
          <available_skills>
            <skill>
              <name>my-skill</name>
              <description>One-line summary</description>
              <location>/path/to/my-skill/SKILL.md</location>
            </skill>
            ...
          </available_skills>
```

### What the Agent Sees

The system prompt includes a "Skills" section that tells the agent:

```
Before replying, scan <available_skills> entries.
- If exactly one skill clearly applies: read its SKILL.md and follow instructions
- If multiple could apply: choose the most specific one
- If none apply: proceed normally with available tools
```

The agent then uses its `read` tool to load the full SKILL.md and follows the instructions within.

### Token Cost

- Base overhead: ~195 tokens for the Skills section
- Per skill: ~24 tokens (name + description + path)
- 40 skills ≈ 1,155 tokens of system prompt

---

## Three Ways to Invoke a Skill

### 1. Model-Driven (Default)

The agent reads the skill list in its prompt, decides a skill is relevant, reads the full SKILL.md, and follows the instructions.

```
User: "Review my pull request #42"
  → Agent sees "review-pr" skill in available_skills
  → Agent reads review-pr/SKILL.md
  → Follows instructions (run git diff, analyze, comment)
```

### 2. User Slash Command (`user-invocable: true`)

Users can trigger skills directly:

```
User: /review-pr 42
  → Routed to agent with: { command: "42", skillName: "review-pr" }
  → Agent reads SKILL.md, executes with provided arguments
```

Works across all channels (Slack, Discord, Telegram, Web UI).

### 3. Tool Dispatch (`command-dispatch: tool`)

Slash command bypasses the model and dispatches directly to a named tool:

```yaml
command-dispatch: tool
command-tool: exec
command-arg-mode: raw
```

```
User: /build --target prod
  → Directly calls exec tool with "--target prod" as argument
  → No model invocation, instant execution
```

---

## Skill Configuration

### Per-Skill Settings

```json5
{
  "skills": {
    "entries": {
      "web-search": {
        "enabled": true,          // Enable/disable this skill
        "apiKey": "SERPAPI_KEY",   // Injected as the skill's primaryEnv
        "env": {                   // Additional env vars
          "SEARCH_ENGINE": "google"
        },
        "config": {               // Custom config (skill-defined schema)
          "maxResults": 10
        }
      }
    }
  }
}
```

### Environment Injection

When a skill has `primaryEnv` in its metadata and an `apiKey` in config:

```
Before agent run:
  process.env["SERPAPI_KEY"] = config.skills.entries["web-search"].apiKey

After agent run:
  process.env["SERPAPI_KEY"] restored to previous value
```

Environment overrides are **scoped to a single agent turn** and reverted afterward.

### Global Skill Settings

```json5
{
  "skills": {
    "allowBundled": ["skill1", "skill2"],  // Restrict bundled skills
    "load": {
      "extraDirs": ["/path/to/more/skills"],  // Additional scan dirs
      "watch": true,                           // Hot-reload on change
      "watchDebounceMs": 250
    },
    "install": {
      "preferBrew": true,    // Prefer Homebrew for binary deps
      "nodeManager": "npm"   // npm/pnpm/yarn/bun
    }
  }
}
```

### Per-Agent Skill Allowlist

Each agent can restrict which skills it has access to:

```json5
{
  "agents": {
    "list": [
      {
        "id": "coding",
        "skills": ["review-pr", "prepare-pr", "web-search"]
        // Only these 3 skills available to this agent
        // Omit "skills" field → all skills available
        // Empty array → no skills
      }
    ]
  }
}
```

---

## Skill Requirements System

Skills can declare what they need to function:

```yaml
metadata: {"openclaw": {
  "requires": {
    "bins": ["ffmpeg", "sox"],           # ALL of these binaries must exist
    "anyBins": ["chrome", "chromium"],   # ANY ONE of these must exist
    "env": ["OPENAI_API_KEY"],           # Env vars that must be set
    "config": ["tools.browser.profile"]  # Config paths that must exist
  },
  "os": ["darwin", "linux"]             # OS restriction
}}
```

**Eligibility check**: If requirements aren't met, the skill is excluded from the agent's prompt entirely. The UI shows what's missing and offers install options.

---

## Skill Installation System

Skills can declare how to install their dependencies:

```yaml
metadata: {"openclaw": {
  "install": [
    {
      "kind": "brew",
      "formula": "ffmpeg",
      "bins": ["ffmpeg", "ffprobe"],
      "label": "Install via Homebrew"
    },
    {
      "kind": "node",
      "package": "@anthropic-ai/tool",
      "bins": ["tool-cli"]
    },
    {
      "kind": "download",
      "url": "https://example.com/tool.tar.gz",
      "archive": "tar.gz",
      "extract": true,
      "bins": ["tool"]
    }
  ]
}}
```

### Supported Installers

| Kind | What It Does |
|------|-------------|
| `brew` | `brew install <formula>` |
| `node` | `npm/pnpm/yarn/bun install -g <package>` |
| `go` | `go install <module>` |
| `uv` | `uv tool install <package>` |
| `download` | Fetch URL, auto-detect archive, extract with security scan |

### Installation Flow

```
1. Find skill by name
2. Select installer (prefer user config, then uv > node > brew > go > download)
3. Execute installation command
4. Security scan: check for dangerous patterns in downloaded files
5. Verify binary exists after install
6. Return result (success/failure + output)
```

---

## Remote Skill Execution

When the gateway runs on Linux but a connected macOS node has required binaries:

```
Linux gateway ← WebSocket → macOS node
  │                            │
  │  "Does this node have      │
  │   ffmpeg, sox, etc.?"      │
  │  ─────────────────────►    │
  │                            │  system.which checks
  │  ◄─────────────────────    │
  │  "Yes: ffmpeg, sox"        │
  │                            │
  └─ Skills requiring darwin    │
     become eligible with       │
     "Remote macOS node         │
      available" note           │
```

The agent must use `nodes.run` to invoke commands on the remote node. Skills requiring macOS-only binaries become available on Linux gateways this way.

---

## Skill Management RPC

| Method | What It Does |
|--------|-------------|
| `skills.status` | Returns per-skill eligibility: enabled, blocked, missing requirements, install options |
| `skills.bins` | Lists all binary requirements across all skills |
| `skills.install` | Runs an installer for a skill's dependencies |
| `skills.update` | Modifies skill config (enabled, apiKey, env vars) |

---

## The 5-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│ Layer 5: INJECTION                                    │
│   Inject env vars, embed skills XML in system prompt  │
├─────────────────────────────────────────────────────┤
│ Layer 4: SNAPSHOT                                     │
│   Cache eligible skills per session, version tracking │
├─────────────────────────────────────────────────────┤
│ Layer 3: FILTERING                                    │
│   Requirements, allowlists, OS, agent scope, remote   │
├─────────────────────────────────────────────────────┤
│ Layer 2: LOADING                                      │
│   Parse SKILL.md frontmatter, extract metadata        │
├─────────────────────────────────────────────────────┤
│ Layer 1: DISCOVERY                                    │
│   Scan 6 precedence directories for SKILL.md files    │
└─────────────────────────────────────────────────────┘
```

---

## For Your Multi-Agent System

The skill architecture translates well to your testing system:

1. **Skills as test capabilities** — each skill = a testing competency (e.g., "bgp-validation", "interface-testing", "traffic-generation")
2. **SKILL.md as instructions** — describe how to use each testing capability, what tools to invoke, what output to expect
3. **Requirements system** — skills can require specific test tools (`iperf`, `tcpdump`, SSH access)
4. **Per-agent skill allowlists** — your topology agent only gets topology-related skills; your test runner only gets execution skills
5. **Environment injection** — inject device credentials, testbed URLs per skill
6. **Precedence-based discovery** — project-specific test skills override generic ones
7. **Hot-reload** — update skill instructions without restarting the system
