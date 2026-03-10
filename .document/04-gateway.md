# 04 — Gateway

**Source files**: `src/gateway/server.impl.ts`, `src/gateway/boot.ts`, `src/gateway/server-startup.ts`, `src/gateway/server-http.ts`, `src/gateway/server-channels.ts`, `src/gateway/config-reload.ts`, `src/gateway/control-ui.ts`

---

## 1. What is the Gateway?

The Gateway is OpenClaw's **always-on control plane** — a long-running HTTP + WebSocket server that:

- Acts as the single entry point for all inbound messages from channels
- Dispatches messages to the AI agent engine
- Manages channel lifecycle (start, stop, health monitoring)
- Runs scheduled jobs (cron)
- Serves the Control UI web dashboard
- Handles authentication for all connected clients and channels
- Manages exec approvals, node (remote host) connections
- Supports live config reload without restart

The gateway is started with:
```bash
openclaw gateway run --port 18789
```

---

## 2. Gateway Boot Sequence

The full startup sequence follows this order:

```
1. CLI parses args → creates CliDeps
2. Gateway CLI entry (src/cli/gateway-cli/) invokes server.impl.ts
3. Config load (loadConfig → validate → migrate legacy config)
4. Plugin discovery + load (loadOpenClawPlugins)
5. Gateway lock acquisition (prevents duplicate gateways on same port)
6. HTTP server bind (server-http.ts)
7. WebSocket server setup
8. Auth system initialization (rate limiter, device auth)
9. ── startGatewaySidecars() ──────────────────────────────────
   a. Stale session lock cleanup
   b. Browser control server start (if enabled)
   c. Gmail watcher start (if configured)
   d. Internal hooks load
   e. Channels start (startChannels())
   f. Internal gateway:startup hook fire
   g. Plugin services start (startPluginServices)
   h. ACP session identity reconciliation (if ACP enabled)
   i. Memory backend start
   j. Restart sentinel wake check
10. BOOT.md run (runBootOnce) — one-time agent run on fresh start
11. Heartbeat runner start
12. Update check schedule
13. Config reload watcher start
```

---

## 3. BOOT.md Mechanism

`src/gateway/boot.ts` implements a special startup feature: if `BOOT.md` exists in the workspace directory, the gateway runs an agent session on startup to process its instructions.

**Use case**: "When I start, check if there are any pending reminders and send them to me on Telegram."

How it works:
1. `loadBootFile(workspaceDir)` reads `BOOT.md`
2. A system prompt wraps the content: "You are running a boot check. Follow BOOT.md instructions exactly."
3. The agent command is invoked with `deliver: false` and `senderIsOwner: true`
4. The main session mapping is snapshot-before / restore-after so boot does not corrupt the main session
5. The agent is expected to reply with `SILENT_REPLY_TOKEN` when done (no visible output)

---

## 4. HTTP Server Structure

`src/gateway/server-http.ts` sets up the HTTP routes. Key route groups:

| Route Pattern | Purpose |
|---------------|---------|
| `/__openclaw__/control/` | Control UI API endpoints |
| `/__openclaw__/ws/` | WebSocket connection endpoint |
| `/__openclaw__/canvas/` | Canvas host (live collaborative canvas) |
| `/__openclaw__/a2ui/` | A2UI (agent-to-UI) bundle |
| `/probe` | Health probe endpoint |
| Webhook paths | Channel-specific webhook receivers |

Authentication is applied at the HTTP layer — every request passes through `evaluateGatewayAuthSurfaceStates()` before reaching handlers.

### Auth Surfaces

The gateway supports multiple auth surfaces (defined in `src/secrets/runtime-gateway-auth-surfaces.ts`):
- Default token auth (bearer token in header)
- Device auth (paired device with device-specific token)
- Control UI auth (session-based for the web dashboard)
- Trusted proxy auth

### Rate Limiting

`src/gateway/auth-rate-limit.ts` implements a fixed-window rate limiter for auth attempts. Too many failed attempts block the IP temporarily. This prevents brute-force attacks on the gateway token.

---

## 5. Channel Management

`src/gateway/server-channels.ts` — `createChannelManager()` manages the lifecycle of all registered channels.

Channel lifecycle:
```
1. Channel plugin discovered (from config + loaded extensions)
2. Channel config validated
3. Channel started (polling or webhook setup)
4. Channel health monitored (channel-health-monitor.ts)
5. Channel stopped on gateway shutdown
```

**Channel health monitor** (`src/gateway/channel-health-monitor.ts`): periodically probes channel connectivity. If a channel becomes unhealthy, it logs warnings and may attempt reconnection.

**Channel health policy** (`src/gateway/channel-health-policy.ts`): defines what "unhealthy" means per channel type and what action to take.

---

## 6. Inbound Message → Agent Dispatch

`src/gateway/server-chat.ts` — `createAgentEventHandler()` is the bridge between an inbound channel message and the agent engine.

Flow:
```
Channel adapter receives message
  → Wraps it in a session envelope
  → server-chat.ts dispatches to Pi embedded runner
  → Agent runs, produces reply
  → server-chat.ts delivers reply back via outbound adapter
```

Agent events (tool calls, streaming, sub-agent activity) are broadcast to connected WebSocket clients via `server-broadcast.ts`.

---

## 7. Cron System

`src/gateway/server-cron.ts` — `buildGatewayCronService()` runs scheduled tasks defined in config (`cron` section of `openclaw.json`).

Cron entries can:
- Run a message to the agent on schedule
- Trigger a hook
- Run a shell command (with exec approval)

The cron system uses the same session/agent pipeline as inbound messages.

---

## 8. Control UI

`src/gateway/control-ui.ts` serves the web dashboard. Key points:

- Assets are served from `dist/control-ui/` (built separately)
- CSP headers are applied (`control-ui-csp.ts`)
- Auth is required (device auth or default token)
- CORS origins are validated (`control-ui-routing.ts`)
- The dashboard communicates with the gateway via WebSocket and REST methods

`control-ui-shared.ts` contains shared constants for the control UI.

---

## 9. Method Scopes

`src/gateway/method-scopes.ts` defines the access control scope for each gateway API method:

- Methods can require specific auth levels (read, write, admin)
- Methods can be gated by role policy (`role-policy.ts`)
- The `server-methods.ts` file registers all gateway API methods and their handlers

This provides fine-grained access control over what clients can do via the gateway API.

---

## 10. Config Reload Without Restart

`src/gateway/config-reload.ts` — `startGatewayConfigReloader()` watches `openclaw.json` for changes and triggers a live config reload.

The reload process:
1. Reads and validates the new config
2. Diffs the new config against the current config
3. Determines which subsystems need to react (channels, agents, hooks, secrets, etc.)
4. Triggers `server-reload-handlers.ts` for each affected subsystem
5. Non-critical failures are logged but do not crash the gateway

The reload plan (`config-reload-plan.ts`) computes what changed and what needs to be restarted vs. what can be updated in-place.

---

## 11. Exec Approval Manager

`src/gateway/exec-approval-manager.ts` — `ExecApprovalManager` manages pending exec approval requests.

When the agent wants to run a shell command that requires approval:
1. The agent tool call is intercepted
2. An approval request is enqueued in the manager
3. The gateway waits for a human to approve or deny (via Control UI or CLI)
4. The result is returned to the agent

This is a key security boundary — dangerous commands do not execute without explicit human approval (unless the command is on the safe-bin allowlist).

---

## 12. Node Registry

`src/gateway/node-registry.ts` — `NodeRegistry` manages connected remote nodes (other machines paired to the gateway via `openclaw nodes pair`).

Nodes extend the gateway's execution capability to remote hosts. The gateway can invoke tools on a node the same way it invokes local tools.

---

## 13. Key Files Summary

| File | Role |
|------|------|
| `src/gateway/server.impl.ts` | Main gateway server (~1,000 LOC) |
| `src/gateway/server-startup.ts` | `startGatewaySidecars()` — sidecar startup |
| `src/gateway/boot.ts` | BOOT.md one-time startup agent run |
| `src/gateway/server-http.ts` | HTTP route setup |
| `src/gateway/server-channels.ts` | `createChannelManager()` |
| `src/gateway/server-chat.ts` | Inbound message → agent dispatch |
| `src/gateway/server-cron.ts` | Scheduled task runner |
| `src/gateway/control-ui.ts` | Web dashboard serving |
| `src/gateway/config-reload.ts` | Live config reload without restart |
| `src/gateway/auth.ts` | Auth logic |
| `src/gateway/auth-rate-limit.ts` | Rate limiter for auth attempts |
| `src/gateway/exec-approval-manager.ts` | Pending exec approvals |
| `src/gateway/node-registry.ts` | Remote node registry |
| `src/gateway/channel-health-monitor.ts` | Channel health monitoring |
| `src/gateway/method-scopes.ts` | API method access control |
| `src/gateway/server-methods.ts` | Gateway API method registration |
| `src/gateway/hooks.ts` | Gateway hook dispatch |
