# 13 — Hooks System

**Source files**: `src/hooks/internal-hooks.ts`, `src/hooks/loader.ts`, `src/hooks/install.ts`, `src/hooks/workspace.ts`

---

## 1. Overview

OpenClaw has **two hook systems** that are distinct but complementary:

| System | Location | Configured via | Purpose |
|--------|---------|---------------|---------|
| **Internal hooks** | `src/hooks/` | `hooks.internal` in config + bundled | TypeScript/JS event handlers |
| **Plugin hooks** | `src/plugins/hooks.ts` | Plugin `init()` function | Plugin lifecycle interception |

This document covers the **internal hooks system** (`src/hooks/`). For plugin hooks, see `07-plugin-sdk.md` §5.

---

## 2. Internal Hook Events

`src/hooks/internal-hooks.ts` defines the hook event types:

```typescript
// Top-level event types
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message";

// Specific events (type:action format)
type InternalHookEvent =
  | AgentBootstrapHookEvent        // "agent:bootstrap"
  | GatewayStartupHookEvent        // "gateway:startup"
  | MessageReceivedHookEvent       // "message:received"
  | MessageSentHookEvent           // "message:sent"
  | MessageTranscribedHookEvent    // "message:transcribed"
  | MessagePreprocessedHookEvent   // "message:preprocessed"
  // ... command and session events
```

Each event carries:
- `type` and `action` — event identity
- `sessionKey` — which conversation thread fired the event
- `context` — gateway deps and config
- `timestamp` — ISO timestamp
- `messages` — transcript messages (where relevant)

---

## 3. Hook Registration

`internal-hooks.ts` — the global registry is a `globalThis` singleton (to survive bundle splits):

```typescript
// Register a handler for a specific event
registerInternalHook("message:received", async (event) => {
  // event is typed to MessageReceivedHookEvent
});

// Register for all events of a type
registerInternalHook("agent", async (event) => {
  // fires on any agent:* event
});

// Unregister
const handle = registerInternalHook("gateway:startup", handler);
unregisterInternalHook(handle);
```

Handlers run **in parallel** with isolated error handling — one failing handler does not block others.

---

## 4. Bundled Hooks

Three hooks are bundled with OpenClaw (always available, enabled by default):

### 4.1 `session-memory`
- **Event**: `session:end`
- **Purpose**: Extracts notable facts from the session transcript and writes them to the memory index (see `12-memory-system.md` §8)
- **Location**: `src/hooks/bundled/session-memory/handler.ts`

### 4.2 `command-logger`
- **Event**: `command:*`
- **Purpose**: Logs command executions to a structured log for auditing
- **Location**: `src/hooks/bundled/command-logger/handler.ts`

### 4.3 `boot-md`
- **Event**: `agent:bootstrap`
- **Purpose**: Loads markdown files referenced in `BOOT.md` into the agent's bootstrap context
- **Location**: `src/hooks/bundled/boot-md/handler.ts`

---

## 5. Hook Discovery and Loading

`src/hooks/loader.ts` — hook discovery scans three locations:

```
1. Bundled hooks     → src/hooks/bundled/*/handler.ts
2. Managed hooks     → ~/.openclaw/agents/<id>/hooks/*/handler.ts (installed via CLI)
3. Workspace hooks   → <workspace>/hooks/*/handler.ts
```

For each discovered hook:
1. Config eligibility is checked (`config.ts`) — is this hook enabled?
2. Module is dynamically loaded via `jiti` (supports TypeScript and ESM/CJS)
3. Handler is registered in the global registry

`src/hooks/module-loader.ts` handles the ESM/CJS boundary, using jiti for TypeScript hooks in workspaces.

---

## 6. Hook Installation

`src/hooks/install.ts` — hooks can be installed as npm packages:

```bash
openclaw hooks install @openclaw/hook-gmail-watcher
```

Installation steps:
1. Validates npm package spec
2. Extracts tarball to `~/.openclaw/agents/<id>/hooks/<name>/`
3. Runs `npm install --omit=dev` in the hook directory
4. Persists manifest (version, install date)

**Update mode**: `openclaw hooks update <name>` re-runs the install, replacing the old version.

---

## 7. User-Configured Hooks (YAML/config)

In addition to TypeScript handlers, users can configure hooks in `openclaw.json`:

```json
{
  "hooks": {
    "message.received": [
      {
        "type": "http",
        "url": "https://my-webhook.example.com/openclaw",
        "method": "POST"
      }
    ],
    "gateway.startup": [
      {
        "type": "shell",
        "command": "notify-send 'OpenClaw started'"
      }
    ]
  }
}
```

These config-based hooks are dispatched by `src/gateway/hooks.ts` (gateway hook dispatcher) — distinct from the internal hook registry.

---

## 8. Gmail Watcher

One important built-in hook (not bundled — installed separately) is the **Gmail watcher**:
- Monitors a Gmail inbox via Gmail API watch
- On new email: fires `message:received` to the agent with the email content
- Effectively makes Gmail a "channel" without the channel adapter overhead

---

## 9. Key Files Summary

| File | Role |
|------|------|
| `src/hooks/internal-hooks.ts` | Global registry, event types, register/trigger/unregister |
| `src/hooks/loader.ts` | Hook discovery from bundled/managed/workspace |
| `src/hooks/install.ts` | npm package hook installation and updates |
| `src/hooks/workspace.ts` | Workspace hook directory scanning |
| `src/hooks/config.ts` | Hook configuration and eligibility checks |
| `src/hooks/module-loader.ts` | ESM/CJS dynamic module loading via jiti |
| `src/hooks/bundled/session-memory/` | Session memory extraction hook |
| `src/hooks/bundled/command-logger/` | Command audit logger hook |
| `src/hooks/bundled/boot-md/` | Bootstrap markdown loader hook |
| `src/gateway/hooks.ts` | Gateway-level hook dispatch (config-based hooks) |
