# OpenClaw - Model Selection, Defaults & Fallback Hierarchy

**Generated**: 2026-02-15
**Source**: Deep scan of model resolution, fallback, and configuration code

---

## Quick Reference: Resolution Priority

When OpenClaw needs to decide which model to use for an agent run, it checks in this order (first match wins):

| Priority | Source | Config Location | Scope |
|----------|--------|-----------------|-------|
| 1 | `/model` directive | Chat command during session | Per-turn |
| 2 | Session stored override | `SessionEntry.modelOverride` | Per-session (sticky) |
| 3 | Parent session override | Inherited from parent session | Per-session |
| 4 | Agent-specific model | `agents.list[].model` | Per-agent |
| 5 | Global default model | `agents.defaults.model` | All agents |
| 6 | Built-in hardcoded default | `anthropic/claude-opus-4-6` | Ultimate fallback |

---

## Setting the Default Model

### Global Default

The primary model for all agents is set in `openclaw.json`:

```json5
{
  "agents": {
    "defaults": {
      // Simple form: just a string
      "model": "anthropic/claude-opus-4-6",

      // OR object form with fallbacks:
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": [
          "anthropic/claude-sonnet-4-5-20250929",
          "openai/gpt-5.2"
        ]
      }
    }
  }
}
```

**Source**: `src/agents/model-selection.ts` — `resolveConfiguredModelRef()`

**Format**: `"provider/model-id"` (e.g., `"anthropic/claude-opus-4-6"`, `"openai/gpt-5.2"`)

If no model is configured anywhere, the hardcoded default is:
```typescript
// src/agents/defaults.ts
export const DEFAULT_PROVIDER = "anthropic";
export const DEFAULT_MODEL = "claude-opus-4-6";
```

### Per-Agent Override

Each agent in `agents.list[]` can specify its own model:

```json5
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-6"  // global default
    },
    "list": [
      {
        "id": "main",
        "name": "Main Assistant"
        // ← uses global default (claude-opus-4-6)
      },
      {
        "id": "coding",
        "name": "Coder",
        "model": "anthropic/claude-sonnet-4-5-20250929"
        // ← overrides to sonnet for this agent only
      },
      {
        "id": "research",
        "name": "Researcher",
        "model": {
          "primary": "openai/gpt-5.2",
          "fallbacks": ["anthropic/claude-opus-4-6"]
        }
        // ← different primary + agent-specific fallbacks
      }
    ]
  }
}
```

**Source**: `src/agents/agent-scope.ts` — `resolveAgentModelPrimary()` and `resolveAgentModelFallbacksOverride()`

**Key behavior**: Agent-level `fallbacks` override the global fallbacks. An explicitly empty `fallbacks: []` disables fallback for that agent entirely.

### Session-Level Override (`/model` command)

During a chat session, users can switch models with the `/model` directive:

```
/model claude-sonnet-4-5
/model openai/gpt-5.2
/model fast           ← fuzzy matching with alias support
```

This is **sticky** — once set, it persists for all future turns in that session until changed or reset.

**Source**: `src/auto-reply/reply/model-selection.ts` — `resolveModelDirectiveSelection()`

**Features**:
- Fuzzy matching (Levenshtein distance ≤ 3)
- Alias support (e.g., `fast` → `anthropic/claude-sonnet-4-5-20250929`)
- Allowlist enforcement
- Inherits to child/spawned sessions

### Heartbeat Model Override

Heartbeat (proactive background) runs can use a different model:

```json5
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "model": "anthropic/claude-sonnet-4-5-20250929"
        // ← cheaper model for background checks
      }
    }
  }
}
```

When `heartbeat.model` is set, it **skips session-level overrides** so the heartbeat always uses the configured model.

### Subagent Model Override

Spawned sub-agents can have their own default model:

```json5
{
  "agents": {
    "defaults": {
      "subagents": {
        "model": "anthropic/claude-sonnet-4-5-20250929",
        // OR with fallbacks:
        "model": {
          "primary": "anthropic/claude-sonnet-4-5-20250929",
          "fallbacks": ["anthropic/claude-haiku-4-5-20251001"]
        }
      }
    }
  }
}
```

Per-agent subagent model overrides are also supported:

```json5
{
  "agents": {
    "list": [
      {
        "id": "main",
        "subagents": {
          "model": "anthropic/claude-sonnet-4-5-20250929"
        }
      }
    ]
  }
}
```

---

## Fallback Chain: What Happens When a Model Fails

### How Fallbacks Are Assembled

**Source**: `src/agents/model-fallback.ts` — `resolveFallbackCandidates()`

The fallback candidate list is built in this order:

```
1. Primary model (from resolution hierarchy above)
2. Explicit fallbacks from config:
   - Agent-specific: agents.list[].model.fallbacks[]
   - OR global: agents.defaults.model.fallbacks[]
3. Configured default model (as final safety net)
```

**Deduplication**: If a model appears in multiple places, it's only tried once.

**Allowlist enforcement**: Fallback models must be in the model allowlist (if one is configured).

### What Triggers a Fallback

**Source**: `src/agents/model-fallback.ts` — `runWithModelFallback()`

Fallback is triggered on these errors:

| Error Type | Triggers Fallback? |
|---|---|
| Network/connection failure | Yes |
| Rate limiting (429) | Yes |
| Authentication failure (401) | Yes |
| Timeout | Yes |
| Provider internal error (500) | Yes |
| **Context overflow** | **No** — rethrown immediately |
| **User abort** | **No** — rethrown immediately |
| **Non-failover errors** | **No** — rethrown immediately |

### Auth Profile Cooldown

When a provider's auth profile fails (e.g., rate limited), it enters a **cooldown** period. During cooldown:
- That profile is skipped in fallback attempts
- If ALL profiles for a provider are in cooldown, the entire provider is skipped
- Other providers in the fallback chain are still tried

**Source**: `src/agents/auth-profiles.ts` — `isProfileInCooldown()`

### Fallback Execution Flow

```
runWithModelFallback()
  │
  ├─ Candidate 1: anthropic/claude-opus-4-6
  │   ├─ Check auth profiles for anthropic → any available?
  │   │   └─ All in cooldown? → SKIP, push to attempts[]
  │   ├─ Run agent with this model
  │   │   ├─ Success → return result
  │   │   ├─ Failover error → push to attempts[], continue
  │   │   ├─ Context overflow → RETHROW (don't try smaller model)
  │   │   └─ Abort → RETHROW
  │
  ├─ Candidate 2: anthropic/claude-sonnet-4-5-20250929 (fallback)
  │   └─ Same flow as above
  │
  ├─ Candidate 3: openai/gpt-5.2 (fallback)
  │   └─ Same flow as above
  │
  └─ All failed → throw Error("All models failed (3): anthropic/claude-opus-4-6: rate_limit | ...")
```

### Image Model Fallback

Image-capable models have their own separate fallback chain:

```json5
{
  "agents": {
    "defaults": {
      "imageModel": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["openai/gpt-5.2"]
      }
    }
  }
}
```

**Source**: `src/agents/model-fallback.ts` — `runWithImageModelFallback()`

---

## Model Allowlist (Restricting Available Models)

If you define `agents.defaults.models`, it acts as an **allowlist** — only listed models can be used:

```json5
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-opus-4-6": {},
        "anthropic/claude-sonnet-4-5-20250929": { "alias": "fast" },
        "openai/gpt-5.2": { "alias": "gpt" }
      }
    }
  }
}
```

**Key rules**:
- If `models` is **empty or omitted** → any model is allowed
- If `models` has entries → **only those models** can be used (via `/model` or as fallbacks)
- The primary default model is always allowed regardless
- Each entry can have an `alias` for quick `/model` switching
- Entries can have `params` for provider-specific settings (e.g., thinking mode)
- Entries can set `streaming: false` (useful for Ollama)

**Source**: `src/agents/model-selection.ts` — `buildAllowedModelSet()`

---

## Model Aliases

Aliases let users type short names instead of full `provider/model` identifiers:

```json5
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-opus-4-6": { "alias": "opus" },
        "anthropic/claude-sonnet-4-5-20250929": { "alias": "sonnet" },
        "openai/gpt-5.2": { "alias": "gpt" }
      }
    }
  }
}
```

Then in chat:
```
/model opus    →  switches to anthropic/claude-opus-4-6
/model sonnet  →  switches to anthropic/claude-sonnet-4-5-20250929
/model gpt     →  switches to openai/gpt-5.2
```

**Built-in aliases** (no config needed):
| Short Form | Resolves To |
|---|---|
| `opus-4.6` | `claude-opus-4-6` |
| `opus-4.5` | `claude-opus-4-5` |
| `sonnet-4.5` | `claude-sonnet-4-5` |

**Source**: `src/agents/model-selection.ts` — `buildModelAliasIndex()`

---

## Thinking Mode Configuration

### Default Thinking Level

```json5
{
  "agents": {
    "defaults": {
      "thinkingDefault": "high"  // off | minimal | low | medium | high | xhigh
    }
  }
}
```

**Resolution order**:
1. `/think <level>` directive in chat → per-turn override
2. `agents.defaults.thinkingDefault` → global config
3. Model has `reasoning` capability in catalog → defaults to `"low"`
4. Otherwise → `"off"`

### Thinking Levels

| Level | Description |
|---|---|
| `off` | No extended thinking |
| `minimal` | Minimal reasoning |
| `low` | Light reasoning (default for reasoning-capable models) |
| `medium` | Moderate reasoning |
| `high` | Deep reasoning |
| `xhigh` | Maximum reasoning (only supported by specific models) |

**xhigh-capable models**: Selected OpenAI Codex models (gpt-5.3-codex variants).

**Subagent thinking**: Can be set independently:
```json5
{
  "agents": {
    "defaults": {
      "thinkingDefault": "high",       // main agents
      "subagents": {
        "thinking": "low"              // cheaper for sub-agents
      }
    }
  }
}
```

---

## Provider Configuration

### Defining Providers

Providers are configured under `models.providers`:

```json5
{
  "models": {
    "providers": {
      "anthropic": {
        // Usually auto-configured via auth profiles
      },
      "openai": {
        "apiKey": "sk-...",
        "baseUrl": "https://api.openai.com/v1"  // optional override
      },
      "ollama": {
        "baseUrl": "http://localhost:11434",
        "models": [
          { "id": "llama3", "contextTokens": 8192 }
        ]
      },
      "custom-provider": {
        "api": "openai-responses",  // API compatibility mode
        "apiKey": "...",
        "baseUrl": "https://my-api.example.com/v1",
        "models": [
          { "id": "my-model", "contextTokens": 32000 }
        ]
      }
    }
  }
}
```

### Provider Auto-Discovery

Some providers are automatically discovered based on environment variables or auth profiles:

| Provider | Auto-Discovered When |
|---|---|
| `anthropic` | Auth profile exists or `ANTHROPIC_API_KEY` set |
| `openai` | `OPENAI_API_KEY` set |
| `amazon-bedrock` | AWS credentials available |
| `github-copilot` | Copilot auth profile exists |
| `google` | Google auth configured |

**Not auto-discovered** (must be explicitly configured):
- `ollama`, `vllm`, `huggingface` (local/custom providers)

### Provider ID Normalization

| Input | Normalized To |
|---|---|
| `z.ai`, `z-ai` | `zai` |
| `opencode-zen` | `opencode` |
| `qwen` | `qwen-portal` |
| `kimi-code` | `kimi-coding` |

**Source**: `src/agents/model-selection.ts` — `normalizeProviderId()`

---

## Model Discovery Chain

When OpenClaw needs to resolve a model for API calls:

**Source**: `src/agents/pi-embedded-runner/model.ts`

```
1. Check Pi SDK ModelRegistry (built-in catalog)
   → Known models like claude-opus-4-6, gpt-5.2, etc.

2. Check inline provider models (models.providers[provider].models[])
   → Custom/local models defined in config

3. Forward compatibility fallbacks
   → e.g., claude-opus-4-6 → claude-opus-4-5 if 4.6 unavailable
   → Handles version migrations automatically

4. Generic provider fallback
   → If provider is configured but model isn't in catalog
   → Creates synthetic model with default context window

5. Error: "Unknown model"
```

### Forward Compatibility

**Source**: `src/agents/model-forward-compat.ts`

When a newer model isn't available in the SDK yet, it falls back automatically:

| Requested Model | Falls Back To |
|---|---|
| `claude-opus-4-6` | `claude-opus-4-5` |
| `gpt-5.3-codex` | `gpt-5.2-codex` |
| `glm-5` | `glm-4.7` |

---

## Auth Resolution for Models

When a model's provider needs authentication:

**Source**: `src/agents/model-auth.ts` — `resolveApiKeyForProvider()`

```
1. Explicit profile ID (if specified)
2. Provider auth override (e.g., "aws-sdk" for Bedrock)
3. Auth profile store (ordered, respecting cooldown)
4. Environment variables:
   - ANTHROPIC_OAUTH_TOKEN or ANTHROPIC_API_KEY
   - OPENAI_API_KEY
   - etc.
5. Custom provider API key from config (models.providers[].apiKey)
6. AWS SDK default (for amazon-bedrock)
7. Error: "No API key found"
```

### Environment Variable Mapping

| Provider | Environment Variables (checked in order) |
|---|---|
| `anthropic` | `ANTHROPIC_OAUTH_TOKEN`, `ANTHROPIC_API_KEY` |
| `openai` | `OPENAI_API_KEY` |
| `google` | `GOOGLE_API_KEY`, `GEMINI_API_KEY` |
| `amazon-bedrock` | AWS SDK credentials |
| `github-copilot` | `GITHUB_TOKEN` |

---

## Complete Example Configuration

Here's a full example showing all model-related settings:

```json5
{
  "agents": {
    "defaults": {
      // Primary model + fallbacks
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": [
          "anthropic/claude-sonnet-4-5-20250929",
          "openai/gpt-5.2"
        ]
      },

      // Image-capable model
      "imageModel": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["openai/gpt-5.2"]
      },

      // Model allowlist + aliases
      "models": {
        "anthropic/claude-opus-4-6": { "alias": "opus" },
        "anthropic/claude-sonnet-4-5-20250929": { "alias": "sonnet" },
        "openai/gpt-5.2": { "alias": "gpt" }
      },

      // Thinking
      "thinkingDefault": "high",

      // Heartbeat uses cheaper model
      "heartbeat": {
        "model": "anthropic/claude-sonnet-4-5-20250929"
      },

      // Sub-agents use cheaper model
      "subagents": {
        "model": {
          "primary": "anthropic/claude-sonnet-4-5-20250929",
          "fallbacks": ["anthropic/claude-haiku-4-5-20251001"]
        },
        "thinking": "low"
      }
    },

    // Per-agent overrides
    "list": [
      {
        "id": "main",
        "name": "Main"
        // ← inherits global defaults
      },
      {
        "id": "coding",
        "name": "Coder",
        "model": {
          "primary": "anthropic/claude-sonnet-4-5-20250929",
          "fallbacks": []  // ← explicitly disable fallbacks
        }
      }
    ]
  },

  "models": {
    "providers": {
      "anthropic": {},
      "openai": {
        "apiKey": "sk-..."
      }
    }
  }
}
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| [defaults.ts](src/agents/defaults.ts) | Hardcoded defaults: `anthropic/claude-opus-4-6` |
| [model-selection.ts](src/agents/model-selection.ts) | Model ref parsing, normalization, aliases, allowlist, thinking defaults |
| [model-fallback.ts](src/agents/model-fallback.ts) | Fallback chain execution, error classification, retry logic |
| [agent-scope.ts](src/agents/agent-scope.ts) | Per-agent model resolution, effective fallback computation |
| [model-auth.ts](src/agents/model-auth.ts) | Provider → API key resolution, env var mapping |
| [pi-embedded-runner/model.ts](src/agents/pi-embedded-runner/model.ts) | Model discovery via Pi SDK registry + config |
| [model-forward-compat.ts](src/agents/model-forward-compat.ts) | Version migration fallbacks |
| [model-catalog.ts](src/agents/model-catalog.ts) | Model catalog loading and caching |
| [models-config.providers.ts](src/agents/models-config.providers.ts) | Provider setup, auto-discovery, inline models |
| [auto-reply/reply/model-selection.ts](src/auto-reply/reply/model-selection.ts) | `/model` directive handling, fuzzy matching, session overrides |
| [config/types.agent-defaults.ts](src/config/types.agent-defaults.ts) | TypeScript types for all agent default settings |
| [config/types.agents.ts](src/config/types.agents.ts) | TypeScript types for per-agent config |

---

## Behavioral Rules Summary

1. **Explicit always wins**: `/model` directive > session override > agent config > global default
2. **Session overrides are sticky**: Once set with `/model`, persists for all future turns until changed
3. **Parent inheritance**: Spawned sessions inherit parent's model override (unless heartbeat overrides)
4. **Fallbacks only on failure**: Primary is always tried first; fallbacks trigger only on specific errors
5. **Context overflow never falls back**: A model failing on context size won't try a (likely smaller) fallback
6. **Allowlist gating**: If `agents.defaults.models` is defined, only those models can be used
7. **Agent fallbacks override global**: Per-agent `fallbacks` completely replaces global fallbacks
8. **Empty fallbacks = no fallback**: Setting `fallbacks: []` on an agent disables fallback entirely
9. **Auth cooldown**: Failed auth profiles are skipped during cooldown, enabling other providers to be tried
10. **Forward compat**: Newer model IDs auto-fallback to older versions if unavailable in SDK
