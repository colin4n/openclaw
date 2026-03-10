# 02 — Overall Architecture

**Source files**: `src/entry.ts`, `src/index.ts`, `src/gateway/server.impl.ts`, `src/gateway/server-startup.ts`

---

## 1. System Overview

OpenClaw is a **multi-layered, event-driven AI gateway**. At the highest level it has three layers:

```
┌──────────────────────────────────────────────────────────────────┐
│  CLI Layer  (src/cli/, src/commands/)                            │
│  Entry point, program builder, command routing, DI wiring        │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gateway Layer  (src/gateway/)                                   │
│  Long-running HTTP/WS server, auth, config reload, cron,         │
│  control UI, channel health monitor, exec approval manager       │
└──────┬──────────────────┬─────────────────┬───────────────────┬──┘
       │                  │                 │                   │
       ▼                  ▼                 ▼                   ▼
┌──────────┐  ┌──────────────────┐  ┌───────────┐  ┌──────────────┐
│ Agents   │  │ Channels         │  │ Plugins   │  │ Infra        │
│ src/     │  │ src/channels/    │  │ src/      │  │ src/infra/   │
│ agents/  │  │ extensions/*/    │  │ plugins/  │  │              │
│          │  │                  │  │           │  │              │
│ Pi runner│  │ Telegram, Slack, │  │ Memory,   │  │ Exec, fs,    │
│ Sessions │  │ Discord, Signal, │  │ Hooks,    │  │ Heartbeat,   │
│ Tools    │  │ iMessage, etc.   │  │ Auth-prov │  │ Retry, etc.  │
└──────────┘  └──────────────────┘  └───────────┘  └──────────────┘
```

---

## 2. Layer Responsibilities

### 2.1 CLI Layer (`src/cli/`, `src/commands/`)

- **Entry**: `src/entry.ts` — process bootstrap, respawn guard, profile wiring.
- **Program**: `src/cli/program.ts` → `src/cli/program/` — Commander-based CLI program builder.
- **Commands**: `src/commands/` — implementations for `agent`, `gateway`, `message`, `channels`, `models`, `plugins`, etc.
- **DI**: `src/cli/deps.ts` — `createDefaultDeps()` wires send functions lazily (dynamic imports at runtime boundary).
- **Profile**: `src/cli/profile.ts` — `--profile` flag to load per-profile env vars.

The CLI layer is the **thin dispatch layer**. Heavy logic lives in the Gateway or Agents modules, not commands.

### 2.2 Gateway Layer (`src/gateway/`)

The Gateway is the **always-on control plane**. It is a long-running HTTP + WebSocket server that:

- Authenticates incoming connections and messages
- Routes inbound messages to the agent engine
- Manages channel lifecycle (start, stop, health monitor)
- Runs cron jobs
- Serves the Control UI (web dashboard)
- Handles config reload without restart
- Manages exec approvals and node (remote host) connections

Key files:
- `server.impl.ts` — main gateway server implementation (~1,000 LOC)
- `server-startup.ts` — sidecar startup (browser control, hooks, channels, plugins, memory)
- `server-http.ts` — HTTP route setup
- `server-channels.ts` — channel manager
- `server-chat.ts` — inbound message → agent dispatch
- `boot.ts` — BOOT.md one-time startup agent run
- `config-reload.ts` — live config reload without restart

### 2.3 Agents Layer (`src/agents/`)

The Agents layer contains the **AI session engine**. It:

- Wraps LLM provider calls (Anthropic, OpenAI, Gemini, Ollama, etc.) via the "Pi embedded runner"
- Manages session lifecycle (create, run, compact, persist)
- Implements the tool pipeline (policy → invoke → result guard)
- Spawns and manages sub-agents
- Loads skills and injects them into system prompts
- Handles model auth, failover, and catalog management

### 2.4 Channels Layer (`src/channels/`, `extensions/`)

- Defines the **channel plugin interface** (`ChannelPlugin` type)
- Built-in channels live in `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/`
- Extension channels live in `extensions/` (Matrix, MS Teams, Zalo, Feishu, etc.)
- All channels implement the same adapter interface, making them interchangeable to the gateway

### 2.5 Plugins Layer (`src/plugins/`)

- Plugin **discovery**, **install** (npm), **load**, **register**, **enable**
- Hook system: plugins can register hooks for lifecycle events
- Separate from channel plugins — this layer handles general plugins (memory, context engine, auth providers, etc.)

### 2.6 Infra Layer (`src/infra/`)

Shared infrastructure utilities used by all other layers:
- Exec approval system (safe-bins, allowlist, obfuscation detection)
- `fs-safe.ts`: atomic file operations, boundary path enforcement
- Port management, lock files
- Heartbeat runner (ghost reminder, scheduling)
- Retry/backoff utilities
- Update runner (check, download, apply)
- Bonjour/mDNS discovery

---

## 3. Runtime Boundaries

### 3.1 Gateway Lock

When the gateway starts (`openclaw gateway run`), it acquires a **lock file** to prevent duplicate gateway instances on the same port. The lock is in `src/infra/gateway-lock.ts`.

### 3.2 Process Respawn

The CLI entry (`src/entry.ts`) may **respawn itself** with additional Node flags (`--disable-warning=ExperimentalWarning`) before running the actual CLI. This is transparent to the user but important for understanding the startup sequence.

### 3.3 Module Boundary: `dist/` vs `src/`

- In development: `src/` is run directly via Bun or tsx.
- In production: `pnpm build` compiles to `dist/` (tsdown/rollup), and `openclaw.mjs` is the entry wrapper.
- The `plugin-sdk` is a separate export path: `openclaw/plugin-sdk`.

### 3.4 Extension Isolation

Extensions (plugins in `extensions/`) are loaded as npm packages at runtime. They must declare runtime deps in `dependencies` (not `devDependencies`). They resolve `openclaw/plugin-sdk` via a jiti alias at runtime.

---

## 4. Data Flow: Inbound Message → Agent Response

```
User sends message (Telegram / WhatsApp / Slack / etc.)
       │
       ▼
Channel adapter receives message (polling or webhook)
       │
       ▼
Channel dock (src/channels/dock.ts) validates and envelopes the message
       │
       ▼
Gateway server-chat.ts dispatches to agent engine
       │
       ▼
Pi embedded runner creates/resumes a session
       │
       ▼
System prompt is built (identity, skills, memory, tools)
       │
       ▼
LLM API call (Anthropic/OpenAI/etc.)
       │
       ▼
Tool calls are intercepted and dispatched through tool pipeline
│  (each tool: policy check → invoke → result guard)
       │
       ▼
Final reply is assembled (text + optional attachments)
       │
       ▼
Outbound adapter sends reply back to the channel
       │
       ▼
Session transcript is persisted (~/.openclaw/agents/<id>/sessions/)
```

---

## 5. Configuration Flow

```
openclaw.json (~/.openclaw/openclaw.json)
       │
       ▼
src/config/io.ts — loadConfig() reads, parses, validates via Zod schemas
       │
       ▼
src/config/validation.ts — validates config against schema
       │
       ▼
Gateway holds config in memory
       │
       ▼
src/gateway/config-reload.ts — watches for file changes, triggers reload
       │
       ▼
Server reload handlers apply changes without restart
```

---

## 6. Key Design Patterns

### 6.1 Dependency Injection via `createDefaultDeps()`

All gateway commands receive a `deps` object created by `createDefaultDeps()` in `src/cli/deps.ts`. This wires send functions lazily (dynamic imports) so they are only loaded when needed. It also makes testing easy — tests inject mock deps.

### 6.2 Plugin Adapter Pattern

All channel integrations implement the `ChannelPlugin` interface (defined in `src/channels/plugins/types.plugin.ts`). The gateway does not know about specific channels — it only knows about `ChannelPlugin` instances. This means adding a new channel requires only implementing the adapter, not changing the gateway.

### 6.3 Subsystem Logger

Every subsystem creates its own logger via `createSubsystemLogger(name)` (in `src/logging/subsystem.ts`). This provides structured, filterable logs per subsystem.

### 6.4 Dynamic Import at Runtime Boundaries

Per `AGENTS.md`: "do not mix `await import('x')` and static `import ... from 'x'` for the same module in production code paths. If you need lazy loading, create a dedicated `*.runtime.ts` boundary."

This pattern is visible in `src/cli/deps.ts` (each send function has a `*.runtime.ts` file).

---

## 7. Key Files Summary

| File | Layer | Role |
|------|-------|------|
| `src/entry.ts` | CLI | Process bootstrap, respawn guard |
| `src/index.ts` | CLI | Legacy entry, exports public API |
| `src/cli/program.ts` | CLI | Commander program builder |
| `src/cli/deps.ts` | CLI | `createDefaultDeps()` DI factory |
| `src/gateway/server.impl.ts` | Gateway | Main server implementation |
| `src/gateway/server-startup.ts` | Gateway | Sidecar startup sequence |
| `src/gateway/server-http.ts` | Gateway | HTTP route setup |
| `src/gateway/server-channels.ts` | Gateway | Channel lifecycle manager |
| `src/gateway/server-chat.ts` | Gateway | Inbound message → agent dispatch |
| `src/gateway/boot.ts` | Gateway | BOOT.md startup agent run |
| `src/agents/pi-embedded-runner/` | Agents | LLM session runner |
| `src/agents/subagent-registry.ts` | Agents | Subagent lifecycle management |
| `src/channels/plugins/types.plugin.ts` | Channels | ChannelPlugin interface |
| `src/plugins/loader.ts` | Plugins | Plugin discovery and loading |
| `src/infra/` | Infra | Exec, fs, ports, heartbeat utilities |
