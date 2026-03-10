# 06 — Channels

**Source files**: `src/channels/`, `src/channels/plugins/types.plugin.ts`, `src/channels/dock.ts`, `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/`, `extensions/`

---

## 1. Overview

Channels are the **messaging platform integrations** in OpenClaw. They represent the surfaces through which users talk to the AI assistant:

- WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, BlueBubbles
- Matrix, MS Teams, Feishu, LINE, Mattermost, Nextcloud Talk, IRC, Nostr
- Synology Chat, Tlon, Twitch, Zalo, Zalo Personal, WebChat

All channels implement the same **adapter interface** (`ChannelPlugin`). The gateway does not need to know about Telegram vs. Slack vs. Discord — it just knows about `ChannelPlugin` instances.

---

## 2. Architecture: The Adapter Pattern

The channel system is built around a strict adapter pattern. Every channel must implement the `ChannelPlugin` type:

```typescript
// src/channels/plugins/types.plugin.ts
export type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  // Optional adapters — implement only what the channel supports:
  config: ChannelConfigAdapter<ResolvedAccount>;    // required
  setup?: ChannelSetupAdapter;                      // first-time setup
  pairing?: ChannelPairingAdapter;                  // QR code pairing
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;                     // group chat support
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;                // send messages
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>; // polling/webhook
  auth?: ChannelAuthAdapter;
  streaming?: ChannelStreamingAdapter;              // typing indicators
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;              // receive messages
  agentPrompt?: ChannelAgentPromptAdapter;          // inject channel context
  heartbeat?: ChannelHeartbeatAdapter;              // proactive messages
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[]; // channel-specific tools
  // ... more optional adapters
};
```

A channel only implements the adapters it supports. A simple read-only channel might implement only `config`, `gateway` (for polling), and `messaging` (for receiving). A full-featured channel like Telegram implements almost all adapters.

---

## 3. Built-in Channels

These channels are part of the core source (`src/`):

| Channel | Source Dir | Notes |
|---------|-----------|-------|
| Telegram | `src/telegram/` | Polling + webhook, rich features |
| Discord | `src/discord/` | Bot API, thread bindings |
| Slack | `src/slack/` | Bolt app, events API |
| Signal | `src/signal/` | signal-cli integration |
| iMessage | `src/imessage/` | macOS only, AppleScript bridge |
| WhatsApp | `src/web/` | WhatsApp Web via Baileys |

Built-in channels are registered in the channel catalog (`src/channels/plugins/catalog.ts`) and are always available without installation.

---

## 4. Extension Channels

These channels live in `extensions/` as separate npm workspace packages. They must be installed and enabled to be used:

| Channel | Package |
|---------|---------|
| Matrix | `extensions/matrix/` |
| Microsoft Teams | `extensions/msteams/` |
| Feishu | `extensions/feishu/` |
| Google Chat | `extensions/googlechat/` |
| Mattermost | `extensions/mattermost/` |
| Nextcloud Talk | `extensions/nextcloud-talk/` |
| IRC | `extensions/irc/` |
| LINE | `extensions/line/` |
| Nostr | `extensions/nostr/` |
| Synology Chat | `extensions/synology-chat/` |
| Tlon | `extensions/tlon/` |
| Twitch | `extensions/twitch/` |
| Zalo | `extensions/zalo/` |
| Zalo Personal | `extensions/zalouser/` |
| BlueBubbles | `extensions/bluebubbles/` |
| Discord (ext) | `extensions/discord/` |
| Signal (ext) | `extensions/signal/` |
| Slack (ext) | `extensions/slack/` |
| Telegram (ext) | `extensions/telegram/` |
| iMessage (ext) | `extensions/imessage/` |
| WhatsApp (ext) | `extensions/whatsapp/` |

---

## 5. Channel Dock

`src/channels/dock.ts` — the **Channel Dock** is the central message processing hub within the gateway. Each channel's inbound messages arrive at the dock.

The dock:
1. Validates the inbound envelope (sender, account, channel)
2. Applies the allowlist/gating check (who is allowed to talk to the bot)
3. Applies rate limiting and debounce
4. Creates a session envelope (maps the message to a session key)
5. Dispatches to the agent engine
6. Handles typing indicators and ack reactions (read receipts)

The dock handles the command prefix system (`/new`, `/reset`, `/help`, etc.) before passing messages to the agent.

---

## 6. Inbound Flow

```
External message arrives (Telegram update, Slack event, WhatsApp message, etc.)
    │
    ▼
Channel gateway adapter (polling loop or webhook handler)
    │  e.g., Telegram long-poll: GET /getUpdates
    │         Slack: POST webhook from Slack Events API
    ▼
Message normalized into inbound envelope
    │  (sender ID, account ID, message text, attachments, thread ID, etc.)
    ▼
Channel dock (src/channels/dock.ts)
    │  • allowlist check (is this sender allowed?)
    │  • debounce (wait for multi-part message completion)
    │  • session key derivation (sender + channel → session key)
    │  • command parsing (/new, /reset, etc.)
    ▼
Agent dispatch (gateway/server-chat.ts)
    │
    ▼
Pi embedded runner session run
    │
    ▼
Reply generated
    │
    ▼
Outbound adapter sends reply back
```

---

## 7. Outbound Flow

```
Agent produces reply text (+ optional attachments)
    │
    ▼
src/infra/outbound/ — outbound delivery pipeline
    │  • Text chunking (split long messages into platform-safe chunks)
    │  • Attachment handling (images, documents, audio)
    │  • Channel-specific formatting (Markdown → HTML for Telegram, etc.)
    ▼
Channel outbound adapter (ChannelOutboundAdapter)
    │
    ▼
Channel API call (Telegram sendMessage, Slack chat.postMessage, etc.)
```

### Text Chunking

Messaging platforms have character limits. `src/plugin-sdk/text-chunking.ts` splits long replies into multiple messages, respecting:
- Platform-specific character limits
- Fenced code block boundaries (does not split inside a code block)
- Paragraph boundaries

---

## 8. Session Envelope

`src/channels/session-envelope.ts` — the session envelope maps an inbound message to a **session key**.

Session keys determine which conversation thread the message belongs to:
- Direct messages → `dm:<channelId>:<accountId>`
- Group chats → `group:<channelId>:<groupId>`
- Topic threads (Telegram groups) → derived from the topic

Session key scoping determines whether a group chat shares one AI session or has per-user sessions.

---

## 9. Channel Capabilities

`ChannelCapabilities` declares what features a channel supports:

```typescript
type ChannelCapabilities = {
  dm?: boolean;            // supports direct messages
  groups?: boolean;        // supports group chats
  threads?: boolean;       // supports threads
  reactions?: boolean;     // supports emoji reactions (ack/status)
  streaming?: boolean;     // supports typing indicators
  editMessage?: boolean;   // can edit sent messages (for streaming)
  media?: {
    images?: boolean;
    audio?: boolean;
    files?: boolean;
    video?: boolean;
  };
};
```

Capabilities are used by the gateway and agent system to decide which features to enable for a given channel.

---

## 10. Heartbeat

`ChannelHeartbeatAdapter` — some channels support **heartbeat** (proactive messages from the assistant to the user without a user prompt). This powers features like:
- Daily briefings
- Reminder messages
- Ghost reminders (assistant pings user if they haven't talked for a while)

The heartbeat runner (`src/infra/heartbeat-runner.ts`) schedules and triggers heartbeats per channel.

---

## 11. Status Reactions (Ack Reactions)

`src/channels/status-reactions.ts` — the assistant uses emoji reactions on messages to indicate status:
- Pending processing: typically a clock emoji
- Processing started: a different indicator
- Done: checkmark

This is only available on channels that support `reactions` capability (Slack, Discord, some others).

---

## 12. Key Files Summary

| File | Role |
|------|------|
| `src/channels/plugins/types.plugin.ts` | `ChannelPlugin` interface definition |
| `src/channels/plugins/types.adapters.ts` | All adapter type definitions |
| `src/channels/plugins/types.core.ts` | Core channel type definitions |
| `src/channels/plugins/catalog.ts` | Built-in channel catalog |
| `src/channels/dock.ts` | Channel dock — message routing hub |
| `src/channels/session-envelope.ts` | Session key derivation |
| `src/channels/session.ts` | Session management helpers |
| `src/channels/status-reactions.ts` | Emoji ack reactions |
| `src/channels/typing.ts` | Typing indicator management |
| `src/channels/run-state-machine.ts` | Per-channel run state |
| `src/infra/outbound/` | Outbound message delivery pipeline |
| `src/telegram/` | Telegram built-in channel |
| `src/discord/` | Discord built-in channel |
| `src/slack/` | Slack built-in channel |
| `src/signal/` | Signal built-in channel |
| `src/imessage/` | iMessage built-in channel |
| `src/web/` | WhatsApp web built-in channel |
| `extensions/matrix/` | Matrix extension channel |
| `extensions/msteams/` | Microsoft Teams extension channel |
