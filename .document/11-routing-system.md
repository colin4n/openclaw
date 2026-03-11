# 11 — Routing System

**Source files**: `src/routing/resolve-route.ts`, `src/routing/session-key.ts`, `src/routing/bindings.ts`, `src/routing/account-lookup.ts`

---

## 1. Overview

The routing module resolves **which agent handles an incoming message** based on channel, account, peer, guild/team, and role constraints. It is the bridge between channel adapter layer and the agent engine.

Every inbound message goes through route resolution before being dispatched to the session engine. The result determines the `agentId` and `sessionKey` for the conversation.

---

## 2. Route Resolution

`src/routing/resolve-route.ts` — `resolveAgentRoute()` implements a **7-tier priority matching system**:

```
Priority (high → low):
  1. Peer bindings         (exact sender match)
  2. Parent peer inherit   (parent session's peer binding)
  3. Guild + roles         (server + role combo)
  4. Guild                 (server-level binding)
  5. Team                  (workspace-level binding)
  6. Account               (channel account binding)
  7. Default fallback      (resolveDefaultAgentId)
```

Inputs to route resolution:

```typescript
type ResolveAgentRouteInput = {
  channel: ChannelId;
  accountId: string;
  peer: PeerId;          // sender identifier
  guildId?: string;      // Discord server / Slack workspace
  teamId?: string;
  memberRoleIds?: string[];
  parentPeer?: PeerId;   // for inherited bindings from parent session
};
```

Output:

```typescript
type ResolvedAgentRoute = {
  agentId: string;
  sessionKey: string;
  mainSessionKey: string;
  matchedBy: RouteMatchReason;  // which tier matched
};
```

**Caching**: route resolution is memoized using a `WeakMap` with a 4,000-entry LRU-style limit to avoid repeated config lookups on busy channels.

---

## 3. Session Key Construction

`src/routing/session-key.ts` — session keys identify a specific conversation thread. The format:

```
agent:<agentId>:<channel>:<peerType>:<peerId>
```

**Session key variants** (DM scope modes):

| Mode | Pattern | Description |
|------|---------|-------------|
| `main` | `agent:X:ch:dm:peer` | Default DM session |
| `per-peer` | `agent:X:ch:dm:peer` | One session per peer (same as main) |
| `per-channel-peer` | `agent:X:ch:ch-dm:peer` | Scoped to channel+peer |
| `per-account-channel-peer` | `agent:X:ch:acct-ch-dm:peer` | Fully scoped |

**Group chat session keys** are derived from the group/thread ID rather than the sender peer.

**Identity linking** — the session key system supports cross-channel user mapping: a verified user on Telegram can be linked to the same user on Slack, sharing a session.

---

## 4. Bindings

`src/routing/bindings.ts` — route bindings are configured in `openclaw.json` under `agents[].bindings[]`.

A binding entry:

```json
{
  "channel": "telegram",
  "accountId": "mybot",
  "peer": "@username",
  "agentId": "alice"
}
```

Binding types:
- **Peer binding**: exact sender → agent
- **Account binding**: all messages from a channel account → agent
- **Guild binding**: all messages from a Discord server/Slack workspace → agent
- **Guild+roles binding**: guild members with specific roles → agent

`bindings.ts` also provides:
- `listRouteBindings()` — enumerate all bindings from config
- `extractBoundAccountIdsByChannel()` — which account IDs are bound per channel
- `buildChannelToAgentMap()` — channel → agentId mapping for quick lookup

---

## 5. Account Lookup

`src/routing/account-lookup.ts` — `resolveAccountEntry()` performs **case-insensitive account ID lookup** in config. This handles platform account ID normalization (e.g., some platforms normalize casing, some don't).

---

## 6. Connection to Other Modules

```
Channel dock (src/channels/dock.ts)
  ↓ inbound message + sender info
Routing module (resolve-route.ts)
  ↓ agentId + sessionKey
Agent engine (src/agents/pi-embedded-runner/)
  ↓ session key
Memory system (src/memory/) — session-scoped memory
ACP module (src/acp/) — session identity
```

---

## 7. Key Files Summary

| File | Role |
|------|------|
| `src/routing/resolve-route.ts` | Core 7-tier route resolution with caching |
| `src/routing/session-key.ts` | Session key format construction and parsing |
| `src/routing/bindings.ts` | Binding enumeration and channel-to-agent map |
| `src/routing/account-lookup.ts` | Case-insensitive account ID resolution |
| `src/routing/account-id.ts` | Account ID normalization |
| `src/routing/default-account-warnings.ts` | Default account validation warnings |
