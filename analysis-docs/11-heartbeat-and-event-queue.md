# OpenClaw - Heartbeat System & Event Queue

**Generated**: 2026-02-15
**Focus**: What the heartbeat does, when it wakes up, and how the event queue works

---

## What Is the Heartbeat?

The heartbeat is a **periodic polling mechanism** that runs full agent turns at regular intervals. It lets the agent proactively check for work, relay scheduled reminders, and surface anything that needs attention — without the user sending a message.

Think of it as a cron job that wakes the agent, says "anything need attention?", and if the agent says "nope", goes back to sleep silently.

---

## When Does It Wake Up?

### Triggers

| Trigger | Reason | Priority |
|---------|--------|----------|
| **Interval timer** | Scheduled wake (default: every 30 min) | Low |
| **Manual wake** | `openclaw system event --mode now` | High |
| **Cron event** | Scheduled reminder fires | High |
| **Exec completion** | Background command finished | High |
| **External hook** | Application-triggered wake | High |
| **Retry** | Previous wake skipped (queue busy) | Low |

### Priority-Based Coalescing

If multiple wake requests come in close together (within 250ms), the system coalesces them — higher-priority reasons (manual, exec, cron) preempt lower ones (interval, retry). This prevents thrashing.

---

## What Happens When Heartbeat Fires

### Pre-Flight Checks (in order)

```
Heartbeat timer fires
  │
  ├─ Is heartbeat enabled (globally + for this agent)? → No: skip
  ├─ Is it within active hours? → No: reschedule for next valid window
  ├─ Is the main command queue empty? → No: retry in 1 second
  ├─ Is HEARTBEAT.md effectively empty? → Yes + no events: skip
  │
  └─ All checks pass → Run agent turn
```

### The Agent Turn

```
1. Read HEARTBEAT.md (if it exists in workspace)
2. Resolve delivery target (which channel to send reply to)
3. Build prompt:
   Default: "Read HEARTBEAT.md if it exists. Follow it strictly.
             Do not infer or repeat old tasks from prior chats.
             If nothing needs attention, reply HEARTBEAT_OK."
4. Run full agent turn in main session
   - Full session context available
   - Optional different model (cost savings)
   - All tools available
5. Process reply:
   - "HEARTBEAT_OK" → strip token, suppress delivery (silent)
   - Actual content → deliver to channel
```

### The HEARTBEAT_OK Token

This is a special acknowledgment that means "nothing needs attention."

| Agent Reply | What Happens |
|------------|-------------|
| `"HEARTBEAT_OK"` | Stripped, delivery suppressed (silent) |
| `"HEARTBEAT_OK. All quiet."` | Stripped; if remaining text ≤ 300 chars, also suppressed |
| `"Alert: server down. HEARTBEAT_OK"` | Token stripped, remaining text delivered |
| `"Status: HEARTBEAT_OK at 3pm"` | Token in middle — not special, delivered as-is |

The 300-char threshold (`ackMaxChars`) is configurable.

---

## Configuration

### Basic Setup

```json5
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",         // Interval (default: 30m, 1h for OAuth users)
        "model": "anthropic/claude-sonnet-4-5-20250929",  // Optional cheaper model
        "target": "last",       // "last" | "none" | channel id
        "to": "+15551234567",   // Optional explicit recipient
        "prompt": "Custom heartbeat prompt...",  // Override default prompt
        "ackMaxChars": 300,     // Max chars after HEARTBEAT_OK before delivery
        "includeReasoning": false  // Show model's reasoning?
      }
    }
  }
}
```

### Active Hours (Time Window)

Restrict heartbeats to specific hours:

```json5
{
  "heartbeat": {
    "activeHours": {
      "start": "09:00",          // HH:MM (inclusive)
      "end": "22:00",            // HH:MM (exclusive, "24:00" allowed)
      "timezone": "America/New_York"  // "user" | "local" | IANA timezone
    }
  }
}
```

Outside the window, heartbeats are **skipped and rescheduled** for the next valid time.

### Per-Agent Heartbeat

Per-agent overrides are supported. **Important rule**: if any agent has an explicit heartbeat config, only those agents run heartbeats (all others are disabled).

```json5
{
  "agents": {
    "list": [
      {
        "id": "main",
        "heartbeat": { "every": "1h" }    // Main checks hourly
      },
      {
        "id": "monitor",
        "heartbeat": { "every": "5m" }    // Monitor checks every 5 min
      }
    ]
  }
}
```

### Delivery Target

| `target` Value | Behavior |
|---------------|----------|
| `"last"` (default) | Deliver to the last external channel the user interacted with |
| `"none"` | Run internally, no external delivery |
| `"telegram"` / `"slack"` / etc. | Deliver to a specific channel |

---

## Special Heartbeat Prompts

Heartbeat isn't just for polling HEARTBEAT.md. It also relays **system events**:

### Cron Event Relay

When a scheduled reminder fires:
```
Prompt becomes: "A scheduled reminder has been triggered.
The reminder content is: [EVENT TEXT].
Please relay this reminder to the user in a helpful and friendly way."
```

### Exec Completion Relay

When a background command finishes:
```
Prompt becomes: "An async command you ran earlier has completed.
The result is shown in the system messages above.
Please relay the command output to the user in a helpful way."
```

The system **peeks** at pending events to decide which prompt to use, but doesn't consume them — the main session can still process them.

---

## The Event Queue

### System Events

System events are **ephemeral, in-memory, session-scoped** queues:

```
enqueueSystemEvent(text, { sessionKey, contextKey })  → add event
peekSystemEventEntries(sessionKey)                    → view without consuming
drainSystemEventEntries(sessionKey)                   → view and consume
```

| Event Source | contextKey Example | What It Contains |
|-------------|-------------------|-----------------|
| Cron reminder | `"cron:reminder-123"` | Reminder text |
| Exec completion | `"exec-event"` | Command output / exit code |
| Manual | `"manual"` | User-specified text |
| Hook | `"hook:custom"` | Application-defined payload |

### How Events Interact with Heartbeat

```
Events queued → Heartbeat wakes
  → Peek at pending events
  → Detect event type (cron? exec? manual?)
  → Choose appropriate prompt
  → Agent processes events in context
  → Delivers meaningful relay to user
  → Events remain queued for main session (not consumed)
```

---

## The Command Queue (Lane System)

Agent runs are serialized per-session using a **lane-based command queue**:

### How It Works

```
Lane: "session:agent:main:telegram:direct:123"
  ├── Run 1 (in progress) ← agent processing
  ├── Run 2 (queued)      ← waiting
  └── Run 3 (queued)      ← waiting

Lane: "session:agent:coding:slack:channel:456"
  └── Run 1 (in progress) ← runs in parallel with above lane
```

**Key rules**:
- **Per-session serialization**: One agent turn at a time per session
- **Cross-session parallelism**: Different sessions run independently
- **Heartbeat gating**: Heartbeat checks if main lane is empty before running. If busy → skip and retry in 1 second.

### Why Heartbeat Checks the Queue

Heartbeats should never interrupt user conversations. If the queue has pending user messages, the heartbeat defers:

```
Heartbeat wakes → queueSize("main") > 0?
  → Yes: skip, retry in 1 second
  → No: proceed with heartbeat turn
```

---

## The FollowupRun Queue

When messages arrive while the agent is already running, they're queued as **FollowupRuns**:

### How It Works

```
User sends "message A" → agent starts processing
User sends "message B" → queued as FollowupRun
User sends "message C" → queued as FollowupRun
Agent finishes "message A" → picks up "message B"
Agent finishes "message B" → picks up "message C"
```

### Queue Modes

| Mode | Behavior |
|------|----------|
| `"queue"` | Standard FIFO queue |
| `"steer"` | Queue all, drain via background worker |
| `"followup"` | Queue for multi-turn responses |
| `"collect"` | Batch messages before running |
| `"interrupt"` | Preempt existing messages |

### Deduplication

Messages are deduplicated by:
- `"message-id"` (default): skip if same message ID already queued
- `"prompt"`: skip if same prompt text already queued
- `"none"`: no dedup

### Drop Policy (when queue is full)

| Policy | Behavior |
|--------|----------|
| `"old"` | Drop oldest item |
| `"new"` | Reject new item |
| `"summarize"` | Consolidate/summarize items |

---

## Session Lifecycle with Heartbeat

```
┌──────────────────────────────────────────────────────┐
│ HEARTBEAT RUNNER (per agent)                          │
│                                                       │
│  Timer fires every N minutes                          │
│    → Pre-flight checks                                │
│    → Queue empty? Active hours? HEARTBEAT.md exists?  │
│    → Run agent turn in main session                   │
│    → Process reply (strip HEARTBEAT_OK if present)    │
│    → Deliver to target channel (or suppress if OK)    │
│    → Restore session timestamp (don't keep alive)     │
│    → Reschedule next wake                             │
└──────────────────────────────────────────────────────┘

Important: Heartbeat-only replies do NOT keep sessions alive.
The session's `updatedAt` is restored to its pre-heartbeat value
to prevent "ghost sessions" from heartbeat pings.
```

---

## For Your Multi-Agent System

The heartbeat pattern is directly applicable to your testing system:

1. **Periodic polling** — your orchestrator agent could heartbeat to check for new test tasks, topology changes, or test results that need attention
2. **Event relay** — when a test suite completes or a device goes offline, queue a system event and let heartbeat relay it
3. **HEARTBEAT.md as a task list** — your agents could maintain a checklist of pending validations, and heartbeat polls it
4. **Active hours** — restrict expensive test runs to off-peak hours
5. **Lane-based serialization** — ensure one test run per topology at a time, while different topologies test in parallel
6. **FollowupRun queue** — if multiple test requests arrive while a suite is running, queue them for sequential execution
