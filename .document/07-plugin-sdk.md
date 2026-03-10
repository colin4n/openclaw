# 07 — Plugin SDK

**Source files**: `src/plugin-sdk/index.ts`, `src/plugins/`, `src/plugins/types.ts`, `src/plugins/hooks.ts`, `src/plugins/loader.ts`, `extensions/*/`

---

## 1. Overview

OpenClaw has an **extensive plugin system** that allows extending the gateway with:
- New channel integrations (channel plugins)
- Memory backends (memory plugins)
- Auth providers (OAuth/token exchange for model providers)
- Lifecycle hooks (before/after agent events)
- Custom agent tools
- HTTP route handlers
- CLI commands

The plugin SDK is exported as a separate import path: `openclaw/plugin-sdk`.

---

## 2. Plugin Types

`src/plugins/types.ts` defines the plugin contract. There are several plugin kinds:

### 2.1 Channel Plugin

The most common type. Implements the `ChannelPlugin` interface (see `06-channels.md`). Adds a new messaging platform to the gateway.

Example: `extensions/matrix/` adds Matrix protocol support.

### 2.2 Memory Plugin (`PluginKind = "memory"`)

Provides a memory backend for the agent. Only one memory plugin can be active at a time.

Examples:
- `extensions/memory-core/` — built-in file-based memory
- `extensions/memory-lancedb/` — vector database memory with semantic search

### 2.3 Context Engine Plugin (`PluginKind = "context-engine"`)

Provides additional context to agent sessions. Can inject information into the system prompt or tool results.

### 2.4 Auth Provider Plugin

Adds a new OAuth or token-exchange flow for a model provider. Used by:
- `extensions/google-gemini-cli-auth/` — Google Gemini CLI authentication
- `extensions/minimax-portal-auth/` — MiniMax portal auth
- `extensions/qwen-portal-auth/` — Qwen portal auth

### 2.5 General Plugin

General-purpose plugins that hook into lifecycle events, register tools, add HTTP routes, or register CLI commands.

---

## 3. Plugin Lifecycle

```
1. Discovery
   src/plugins/discovery.ts
   └─ scans config.plugins[] entries
   └─ scans bundled extensions in extensions/
   └─ finds npm-installed plugins in ~/.openclaw/plugins/

2. Install (first time)
   src/plugins/install.ts
   └─ npm install --omit=dev in plugin dir
   └─ validates npm integrity
   └─ pins version in plugin store

3. Load
   src/plugins/loader.ts — loadOpenClawPlugins()
   └─ dynamic import of plugin's main entry
   └─ validates manifest (package.json "openclaw" field)
   └─ calls plugin.init() if defined

4. Register
   src/plugins/registry.ts
   └─ adds channel plugins to channel catalog
   └─ registers hooks
   └─ registers tools
   └─ registers HTTP handlers

5. Enable
   src/plugins/enable.ts
   └─ plugin is marked enabled in config
   └─ channel plugins start their gateway adapter

6. Runtime
   └─ hooks fire at lifecycle events
   └─ tools are available to agent sessions
   └─ HTTP handlers respond to routes
```

---

## 4. Plugin SDK API Surface

The public SDK is exported from `src/plugin-sdk/index.ts` (727 lines of exports). Key groups:

### 4.1 Channel Types

```typescript
// Channel plugin interface types
export type { ChannelPlugin, ChannelConfigSchema } from "./types.plugin.js";
export type {
  ChannelMessagingAdapter,
  ChannelOutboundAdapter,
  ChannelGatewayAdapter,
  ChannelSetupAdapter,
  ChannelStatusAdapter,
  ChannelAuthAdapter,
  // ... all adapter types
} from "./types.adapters.js";

export type {
  ChannelCapabilities,
  ChannelId,
  ChannelMeta,
  // ... core types
} from "./types.core.js";
```

### 4.2 Plugin Infrastructure

```typescript
// Webhook helpers (for webhook-based channels)
export { WebhookTargets } from "./webhook-targets.js";
export { WebhookRequestGuards } from "./webhook-request-guards.js";
export { WebhookMemoryGuards } from "./webhook-memory-guards.js";

// Auth helpers
export { FetchAuth } from "./fetch-auth.js";
export { OAuthUtils } from "./oauth-utils.js";

// File utilities
export { JsonStore } from "./json-store.js";
export { FileLock } from "./file-lock.js";

// Queuing
export { KeyedAsyncQueue } from "./keyed-async-queue.js";
export { PersistentDedupe } from "./persistent-dedupe.js";
```

### 4.3 Inbound/Outbound

```typescript
// Inbound message envelope handling
export type { InboundEnvelope } from "./inbound-envelope.js";

// Outbound media
export { OutboundMedia } from "./outbound-media.js";
export { ReplyPayload } from "./reply-payload.js";
```

### 4.4 Security

```typescript
// SSRF protection for outbound HTTP calls
export { SsrfPolicy } from "./ssrf-policy.js";

// Group access control
export { GroupAccess } from "./group-access.js";
export { AllowFrom } from "./allow-from.js";
```

### 4.5 Channel-Specific SDK Extras

The `openclaw/plugin-sdk` package has sub-paths for channel-specific helpers:
```
openclaw/plugin-sdk/telegram   → Telegram-specific helpers
openclaw/plugin-sdk/discord    → Discord-specific helpers  
openclaw/plugin-sdk/slack      → Slack-specific helpers
openclaw/plugin-sdk/whatsapp   → WhatsApp-specific helpers
// etc.
```

---

## 5. Hook System

`src/plugins/hooks.ts` — the hook system allows plugins to intercept lifecycle events.

### 5.1 Hook Types

| Hook Event | Trigger |
|------------|---------|
| `before-agent-start` | Before a new agent session begins |
| `model-override` | Allows plugin to override which model is used |
| `after-tool-call` | After each tool invocation |
| `compaction` | Before/after context compaction |
| `session-end` | When a session ends |
| `message-inbound` | When a message arrives (before agent) |
| `message-outbound` | Before a reply is sent |
| `gateway-startup` | When the gateway starts |
| `gateway-stop` | When the gateway shuts down |

### 5.2 Hook Registration

Plugins register hooks in their `init()` function:

```typescript
// In a plugin's init function:
export async function init(ctx: PluginContext) {
  ctx.hooks.register({
    event: "before-agent-start",
    handler: async (hookCtx) => {
      // Modify session settings, inject context, etc.
      return { ...hookCtx, systemPromptExtra: "Additional context..." };
    }
  });
}
```

### 5.3 Internal Hooks vs. Plugin Hooks

- **Internal hooks** (`src/hooks/`): YAML/JS-based hooks configured in `openclaw.json` under `hooks.*`. These are the user-facing automation hooks (Gmail watcher, HTTP webhooks, scheduled triggers).
- **Plugin hooks** (`src/plugins/hooks.ts`): TypeScript hooks registered by plugins at load time.

---

## 6. Plugin Manifest

Every plugin must have a valid `package.json` with an `"openclaw"` field:

```json
{
  "name": "@openclaw/matrix",
  "version": "2026.3.3",
  "openclaw": {
    "type": "channel",
    "id": "matrix",
    "displayName": "Matrix",
    "description": "Matrix protocol integration"
  },
  "dependencies": {
    "matrix-js-sdk": "..."
  },
  "devDependencies": {
    "openclaw": "workspace:*"  // NOT in dependencies!
  }
}
```

**Critical rule**: `openclaw` must be in `devDependencies` or `peerDependencies`, NOT `dependencies`. The runtime resolves `openclaw/plugin-sdk` via jiti alias. Putting `openclaw` in `dependencies` breaks `npm install --omit=dev` plugin installs.

---

## 7. Extension Development Patterns

### 7.1 Channel Extension Structure

A typical channel extension (`extensions/telegram/`) contains:
```
extensions/telegram/
├── package.json          # npm package with "openclaw" manifest field
├── src/
│   └── channel.ts        # ChannelPlugin implementation
└── dist/                 # compiled output (built by tsdown)
```

The `channel.ts` file exports a `ChannelPlugin` object.

### 7.2 Plugin vs. Core Boundary

Rule: if a feature is optional (a specific platform, a memory backend, an OAuth flow), it belongs in `extensions/`, not `src/`.

The bar for adding to core is high: the feature must be used by the majority of users, or there is a security reason it must be in core.

### 7.3 Runtime Dependencies vs. Dev Dependencies

- Channel SDK helpers (`openclaw/plugin-sdk`) → resolved at runtime via jiti alias → put in `devDependencies`
- Channel-specific libraries (e.g., `matrix-js-sdk`) → needed at runtime → put in `dependencies`

This matters because plugins are installed with `npm install --omit=dev`.

---

## 8. Plugin Config Schema

`src/plugins/schema-validator.ts` — plugins can declare a config schema to enable:
- Config validation (Zod or custom `validate()`)
- Control UI config form rendering (`uiHints`)
- JSON Schema for documentation

---

## 9. Plugin Slots

`src/plugins/slots.ts` — defines the concept of **plugin slots**: named slots that can hold only one plugin at a time. Currently used for:
- Memory slot: only one memory plugin active at a time
- Context engine slot: only one context engine at a time

---

## 10. Key Files Summary

| File | Role |
|------|------|
| `src/plugin-sdk/index.ts` | Public SDK exports (727 lines) |
| `src/plugins/types.ts` | Plugin type definitions |
| `src/plugins/loader.ts` | Plugin discovery and loading |
| `src/plugins/registry.ts` | Plugin registration |
| `src/plugins/hooks.ts` | Hook system implementation |
| `src/plugins/discovery.ts` | Plugin discovery from config + npm |
| `src/plugins/install.ts` | Plugin installation from npm |
| `src/plugins/manifest.ts` | Plugin manifest validation |
| `src/plugins/slots.ts` | Exclusive plugin slots (memory, etc.) |
| `src/plugins/schema-validator.ts` | Plugin config schema validation |
| `src/plugins/runtime/` | Plugin runtime environment |
| `src/plugins/services.ts` | Plugin services lifecycle |
| `extensions/*/` | All extension implementations |
