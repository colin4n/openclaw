# 09 — Config & Secrets

**Source files**: `src/config/`, `src/config/io.ts`, `src/config/schema.ts`, `src/config/sessions/`, `src/secrets/`

---

## 1. Overview

OpenClaw's configuration and secrets systems manage:

- **Config**: the main `openclaw.json` file — gateway settings, agent defaults, channel config, model providers, hooks, etc.
- **Sessions**: mapping from channel/account identifiers to AI session files
- **Secrets**: runtime snapshot of sensitive values (API keys, tokens) that are resolved and cached at gateway start

---

## 2. Config File

### 2.1 Location

Default: `~/.openclaw/openclaw.json`

Can be overridden with:
- `--config <path>` CLI flag
- `OPENCLAW_CONFIG` environment variable

### 2.2 Schema

The config schema is defined using Zod in `src/config/zod-schema.ts` (28KB) and validated by `src/config/validation.ts`.

Key top-level sections:

| Section | Description |
|---------|-------------|
| `gateway` | Gateway server settings (port, bind, auth, TLS) |
| `agents` | Agent defaults (model, sandbox, tools, concurrency) |
| `agents.list[]` | Per-agent configurations |
| `channels` | Per-channel settings |
| `plugins` | Installed plugins list |
| `hooks` | Internal hooks (Gmail watcher, HTTP webhooks, scheduled) |
| `cron` | Scheduled tasks |
| `session` | Session store path, pruning settings |
| `memory` | Memory backend config |
| `skills` | Skill entries |
| `secrets` | Secret references (paths, env vars, key files) |
| `logging` | Log levels, redaction patterns |
| `tools` | Tool policy (exec, fs, workspaceOnly) |
| `routing` | Message routing policy |
| `identity` | Bot identity (name, avatar, per-channel prefix) |

### 2.3 Loading Flow

```
openclaw.json file
    │
    ▼
src/config/io.ts — loadConfig()
    │  1. Read file (with backup on read error)
    │  2. Parse JSON
    │  3. Apply env var substitution (${SOME_ENV_VAR} in values)
    │  4. Apply includes (config fragments from other files)
    │  5. Normalize (path expansion, legacy field rename)
    │  6. Migrate legacy config
    ▼
Zod schema validation (zod-schema.ts)
    │
    ▼
src/config/validation.ts — semantic validation
    │  (cross-field validation rules, e.g., allowlist requires allowFrom)
    ▼
Loaded config object (OpenClawConfig)
```

### 2.4 Config Includes

`src/config/includes.ts` — config can include fragments from other files:

```json
{
  "$include": ["./work-channels.json", "./personal-settings.json"],
  "gateway": { ... }
}
```

Included files are deep-merged into the main config.

### 2.5 Environment Variable Substitution

`src/config/env-substitution.ts` — config values can reference env vars:

```json
{
  "agents": {
    "defaults": {
      "model": "${OPENCLAW_DEFAULT_MODEL}"
    }
  }
}
```

This allows keeping sensitive values in `.env` files rather than the JSON config.

### 2.6 Legacy Migration

`src/config/legacy.ts` and `src/config/legacy.migrations.part-*.ts` — automatically migrates old config formats to the current schema. OpenClaw has renamed fields and restructured the config several times; the migration system handles this transparently.

---

## 3. Config Schema Details

### 3.1 Agent Defaults

`src/config/types.agent-defaults.ts` — the `agents.defaults` section controls:
- `model` — default model provider and model name
- `models` — allowlist of permitted models
- `sandbox` — sandbox mode settings
- `tools` — tool policy (which tools are enabled, allowlist)
- `concurrency` — max concurrent sessions
- `thinking` — reasoning budget for supported models

### 3.2 Gateway Config

`src/config/types.gateway.ts` — gateway settings:
- `bind` — network bind (`loopback`, `all`, `tailscale`, or specific IP)
- `port` — port number (default 18789)
- `auth` — auth mode (token, device auth, trusted proxy)
- `controlUi` — web dashboard settings
- `tls` — TLS configuration

### 3.3 Secret Input Syntax

Secrets in config can be:
```json
{
  "apiKey": "sk-abc123",                    // plain value
  "apiKey": "${OPENAI_API_KEY}",            // env var
  "apiKey": { "$file": "~/.secrets/key" }, // file content
  "apiKey": { "$env": "MY_API_KEY" }       // explicit env ref
}
```

Resolved by `src/config/zod-schema.secret-input-validation.ts`.

---

## 4. Session Store

Sessions map an inbound message context (channel + sender) to a persistent AI session file.

### 4.1 Session Key

`src/config/sessions/` — session key derivation:

A **session key** is a stable string identifier for a conversation thread. Examples:
- `dm:telegram:123456789` — a direct message from Telegram user 123456789
- `group:slack:C0123456789` — a Slack channel conversation
- Custom agent session keys: `custom:work-project`

The session key determines:
1. Which transcript file to load
2. Which agent ID the session belongs to
3. Where the session is stored

### 4.2 Session Store

`src/config/sessions/store.ts` — `loadSessionStore()` reads the session store JSON file at `~/.openclaw/sessions.json` (or agent-specific path). The store maps session keys to session metadata:

```json
{
  "dm:telegram:123456789": {
    "sessionId": "last-used-session-uuid",
    "model": "anthropic:claude-opus-4",
    "createdAt": "2026-01-15T10:00:00Z"
  }
}
```

### 4.3 Agent ID Resolution

`src/config/sessions/main-session.ts` — `resolveMainSessionKey()` and `resolveAgentIdFromSessionKey()` resolve which agent handles a given session.

The gateway can run multiple agents (`agents.list[]`), each with different tools, sandbox settings, and capabilities.

### 4.4 Session Paths

`src/config/sessions/paths.ts` — `resolveStorePath()` resolves the filesystem path for a session based on:
- Config `session.store` setting
- Agent ID
- State directory (`~/.openclaw/`)

Transcript files: `~/.openclaw/agents/<agentId>/sessions/<sessionKey>.jsonl`

---

## 5. Secrets Runtime

The secrets runtime manages API keys and tokens at gateway runtime.

### 5.1 Runtime Snapshot

`src/secrets/runtime.ts` — the **runtime snapshot** is an in-memory copy of all resolved secrets, built at gateway startup:

```
Gateway starts
    │
    ▼
prepareSecretsRuntimeSnapshot()
    │  Resolves all $file, $env, and plain secret values
    │  Validates API key formats
    ▼
activateSecretsRuntimeSnapshot()
    │  Stores snapshot in memory
    ▼
Agent/model calls use secrets from snapshot
    │  resolveCommandSecretsFromActiveRuntimeSnapshot()
    ▼
On config reload:
    │  New snapshot prepared
    │  clearSecretsRuntimeSnapshot()
    │  activateSecretsRuntimeSnapshot() with new values
```

### 5.2 Auth Surface States

`src/secrets/runtime-gateway-auth-surfaces.ts` — `GATEWAY_AUTH_SURFACE_PATHS` defines the distinct auth surfaces (default token, device auth, control UI, etc.) and `evaluateGatewayAuthSurfaceStates()` computes which auth methods are currently active.

### 5.3 Credential Precedence

`src/gateway/credentials.ts` — when multiple auth methods are configured, there is a defined precedence order. The `resolveCredentialsForRequest()` function applies this precedence consistently.

---

## 6. Config Backup and Recovery

`src/config/backup-rotation.ts` — before writing a config change, a backup is created. Up to N rotating backups are kept. If the config file becomes corrupted, the system can recover from the most recent backup.

---

## 7. Config Redaction

`src/config/redact-snapshot.ts` (~700 LOC) — before logging or displaying config (e.g., in `openclaw config show`), sensitive fields (API keys, tokens, passwords) are redacted. This prevents secrets from appearing in logs or the Control UI.

The redaction rules are defined in `redact-snapshot.secret-ref.ts` and applied recursively to the config object.

---

## 8. Key Config Files Summary

| File | Role |
|------|------|
| `src/config/io.ts` | `loadConfig()` — full config loading pipeline |
| `src/config/schema.ts` | Schema types and validation entry |
| `src/config/zod-schema.ts` | Main Zod schema (28KB) |
| `src/config/validation.ts` | Semantic config validation |
| `src/config/env-substitution.ts` | `${ENV_VAR}` substitution |
| `src/config/includes.ts` | `$include` config fragments |
| `src/config/legacy.ts` | Legacy config migration |
| `src/config/defaults.ts` | Default config values |
| `src/config/paths.ts` | Config and state directory paths |
| `src/config/backup-rotation.ts` | Config backup before writes |
| `src/config/redact-snapshot.ts` | Config redaction for display |
| `src/config/sessions/` | Session store and key management |
| `src/config/types.*.ts` | Per-section type definitions |
| `src/secrets/runtime.ts` | Secrets runtime snapshot |
| `src/secrets/runtime-gateway-auth-surfaces.ts` | Auth surface states |
| `src/gateway/credentials.ts` | Credential precedence resolution |
