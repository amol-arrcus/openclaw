# OpenClaw - How It Connects to Claude Code (Without an API Key)

**Generated**: 2026-02-15
**For**: Understanding the OAuth credential flow when using a Claude Max subscription

---

## The Short Answer (Your Setup)

You used the **setup-token** method. During `openclaw onboard`, you ran `claude setup-token`, which generated a token tied to your Max subscription. That token is stored in OpenClaw's auth profile store and used directly for API calls.

```
You ran: claude setup-token
  → Claude Code generated a token tied to your Max subscription
    → You pasted it during openclaw onboard
      → Stored in: ~/.openclaw/agents/main/agent/auth-profiles.json
        → Profile: "anthropic:claude-token-001" (type: "token")
          → OpenClaw uses this token directly for Anthropic API calls
```

> **Your Keychain IS populated** with Claude Code OAuth credentials (service: `Claude Code-credentials`, account: `amoliyer`), but OpenClaw is **not using those**. It's using the setup-token from `auth-profiles.json` instead.

---

## Your Authentication Chain (Setup-Token Method)

### Step 1: You Generated a Setup Token

You ran `claude setup-token` in your terminal. Claude Code generated a one-time token tied to your Max subscription and displayed it.

### Step 2: You Pasted It During Onboarding

During `openclaw onboard`, you pasted that token. OpenClaw stored it in its auth profile store:

**File**: `~/.openclaw/agents/main/agent/auth-profiles.json`

```json
{
  "version": 1,
  "profiles": {
    "anthropic:claude-token-001": {
      "type": "token",
      "provider": "anthropic",
      "token": "<your-setup-token>"
    }
  },
  "lastGood": {
    "anthropic": "anthropic:claude-token-001"
  }
}
```

### Step 3: Auth Resolution on Each Agent Run

**File**: `src/agents/auth-profiles/oauth.ts` ([line 165](src/agents/auth-profiles/oauth.ts#L165))

When OpenClaw needs to call the Anthropic API:

```
resolveApiKeyForProfile("anthropic:claude-token-001")
  │
  └─ cred.type === "token"
      → Return cred.token directly as the API key
      → No refresh needed, no Keychain access
```

### Step 4: API Call

**File**: `src/agents/model-auth.ts`

The token is passed directly to the Anthropic Messages API:

```
getApiKeyForModel({ model: "claude-opus-4-6", ... })
  → resolveApiKeyForProvider({ provider: "anthropic", ... })
    → resolveAuthProfileOrder() → ["anthropic:default"]
    → resolveApiKeyForProfile("anthropic:default")
      → Returns { apiKey: "eyJhbG...", mode: "oauth" }
  → Pi SDK uses this token in: Authorization: Bearer eyJhbG...
    → POST https://api.anthropic.com/v1/messages
```

---

## Complete Flow Diagram (Your Setup)

```
┌──────────────────────────────────────────────────────────┐
│ YOU: ran `claude setup-token`                            │
│   → Claude Code generated a token tied to Max sub        │
│   → You pasted it during `openclaw onboard`              │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│ AUTH PROFILE STORE:                                      │
│   ~/.openclaw/agents/main/agent/auth-profiles.json       │
│                                                          │
│   profiles: {                                            │
│     "anthropic:claude-token-001": {                      │
│       type: "token",                                     │
│       provider: "anthropic",                             │
│       token: "<setup-token>"                             │
│     }                                                    │
│   }                                                      │
│   lastGood: { anthropic: "anthropic:claude-token-001" }  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       │ On each agent run
                       ▼
┌──────────────────────────────────────────────────────────┐
│ MODEL AUTH RESOLUTION:                                   │
│   src/agents/model-auth.ts                               │
│                                                          │
│   resolveApiKeyForProvider("anthropic")                   │
│     → Check auth profile order                           │
│     → Load "anthropic:claude-token-001" profile          │
│     → type === "token" → return token directly           │
│     → No Keychain access, no refresh needed              │
└──────────────────────┬───────────────────────────────────┘
                       │
                       │ Token as API key
                       ▼
┌──────────────────────────────────────────────────────────┐
│ PI SDK → ANTHROPIC API:                                  │
│                                                          │
│   POST https://api.anthropic.com/v1/messages             │
│   x-api-key: <setup-token>                               │
│                                                          │
│   Body: { model: "claude-opus-4-6", messages: [...] }    │
└──────────────────────────────────────────────────────────┘
```

---

## Auth Resolution Priority

When OpenClaw needs credentials for the Anthropic provider, it checks in this order:

| Priority | Source | How |
|---|---|---|
| 1 | **Auth profile store** | `auth-profiles.json` → profile matching provider |
| 2 | **Environment variable** | `ANTHROPIC_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` |
| 3 | **Config file** | `models.providers.anthropic.apiKey` in openclaw.json |

Within the auth profile store, profiles are tried in **order** (configurable):
```json
{
  "order": {
    "anthropic": ["anthropic:default", "anthropic:work"]
  }
}
```

If one profile fails (expired + refresh fails), it tries the next.

---

## Alternative: OAuth/Keychain Path (Not Your Setup)

OpenClaw also supports reading OAuth credentials directly from Claude Code's stored login. This is used when onboarding selects the "OAuth" option instead of "setup-token":

```
1. Claude Code stores OAuth tokens in macOS Keychain:
     Service: "Claude Code-credentials"
     Value: { claudeAiOauth: { accessToken, refreshToken, expiresAt } }
   Or on Linux/Windows: ~/.claude/.credentials.json

2. OpenClaw reads them via readClaudeCliCredentials()
     src/agents/cli-credentials.ts
     → macOS: security find-generic-password -s "Claude Code-credentials" -w
     → Fallback: read ~/.claude/.credentials.json

3. Stored as an "oauth" type profile in auth-profiles.json
     → Requires automatic token refresh when expired
     → Refreshed tokens written back to Claude CLI store
```

This method is more complex (token refresh, Keychain access) but keeps credentials in sync between Claude Code and OpenClaw.

---

## Three Auth Methods for Anthropic

| Method | Type | Credential | Token Refresh |
|---|---|---|---|
| **Claude Max OAuth** | `oauth` | accessToken + refreshToken from `~/.claude/.credentials.json` or Keychain | Automatic via Pi SDK refresh endpoint |
| **Setup Token** | `token` | One-time token from `claude setup-token` | May have fixed expiry |
| **API Key** | `api_key` | `sk-ant-...` from Anthropic Console | Never expires |

Your case (Claude Max, no API key) uses the **second method** — a setup-token generated by `claude setup-token`, stored as `anthropic:claude-token-001` in `auth-profiles.json`.

---

## Key Source Files

| File | Purpose |
|---|---|
| [cli-credentials.ts](src/agents/cli-credentials.ts) | Read Claude CLI OAuth tokens from Keychain or `~/.claude/.credentials.json` |
| [auth-profiles/store.ts](src/agents/auth-profiles/store.ts) | Load/save/merge auth profiles, sync from external CLIs |
| [auth-profiles/oauth.ts](src/agents/auth-profiles/oauth.ts) | Resolve OAuth profiles, refresh expired tokens with file lock |
| [model-auth.ts](src/agents/model-auth.ts) | Map provider → API key using profiles, env vars, config |
| [auth-profiles/order.ts](src/agents/auth-profiles/order.ts) | Determine which profile to try first |
| [auth-profiles/external-cli-sync.ts](src/agents/auth-profiles/external-cli-sync.ts) | Sync credentials from Qwen CLI, MiniMax CLI |
| [commands/auth-choice.apply.anthropic.ts](src/commands/auth-choice.apply.anthropic.ts) | Onboarding: setup-token and API key flows |

---

## For Your Multi-Agent System

If you want your system to reuse the Claude Max subscription the same way, you have two practical options:

### Option A: Use `claude setup-token` (Recommended — What You're Already Using)

Run `claude setup-token` once, save the token in your app's config, and use it directly as the API key. No Keychain reading needed.

```typescript
// Token from: claude setup-token
const SETUP_TOKEN = process.env.ANTHROPIC_SETUP_TOKEN;

const response = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "x-api-key": SETUP_TOKEN,
    "anthropic-version": "2023-06-01",
    "content-type": "application/json"
  },
  body: JSON.stringify({
    model: "claude-opus-4-6",
    max_tokens: 4096,
    messages: [{ role: "user", content: "Hello" }]
  })
});
```

### Option B: Read Claude CLI Credentials from Keychain (Alternative)

If you prefer automatic credential sharing without pasting tokens:

```typescript
import { execSync } from "node:child_process";

function getClaudeOAuthToken(): { access: string; refresh: string; expires: number } | null {
  // macOS: read from Keychain
  if (process.platform === "darwin") {
    try {
      const raw = execSync(
        'security find-generic-password -s "Claude Code-credentials" -w',
        { encoding: "utf8", timeout: 5000, stdio: ["pipe", "pipe", "pipe"] }
      ).trim();
      const data = JSON.parse(raw);
      const oauth = data?.claudeAiOauth;
      if (oauth?.accessToken && oauth?.refreshToken) {
        return { access: oauth.accessToken, refresh: oauth.refreshToken, expires: oauth.expiresAt };
      }
    } catch {}
  }
  // Linux/Windows: read from ~/.claude/.credentials.json
  // (see cli-credentials.ts for full implementation)
  return null;
}

// Usage — note: requires token refresh handling for expired tokens
const creds = getClaudeOAuthToken();
if (creds && Date.now() < creds.expires) {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "x-api-key": creds.access,
      "anthropic-version": "2023-06-01",
      "content-type": "application/json"
    },
    body: JSON.stringify({
      model: "claude-opus-4-6",
      max_tokens: 4096,
      messages: [{ role: "user", content: "Hello" }]
    })
  });
}
```

**Option A is recommended** — it's what you're already using, and it's simpler (no Keychain access, no token refresh logic).
