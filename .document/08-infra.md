# 08 — Infrastructure (infra)

**Source files**: `src/infra/`

---

## 1. Overview

`src/infra/` is the **shared infrastructure utilities layer**. It provides foundational capabilities used by all other modules:

- Exec approval system (safe-bins, allowlists, obfuscation detection)
- Atomic filesystem operations with boundary enforcement
- Port management and gateway locking
- Heartbeat runner (proactive messaging)
- Retry and backoff utilities
- Update runner (check, download, apply)
- Bonjour/mDNS discovery
- SSH tunnels and Tailscale integration
- Device pairing
- Push notification (APNS)

This layer does NOT contain business logic — it is pure infrastructure.

---

## 2. Exec Approval System

This is one of the most important security systems in OpenClaw. It controls whether the agent can execute shell commands on the host.

### 2.1 Safe-Bin Policy

`src/infra/exec-approvals.ts` — The safe-bin system defines which commands are "always safe" and do not require human approval.

There are several policy levels:
- **`safe-bin-policy-profiles.ts`**: predefined profiles (`strict`, `default`, `permissive`)
- **`exec-safe-bin-trust.ts`**: validates a binary against the safe-bin trust policy
- **`exec-safe-bin-policy.ts`**: the composed policy (profile + custom allowlist)

Default profile allows common dev tools: `cat`, `ls`, `echo`, `grep`, `git`, `npm`, etc. Risky commands like `rm -rf`, `curl | bash`, or anything with pipes through `sh` require approval.

### 2.2 Allowlist

`src/infra/exec-approvals-allowlist.ts` — operator-configured allowlist patterns. When a command matches an allowlist pattern, it is auto-approved without human intervention.

Allowlist patterns can be:
- Exact command strings
- Glob patterns
- Regex patterns
- Named safe-bin entries

### 2.3 Obfuscation Detection

`src/infra/exec-obfuscation-detect.ts` — detects attempts to obfuscate commands that could bypass approval:
- Base64-encoded commands (`echo <base64> | base64 -d | sh`)
- Hex-encoded commands
- Variable injection tricks
- Process substitution abuse

Detected obfuscation triggers a warning and may force approval even if the outer command is on the allowlist.

### 2.4 Approval Flow

```
Agent wants to run: rm -rf /tmp/build
  │
  ▼
exec-approvals.ts — evaluateExecApproval()
  │  1. Is the command on the safe-bin list? → auto-approve
  │  2. Is there a matching allowlist entry? → auto-approve
  │  3. Is obfuscation detected? → force approval required
  │  4. Is exec.profile = "allow-all"? → auto-approve (dangerous!)
  ▼
If approval required:
  │
  ▼
exec-approval-forwarder.ts — forwards to gateway's ExecApprovalManager
  │
  ▼
Human sees approval request in Control UI or CLI
  │
  ▼
Human approves → command runs
Human denies → command rejected, agent gets error
```

---

## 3. `fs-safe.ts` — Atomic File Operations

`src/infra/fs-safe.ts` (~500 LOC) provides safe file system operations:

### 3.1 Atomic Writes

Instead of `fs.writeFile` (which can leave partial files on crash), `fs-safe` writes to a temp file and then atomically renames:
```
write /path/to/file:
  1. Write content to /path/to/file.tmp.<random>
  2. fsync (flush to disk)
  3. rename /path/to/file.tmp.<random> → /path/to/file (atomic)
```

### 3.2 Boundary Path Enforcement

`src/infra/boundary-path.ts` (~24KB) ensures file operations stay within allowed directories. This is critical for:
- `tools.fs.workspaceOnly: true` — restricts reads/writes to workspace
- Sandbox path validation
- Plugin temp path restrictions

Any attempt to read/write outside the allowed boundary is rejected with an error.

### 3.3 Safe Open

`src/infra/safe-open-sync.ts` — synchronous safe file open that validates the path before opening, preventing TOCTOU race conditions.

---

## 4. Port Management

### 4.1 Gateway Lock

`src/infra/gateway-lock.ts` — when the gateway starts, it acquires an exclusive lock file tied to the configured port. This prevents two gateway instances from starting on the same port.

The lock contains the gateway PID. If the lock exists but the PID is dead, the stale lock is removed and the new gateway takes over.

### 4.2 Port Availability

`src/infra/ports.ts` — `ensurePortAvailable()` checks if a port is in use before binding. If the port is occupied, `describePortOwner()` tries to identify what process owns it.

`src/infra/ports-inspect.ts` — inspects port ownership via `lsof` on macOS/Linux.

---

## 5. Heartbeat Runner

`src/infra/heartbeat-runner.ts` (~1,000 LOC) implements the **proactive messaging system**: the assistant can message the user even without the user sending a message first.

### 5.1 Use Cases

- **Ghost reminder**: if the user hasn't talked to the assistant in a configurable time, the assistant sends a reminder
- **Scheduled briefings**: configured via `heartbeat.schedule` in config
- **Wake triggers**: external events can wake the heartbeat runner

### 5.2 Scheduling

The heartbeat runner uses a configurable schedule (CRON expression or interval). It checks:
1. Is it within active hours for this user?
2. Is this channel/session eligible for heartbeat?
3. Has enough time passed since the last heartbeat?

### 5.3 Visibility Control

`src/infra/heartbeat-visibility.ts` — controls whether a heartbeat fires based on the user's apparent activity (last seen time, channel activity).

---

## 6. Retry and Backoff

`src/infra/retry.ts` and `src/infra/backoff.ts` — generic retry utilities used throughout the codebase:

```typescript
// Exponential backoff retry
const result = await retry(
  () => llmApiCall(),
  {
    maxAttempts: 3,
    backoff: exponentialBackoff({ initialMs: 1000, maxMs: 30000 }),
    retryIf: (err) => isRetryableError(err),
  }
);
```

Used by: LLM API calls, channel polling, webhook delivery, plugin installs.

---

## 7. Update Runner

`src/infra/update-runner.ts` (~500 LOC) manages **self-updates**:

1. `update-check.ts` — checks npm registry for a newer version
2. `update-runner.ts` — downloads the new version (npm pack + install)
3. `update-startup.ts` — on gateway startup, schedules a background update check
4. `update-global.ts` — handles global npm install path updates
5. `update-channels.ts` — update channels: stable, beta, dev

Updates are:
- Non-breaking (new version installed to a staging location)
- Require a gateway restart to take effect
- Can be configured to auto-restart (`update.autoRestart: true`)

---

## 8. Bonjour / mDNS Discovery

`src/infra/bonjour.ts` and `src/infra/bonjour-discovery.ts` — devices on the local network can discover the OpenClaw gateway via mDNS (Bonjour/Zeroconf).

This enables companion app pairing (iOS/macOS/Android) without manually entering the gateway IP address.

---

## 9. SSH Tunnels

`src/infra/ssh-tunnel.ts` — creates an SSH tunnel to a remote host for the `openclaw nodes ssh-tunnel` command. Used to securely connect to a remote node without exposing it publicly.

---

## 10. Tailscale Integration

`src/infra/tailscale.ts` — detects and interacts with [Tailscale](https://tailscale.com/) on the host:
- Detects if Tailscale is running
- Resolves the Tailscale IP of the gateway host
- Used by `gateway.bind = "tailscale"` to bind only on the Tailscale network interface

---

## 11. Device Pairing

`src/infra/device-pairing.ts` (~500 LOC) manages the pairing flow between the gateway and client devices (iOS, Android, macOS companion apps).

Pairing:
1. Device generates a pairing token
2. Gateway validates and registers the device
3. Device gets a permanent device auth token
4. Device can now authenticate with the gateway

`src/infra/device-identity.ts` — persistent device identity storage.

---

## 12. Shell Environment

`src/infra/shell-env.ts` — captures the user's shell environment (PATH, etc.) to ensure exec tool calls run in the correct environment. This is important because the gateway may be started as a daemon without a full user shell.

---

## 13. System Events

`src/infra/system-events.ts` — internal pub/sub for gateway-wide system events (config reload complete, update available, channel connected/disconnected, etc.). Components subscribe to events without coupling directly to each other.

---

## 14. Key Files Summary

| File | Role |
|------|------|
| `src/infra/exec-approvals.ts` | Exec approval evaluation |
| `src/infra/exec-approvals-allowlist.ts` | Allowlist pattern matching |
| `src/infra/exec-safe-bin-policy.ts` | Safe-bin policy |
| `src/infra/exec-obfuscation-detect.ts` | Command obfuscation detection |
| `src/infra/exec-approval-forwarder.ts` | Routes approval to gateway |
| `src/infra/fs-safe.ts` | Atomic file operations |
| `src/infra/boundary-path.ts` | Path boundary enforcement |
| `src/infra/gateway-lock.ts` | Gateway instance lock |
| `src/infra/ports.ts` | Port availability checking |
| `src/infra/heartbeat-runner.ts` | Proactive messaging scheduler |
| `src/infra/heartbeat-wake.ts` | Heartbeat wake triggers |
| `src/infra/retry.ts` | Retry with backoff |
| `src/infra/update-runner.ts` | Self-update engine |
| `src/infra/update-startup.ts` | Background update check |
| `src/infra/bonjour.ts` | mDNS service advertisement |
| `src/infra/bonjour-discovery.ts` | mDNS service discovery |
| `src/infra/tailscale.ts` | Tailscale integration |
| `src/infra/device-pairing.ts` | Device pairing flow |
| `src/infra/shell-env.ts` | Shell environment capture |
| `src/infra/system-events.ts` | Internal pub/sub events |
| `src/infra/outbound/` | Outbound delivery pipeline |
