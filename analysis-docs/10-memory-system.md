# OpenClaw - Memory System

**Generated**: 2026-02-15
**Focus**: What the memory system does, how it's structured, and how the agent remembers things

---

## What Is the Memory System?

The memory system is **file-based**. There are no databases or proprietary formats — just plain Markdown files in the agent's workspace. The agent "remembers" by writing to disk and recalls by reading files or searching them with vector embeddings.

---

## The Agent's File-Based Memory

Every agent has a workspace directory (default: `~/.openclaw/workspace/`). The following files make up its memory and identity:

### Core Memory Files

| File | What It Does | When It's Loaded |
|------|-------------|------------------|
| **`MEMORY.md`** | Curated long-term memory. Decisions, preferences, durable facts. | Main session only (never in group chats) |
| **`memory/YYYY-MM-DD.md`** | Daily append-only logs. Raw notes, context, running diary. | Today's + yesterday's file, main session only |
| **`SOUL.md`** | Agent's identity, values, personality, behavioral principles | Every session |
| **`USER.md`** | Human's name, timezone, contact info, preferences | Every session |
| **`AGENTS.md`** | Workspace guidelines: how the agent should behave, memory strategy, group etiquette | Every session |
| **`TOOLS.md`** | Local tool configurations (SSH details, camera names, voice prefs) | Every session |
| **`IDENTITY.md`** | Agent's name, emoji, creature type, vibe | Every session |
| **`HEARTBEAT.md`** | Optional checklist for proactive work (polled by heartbeat) | Heartbeat runs only |
| **`BOOTSTRAP.md`** | One-time first-run ritual. Deleted after initial setup. | First session only |
| **`BOOT.md`** | Optional startup hooks (run on gateway start) | Gateway boot only |

### How Files Map to Memory Concepts

```
Long-term memory     →  MEMORY.md (curated, distilled, durable)
Short-term memory    →  memory/YYYY-MM-DD.md (daily logs, raw notes)
Identity             →  SOUL.md + IDENTITY.md (who am I?)
User knowledge       →  USER.md (who am I talking to?)
Behavioral rules     →  AGENTS.md (how should I act?)
Tool knowledge       →  TOOLS.md (what tools do I have?)
Proactive tasks      →  HEARTBEAT.md (what should I check on?)
```

---

## How Memory Gets Loaded Into the Agent

At session start, workspace files are scanned and injected into the system prompt as embedded context:

### Loading Flow

```
Session starts
  → loadWorkspaceBootstrapFiles()    ← scans workspace for standard files
    → filterBootstrapFilesForSession()  ← filters based on session type
      → Convert to EmbeddedContextFile[] objects
        → Inject into system prompt (once, at start)
```

### What Gets Loaded Per Session Type

| Session Type | Files Loaded |
|-------------|-------------|
| **Main agent (direct chat)** | AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md, MEMORY.md, memory/today.md, memory/yesterday.md |
| **Sub-agent (spawned)** | AGENTS.md, TOOLS.md only |
| **Cron / scheduled** | Minimal (task-specific) |
| **Group chat** | Same as main but **without** MEMORY.md (privacy boundary) |

### Character Limits

Files are embedded as plain text with configurable limits:
- **Per file**: `bootstrapMaxChars` (default: 20,000 chars)
- **Total budget**: `bootstrapTotalMaxChars` (default: 24,000 chars)
- Files exceeding limits are truncated

---

## How the Agent Writes to Memory

There is **no special "save memory" tool**. The agent uses regular file tools (`write`, `edit`) to modify its own workspace files.

### Write Patterns

| What to Remember | Where to Write |
|-----------------|----------------|
| Daily notes, events, running context | `memory/YYYY-MM-DD.md` (create if needed) |
| Significant decisions, distilled learnings | `MEMORY.md` |
| Behavior changes, new rules | `AGENTS.md` |
| Tool configurations | `TOOLS.md` |
| Identity updates | `SOUL.md`, `IDENTITY.md` |
| User info changes | `USER.md` |

### Memory Hygiene (from AGENTS.md guidance)

The agent is instructed to:
- "If you want to remember something, **write it to a file**" (no mental notes)
- "When someone says 'remember this' → update `memory/YYYY-MM-DD.md`"
- "When you learn a lesson → document it so future-you doesn't repeat it"
- Periodically review daily files and distill learnings into `MEMORY.md`
- Remove outdated info from `MEMORY.md`

---

## Vector / Semantic Memory Search

Beyond file reading, OpenClaw provides **hybrid semantic search** over memory files:

### How It Works

```
memory_search("what did we decide about the API design?")
  │
  ├─ Vector search (embeddings)
  │   → Chunk MEMORY.md + memory/*.md into ~400-token segments
  │   → Embed chunks using configured provider (local/OpenAI/Gemini/Voyage)
  │   → Store in SQLite with sqlite-vec extension
  │   → Cosine similarity search
  │
  ├─ BM25 full-text search (keywords)
  │   → SQLite FTS5 index for exact tokens
  │   → Catches IDs, env vars, code symbols that embeddings miss
  │
  └─ Hybrid merge
      → Weighted: 70% vector + 30% BM25 (configurable)
      → Deduplicate by chunk ID
      → Return top results with score, path, line numbers, snippet
```

### Memory Search Tools

| Tool | Purpose |
|------|---------|
| `memory_search(query)` | Semantic + keyword search across MEMORY.md and memory/*.md |
| `memory_get(path, from?, lines?)` | Read specific file ranges by path |

### Search Results Include

- `snippet` (capped ~700 chars)
- `path` (which file matched)
- `startLine`, `endLine` (location in file)
- `score` (relevance)
- Optional `citation` (for transparency)

### Index Storage

- Per-agent SQLite database: `~/.openclaw/memory/{agentId}.sqlite`
- Auto-syncs when files change (debounce 1.5s)
- Full reindex triggered if embedding provider/model changes

### Embedding Provider Auto-Selection

Priority: local → OpenAI → Gemini → Voyage (first available)

---

## Memory Flush (Pre-Compaction)

When a session's context window fills up (~80-90%), OpenClaw automatically triggers a **memory flush** — a silent agent turn that saves important context before compaction erases it.

### What Happens

```
Session context approaching limit
  → runMemoryFlushIfNeeded()
    → Silent agent turn with special prompt:
      "Store durable memories now (use memory/YYYY-MM-DD.md).
       If nothing to store, reply with NO_REPLY."
    → Agent writes important notes to daily file
    → Compaction proceeds (old context summarized/pruned)
```

### When It Triggers

```
flushTrigger = contextWindow - reserveTokensFloor - softThresholdTokens
Flush runs if: totalTokens >= flushTrigger
```

Default values:
- `reserveTokensFloor`: 20,000 tokens
- `softThresholdTokens`: 4,000 tokens

### Safeguards

- **One flush per compaction cycle** (tracked via `memoryFlushCompactionCount`)
- **Skipped if**: workspace is read-only, is heartbeat run, is CLI provider
- **Silent**: User doesn't see the flush turn (NO_REPLY token)
- **Append-only**: Prompt instructs agent to append, not overwrite existing daily entries

### Configuration

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "enabled": true,           // default
          "softThresholdTokens": 4000,
          "prompt": "Custom flush prompt...",
          "systemPrompt": "Custom system guidance..."
        }
      }
    }
  }
}
```

---

## Memory Differences: Main Agent vs Sub-Agents

| Aspect | Main Agent | Sub-Agent |
|--------|-----------|-----------|
| **MEMORY.md** | Loaded (private, curated) | Not loaded |
| **memory/*.md** | Today + yesterday loaded | Not loaded |
| **SOUL.md, USER.md** | Loaded | Not loaded |
| **AGENTS.md, TOOLS.md** | Loaded | Loaded |
| **memory_search tool** | Available | Not available |
| **Can write to memory** | Yes (via file tools) | No direct memory write |
| **Memory flush** | Automatic before compaction | Not applicable |
| **System prompt** | Full (all sections) | Minimal (Tooling, Workspace, Runtime) |

**Why the separation?**
1. **Privacy**: Main agent's personal MEMORY.md doesn't leak to group chats or sub-agents
2. **Focus**: Sub-agents concentrate on their assigned task without unrelated context
3. **Token efficiency**: Lighter prompts for temporary/delegated work

---

## For Your Multi-Agent System

The memory design is elegantly simple and portable to Python/TS:

1. **File-based memory** — no complex infrastructure. Just Markdown files on disk.
2. **Daily logs + curated summary** — raw daily notes distilled into a long-term MEMORY.md
3. **Vector search as an overlay** — optional semantic search using SQLite + embeddings
4. **Pre-compaction flush** — automatic "save before forgetting" mechanism
5. **Privacy boundaries** — sub-agents and group chats don't see private memory

For your testing system, you could use:
- `memory/YYYY-MM-DD.md` → test run logs, discovered issues per day
- `MEMORY.md` → persistent topology knowledge, test patterns that worked/failed
- Vector search → "what tests covered BGP peering failures before?"
- Per-agent memory → each of your 6 agents could have its own workspace with domain-specific memory
