# Architectural Guidance: Multi-Agent Testing System

**Generated**: 2026-02-15
**For**: Amol's multi-agent network testing platform
**Based on**: OpenClaw architecture patterns

---

## Your System at a Glance

You have **6 agents** that need to work together in a **sequential pipeline with feedback loops**:

```
┌──────────────┐
│ Task Input   │ "Test BGP convergence on spine-leaf topology"
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR AGENT                         │
│  Breaks task into steps, manages pipeline, tracks progress   │
└──────┬───────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: TOPOLOGY                                            │
│  ┌────────────────────┐    ┌─────────────────────────────┐  │
│  │ 1. Helper Agent    │───▶│ 2. Topology Agent           │  │
│  │ "What is this      │    │ Generate topology file       │  │
│  │  task about?"      │    │ Validate → iterate → apply   │  │
│  └────────────────────┘    └──────────────┬──────────────┘  │
│                                           │ ✓ topology.yaml │
└───────────────────────────────────────────┼─────────────────┘
                                            │
                                            ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 2: TEST PLANNING                                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Orchestrator analyzes topology bits used                 │ │
│  │ Creates base test plan from:                             │ │
│  │   - Topology structure                                   │ │
│  │   - Existing templates & actions                         │ │
│  │   - Reference materials                                  │ │
│  │   - Network data                                         │ │
│  └──────────────────────────┬──────────────────────────────┘ │
└─────────────────────────────┼────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 3: ACTION & SUITE GENERATION                           │
│  ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │ 3. Action Agent  │◀──▶│ 4. Suite Creation Agent      │   │
│  │ Create API       │    │ Create test suites from plan  │   │
│  │ triggers &       │    │ Reference existing templates  │   │
│  │ validations      │    │                               │   │
│  └──────────────────┘    └──────────────┬───────────────┘   │
│                                         │                    │
│  ┌──────────────────┐                   │                    │
│  │ 5. Template Agent│◀──────────────────┘                    │
│  │ Create/update    │                                        │
│  │ test templates   │                                        │
│  └──────────────────┘                                        │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│ PHASE 4: TEST EXECUTION & DISCOVERY                          │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 6. Testbed Agent                                        │ │
│  │ Execute test cases → validate results                   │ │
│  │ Discover additional test cases → feed back to queue     │ │
│  │ Report pass/fail → iterate on failures                  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Discovery Loop ─────────────────────────────────────┐   │
│  │ New test case discovered                              │   │
│  │   → Queue it                                          │   │
│  │   → May need new actions (→ Action Agent)             │   │
│  │   → May need new templates (→ Template Agent)         │   │
│  │   → Execute when ready                                │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## Recommended Architecture (Inspired by OpenClaw)

### Core Pattern: Orchestrator + Specialized Agents

Like OpenClaw's Gateway that routes messages to agents, you need an **Orchestrator** that:
1. Receives tasks
2. Breaks them into phases
3. Dispatches to specialized agents
4. Tracks progress
5. Handles the feedback loop (discovered test cases)

```typescript
// Core architecture
interface Orchestrator {
  submitTask(task: TaskDefinition): Promise<TaskRun>;
  getStatus(runId: string): TaskRunStatus;
  abort(runId: string): void;
}

interface AgentRunner {
  execute(agent: AgentType, input: AgentInput): Promise<AgentOutput>;
  stream(agent: AgentType, input: AgentInput): AsyncIterable<AgentEvent>;
}

interface TaskQueue {
  enqueue(task: QueuedTask): void;
  dequeue(): QueuedTask | null;
  peek(): QueuedTask[];
}
```

### Component Map

```
your-system/
├── src/
│   ├── orchestrator/
│   │   ├── orchestrator.ts        # Main pipeline controller
│   │   ├── task-planner.ts        # Break task into phases
│   │   ├── task-queue.ts          # Priority queue for work items
│   │   ├── task-state.ts          # JSONL state persistence
│   │   └── discovery-handler.ts   # Handle discovered test cases
│   │
│   ├── agents/
│   │   ├── base-agent.ts          # Common agent interface
│   │   ├── helper-agent.ts        # General Q&A agent
│   │   ├── topology-agent.ts      # Topology generation & validation
│   │   ├── action-agent.ts        # API trigger/validation creation
│   │   ├── suite-agent.ts         # Test suite creation
│   │   ├── template-agent.ts      # Template creation/update
│   │   └── testbed-agent.ts       # Test execution & discovery
│   │
│   ├── execution/
│   │   ├── agent-runner.ts        # Agent execution engine
│   │   ├── command-queue.ts       # Lane-based queue (from OpenClaw)
│   │   ├── model-client.ts        # Claude API client
│   │   ├── tool-registry.ts       # Agent-specific tool registration
│   │   └── session-manager.ts     # Conversation persistence
│   │
│   ├── tools/
│   │   ├── device-tools.ts        # Device connection & data retrieval
│   │   ├── topology-tools.ts      # Topology validation, apply to device
│   │   ├── action-tools.ts        # Action CRUD, API trigger/validation
│   │   ├── template-tools.ts      # Template CRUD
│   │   ├── suite-tools.ts         # Suite management
│   │   ├── test-runner-tools.ts   # Execute tests, collect results
│   │   └── reference-tools.ts     # Search reference materials
│   │
│   ├── state/
│   │   ├── task-store.ts          # Task run persistence
│   │   ├── artifact-store.ts      # Generated artifacts (topology, actions, suites)
│   │   └── test-results-store.ts  # Test execution results
│   │
│   └── ui/  (optional)
│       ├── dashboard.ts           # Progress dashboard
│       ├── websocket-server.ts    # Event streaming to UI
│       └── event-emitter.ts       # Progress events
```

---

## Key Design Decisions

### 1. Lane-Based Command Queue (from OpenClaw)

**Why**: Your pipeline is sequential within a task but you might run multiple tasks in parallel.

```typescript
// Adapted from OpenClaw's src/process/command-queue.ts
class CommandQueue {
  private lanes = new Map<string, Promise<any>>();

  async enqueue<T>(lane: string, task: () => Promise<T>): Promise<T> {
    const prev = this.lanes.get(lane) ?? Promise.resolve();
    const next = prev.then(() => task());
    this.lanes.set(lane, next.catch(() => {}));
    return next;
  }
}

// Usage:
// Each task gets its own lane
queue.enqueue(`task:${taskId}`, () => runTopologyPhase(taskId));
// Multiple tasks run in parallel across lanes
```

### 2. Agent as Tool-Equipped LLM Session (from OpenClaw)

Each of your 6 agents is a **Claude session with specialized tools**:

```typescript
interface Agent {
  id: string;
  systemPrompt: string;      // Domain-specific instructions
  tools: Tool[];              // Agent-specific capabilities
  model: string;              // claude-opus-4-6 or sonnet
}

const topologyAgent: Agent = {
  id: "topology",
  systemPrompt: `You are a network topology expert. You generate, validate,
    and apply network topologies. You iterate until the topology is correct
    and can be applied to the device.`,
  tools: [
    createTopologyTool,       // Generate topology YAML
    validateTopologyTool,     // Validate against schema + device
    applyTopologyTool,        // Push to device
    getDeviceDataTool,        // Read current device state
    readReferenceTool,        // Read reference materials
  ],
  model: "claude-opus-4-6"
};
```

### 3. JSONL State Persistence (from OpenClaw)

Track the full lifecycle of each task run:

```jsonl
{"phase":"init","taskId":"task_001","input":"Test BGP convergence on spine-leaf","ts":"2026-02-15T10:00:00Z"}
{"phase":"topology","status":"started","agentId":"topology","ts":"2026-02-15T10:00:05Z"}
{"phase":"topology","status":"iteration","attempt":1,"result":"validation_failed","errors":["missing AS number"],"ts":"..."}
{"phase":"topology","status":"iteration","attempt":2,"result":"validation_passed","ts":"..."}
{"phase":"topology","status":"applied","deviceResponse":"ok","ts":"..."}
{"phase":"topology","status":"completed","artifact":"topology_001.yaml","ts":"..."}
{"phase":"planning","status":"started","ts":"..."}
{"phase":"planning","testPlan":{"totalCases":12,"actions":5,"templates":3},"ts":"..."}
{"phase":"actions","status":"started","agentId":"action","ts":"..."}
...
```

### 4. Discovery Queue (Your Unique Need)

OpenClaw's FollowupRun queue maps perfectly to your test discovery pattern:

```typescript
interface DiscoveryQueue {
  // During test execution, agent discovers new test cases
  addDiscoveredTest(test: DiscoveredTest): void;

  // Orchestrator checks for pending discoveries
  hasPending(): boolean;
  next(): DiscoveredTest | null;

  // Some discoveries need prerequisite work
  needsAction(test: DiscoveredTest): boolean;
  needsTemplate(test: DiscoveredTest): boolean;
}

// The orchestrator loop:
while (queue.hasPending() || currentTests.length > 0) {
  const test = queue.next();
  if (test && test.needsAction) {
    await actionAgent.createAction(test.actionSpec);
  }
  if (test && test.needsTemplate) {
    await templateAgent.createTemplate(test.templateSpec);
  }
  await testbedAgent.execute(test);
  // testbedAgent may discover more tests → queue.addDiscoveredTest()
}
```

### 5. Event Streaming (from OpenClaw)

Stream progress to any UI or monitoring system:

```typescript
interface TaskEvent {
  taskId: string;
  phase: string;
  type: "started" | "progress" | "iteration" | "completed" | "error" | "discovery";
  data: unknown;
  timestamp: string;
}

// Emit events as the pipeline progresses
emitter.emit("task.event", {
  taskId: "task_001",
  phase: "topology",
  type: "iteration",
  data: { attempt: 2, result: "validation_passed" }
});
```

---

## Pipeline Execution Flow (Detailed)

### Phase 1: Task Understanding

```
Input: "Test BGP convergence on spine-leaf topology"
  │
  └─→ Helper Agent
       System prompt: "You analyze testing tasks and produce structured requirements"
       Tools: [searchReferences, getDeviceCapabilities, listExistingTopologies]
       │
       Output: {
         taskType: "BGP convergence test",
         topology: "spine-leaf",
         protocols: ["BGP"],
         deviceRequirements: ["2+ spine switches", "4+ leaf switches"],
         testObjectives: [
           "Verify BGP sessions establish",
           "Verify route propagation",
           "Test convergence on link failure",
           "Verify ECMP load balancing"
         ]
       }
```

### Phase 2: Topology Generation

```
Topology Agent receives requirements
  │
  Loop:
  │  ├─→ Generate topology YAML
  │  ├─→ Validate (schema + device constraints)
  │  ├─→ If invalid → iterate (fix issues, regenerate)
  │  └─→ If valid → apply to device
  │       ├─→ If apply fails → iterate
  │       └─→ If apply succeeds → DONE
  │
  Output: {
    topologyFile: "topology_001.yaml",
    bitsUsed: ["bgp_spine", "bgp_leaf", "loopback_assignment"],
    deviceState: "applied",
    iterations: 2
  }
```

### Phase 3: Test Plan & Action Generation

```
Orchestrator creates test plan from:
  - Topology bits used
  - Existing templates & actions
  - Reference materials
  - Network data
  │
  ├─→ Action Agent (parallel with Suite Agent)
  │    Create API triggers:
  │      - trigger_bgp_session_reset
  │      - trigger_link_down
  │      - trigger_link_up
  │    Create validations:
  │      - validate_bgp_session_up
  │      - validate_route_count
  │      - validate_convergence_time
  │
  ├─→ Suite Agent
  │    Create test suites:
  │      - suite_bgp_basic: [establish, propagate, verify]
  │      - suite_bgp_convergence: [failover, recovery, timing]
  │      - suite_bgp_ecmp: [load_balance, path_selection]
  │
  └─→ Template Agent (if needed)
       Create/update templates:
         - template_bgp_session_check
         - template_convergence_timer
```

### Phase 4: Test Execution

```
Testbed Agent executes:
  │
  For each suite:
  │  For each test case:
  │  │  ├─→ Setup (apply pre-conditions)
  │  │  ├─→ Execute (run triggers)
  │  │  ├─→ Validate (check results)
  │  │  ├─→ Record (pass/fail + details)
  │  │  └─→ Discover? (new test cases found)
  │  │       └─→ Queue discovered tests
  │  │
  │  End suite → Report results
  │
  Check discovery queue:
  │  ├─→ New test needs new action → Action Agent
  │  ├─→ New test needs new template → Template Agent
  │  └─→ Execute discovered test
  │
  Repeat until queue empty
```

---

## Implementation Strategy

### Step 1: Core Infrastructure (Week 1)

Build the foundation that all agents share:

```
1. Model client (Claude API wrapper with streaming)
2. Tool registry (register/invoke tools)
3. Agent runner (session + tools + system prompt → execute)
4. Command queue (lane-based concurrency)
5. State persistence (JSONL task state)
```

### Step 2: Orchestrator (Week 1-2)

The brain that manages the pipeline:

```
1. Task parser (break input into phases)
2. Phase executor (sequential phase management)
3. Discovery queue (handle found test cases)
4. Progress tracker (state file updates)
5. Event emitter (for monitoring)
```

### Step 3: Individual Agents (Week 2-3)

Build each agent with its specialized tools:

```
Priority order:
1. Topology Agent (most complex — validation loop)
2. Action Agent (API integration)
3. Suite Agent (test plan generation)
4. Testbed Agent (execution + discovery)
5. Template Agent (template CRUD)
6. Helper Agent (general Q&A)
```

### Step 4: Integration & UI (Week 3-4)

Wire everything together:

```
1. End-to-end pipeline test
2. WebSocket event server (for dashboard)
3. Simple progress dashboard
4. Error handling & retry logic
```

---

## Key Patterns to Adopt from OpenClaw

| Pattern | OpenClaw Implementation | Your Implementation |
|---|---|---|
| **Lane queue** | `process/command-queue.ts` | One lane per task, serialize phases within |
| **Agent as session** | `pi-embedded-runner/run.ts` | Each agent = Claude session with tools |
| **Tool system** | `agents/pi-tools.ts` | Per-agent tool registry |
| **System prompt** | `agents/system-prompt.ts` | Per-agent prompt with domain knowledge |
| **Event streaming** | `infra/agent-events.ts` | Broadcast phase progress |
| **State file** | Session JSONL | Task state JSONL |
| **Subagent spawn** | `tools/sessions-spawn-tool.ts` | Orchestrator spawns specialized agents |
| **Retry + fallback** | `infra/retry.ts` | Retry device operations, model fallback |
| **Block chunking** | `pi-embedded-block-chunker.ts` | Stream intermediate results |

---

## Key Differences from OpenClaw

| Aspect | OpenClaw | Your System |
|---|---|---|
| **Trigger** | User message | Task definition |
| **Flow** | Reactive (respond to message) | Proactive (execute pipeline) |
| **Agents** | General purpose, user-facing | Specialized, pipeline stages |
| **Sessions** | Conversations | Task lifecycles |
| **Output** | Chat messages | Test results + artifacts |
| **Concurrency** | Multi-user, multi-channel | Multi-task, sequential phases |
| **Discovery** | N/A | New test cases during execution |
| **Validation** | Tool policy | Device + schema + result validation |
| **Iteration** | User feedback | Automated retry loops |

---

## Minimal Viable Architecture

If you want to start small and iterate:

```typescript
// Minimal orchestrator
async function executeTestTask(task: string) {
  const state = createTaskState();

  // Phase 1: Understand
  const requirements = await runAgent("helper", task, [searchReferences]);
  state.log("requirements", requirements);

  // Phase 2: Topology (with retry loop)
  let topology;
  for (let i = 0; i < 5; i++) {
    topology = await runAgent("topology", requirements, [
      createTopology, validateTopology, applyTopology
    ]);
    if (topology.valid && topology.applied) break;
  }
  state.log("topology", topology);

  // Phase 3: Actions + Suite
  const actions = await runAgent("action", { requirements, topology }, [
    createAction, listExistingActions
  ]);
  const suite = await runAgent("suite", { requirements, topology, actions }, [
    createSuite, listTemplates, listActions
  ]);
  state.log("suite", suite);

  // Phase 4: Execute (with discovery loop)
  const discoveryQueue = new Queue();
  discoveryQueue.addAll(suite.testCases);

  while (!discoveryQueue.isEmpty()) {
    const testCase = discoveryQueue.next();
    const result = await runAgent("testbed", testCase, [
      executeTest, validateResult, reportResult
    ]);

    if (result.discoveredTests) {
      for (const discovered of result.discoveredTests) {
        // May need prerequisites
        if (discovered.needsAction) {
          await runAgent("action", discovered.actionSpec, [createAction]);
        }
        discoveryQueue.add(discovered);
      }
    }
    state.log("test_result", result);
  }

  return state.summary();
}
```

This is ~50 lines of orchestration logic. The complexity lives in:
1. The tools (device APIs, validation, test execution)
2. The agent system prompts (domain knowledge)
3. The retry/iteration loops

Start here, then add streaming, UI, and parallel execution as needed.
