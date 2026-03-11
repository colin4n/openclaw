# OpenClaw CodeWiki

This knowledge base documents the architecture, design philosophy, and key module internals of the [OpenClaw](https://github.com/openclaw/openclaw) project. It is intended as a learning and reference guide for developers exploring the codebase.

> **For AI assistants**: When the user is learning, exploring, or asking architecture questions, write findings here — do NOT write code. See the **[AI Learning Mode](#ai-learning-mode--rules-for-all-tools)** section below.

---

## How to Read This Wiki

Start with the Vision and Overall Architecture documents to get the big picture. Then dive into specific modules as needed. Each document includes references to the actual source files so you can follow up with code reading.

---

## AI Learning Mode — Rules for All Tools

This section applies to **any AI assistant** (Qoder, Claude, Codex, Cursor, etc.) working in this repo.

### When to write here vs. write code

| User intent | Action |
|-------------|--------|
| "How does X work?" | Write to `.document/` |
| "Explain the Y system" | Write to `.document/` |
| "What is the architecture of ..." | Write to `.document/` |
| "Why is it designed this way?" | Write to `.document/` |
| "What files handle X?" | Write to `.document/` |
| "Fix this bug" | Write code as normal |
| "Add feature Z" | Write code as normal |
| "Run the tests" | Execute command |

If intent is **ambiguous**, ask:
> "Should I (A) document this in `.document/` for the knowledge base, or (B) write/run code?"

### Steps when writing knowledge

1. **Read** the relevant source files to research the answer — never invent content.
2. **Find the right document** — check the Table of Contents below. Add a section to an existing doc if the topic fits.
3. **Create a new document** if the topic is substantial and not yet covered (see naming rules below).
4. **Update this README** — add the new file to the Table of Contents below.
5. **Tell the user** what was written and where.

### Document naming convention

`<two-digit-number>-<lowercase-kebab-topic>.md` — Example: `11-routing-system.md`

Next available number: **16**

### New document structure

```
# <NN> — Topic Title

**Source files**: `src/path/to/file.ts`, `src/other/file.ts`

---

## 1. First Concept
...

## 2. Second Concept
...

## N. Key Files Summary

| File | Role |
|------|------|
| `src/...` | Description |
```

### Adding to an existing document

Append a new numbered section at the end of the relevant file:

```markdown
## N. New Section Title

...content...
```

---

## Table of Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Vision & Philosophy](./01-vision-and-philosophy.md) | Project goals, design principles, TypeScript rationale, guardrails |
| 02 | [Overall Architecture](./02-overall-architecture.md) | System layers, module map, runtime boundaries, data flow |
| 03 | [Entry & CLI](./03-entry-and-cli.md) | CLI entry point, respawn, profile, program structure, DI pattern |
| 04 | [Gateway](./04-gateway.md) | Long-running HTTP/WS server, boot sequence, config reload, control UI |
| 05 | [Agents](./05-agents.md) | Pi runner, session lifecycle, subagents, tools, skills, compaction |
| 06 | [Channels](./06-channels.md) | Channel adapter pattern, built-in/extension channels, inbound/outbound flow |
| 07 | [Plugin SDK](./07-plugin-sdk.md) | Plugin types, SDK surface, lifecycle hooks, extension model |
| 08 | [Infrastructure (infra)](./08-infra.md) | Exec approvals, fs-safe, ports, heartbeat, retry, update runner |
| 09 | [Config & Secrets](./09-config-and-secrets.md) | Config schema, session store, secrets runtime, auth surfaces |
| 10 | [Security Model](./10-security-model.md) | Trust model, exec approval flow, SSRF policy, auth boundaries |
| 11 | [Routing System](./11-routing-system.md) | 7-tier agent routing, session key construction, bindings |
| 12 | [Memory System](./12-memory-system.md) | Vector + FTS search, embedding providers, session memory hooks |
| 13 | [Hooks System](./13-hooks-system.md) | Internal hooks, bundled hooks, user-configured hooks |
| 14 | [ACP — Agent Control Protocol](./14-acp.md) | ACP adapter, session modes, streaming events, commands |
| 15 | [Media Pipeline](./15-media-pipeline.md) | Media I/O (FFmpeg, Sharp), AI understanding (transcription, vision) |

---

## Repository Layout (Quick Reference)

```
openclaw/
├── src/                   # Core source code (TypeScript/ESM)
│   ├── cli/               # CLI entry, program builder, commands wiring
│   ├── commands/          # Command implementations (agent, gateway, message, etc.)
│   ├── gateway/           # Gateway server (HTTP/WS, auth, sessions, cron)
│   ├── agents/            # AI agent engine (Pi runner, tools, subagents, skills)
│   ├── channels/          # Channel abstraction + plugin dock system
│   ├── plugins/           # Plugin loader, registry, hooks, install
│   ├── plugin-sdk/        # Public SDK surface exported to extensions
│   ├── infra/             # Infrastructure utilities (exec, fs, ports, heartbeat)
│   ├── config/            # Config schema, I/O, sessions, secrets
│   ├── secrets/           # Secrets runtime snapshot
│   ├── telegram/          # Telegram built-in channel
│   ├── discord/           # Discord built-in channel
│   ├── slack/             # Slack built-in channel
│   ├── signal/            # Signal built-in channel
│   ├── imessage/          # iMessage built-in channel
│   └── web/               # WhatsApp web built-in channel
├── extensions/            # Optional channel/plugin extensions (npm packages)
├── apps/                  # Native companion apps (iOS, Android, macOS)
├── skills/                # Bundled default skills
├── docs/                  # Mintlify documentation source
└── test/                  # Integration and E2E tests
```

---

## Key Design Principles (at a Glance)

- **Lean core, plugin-first**: optional capability lives in `extensions/`, not core.
- **Terminal-first**: setup is explicit and visible to the user.
- **One trusted operator per gateway**: not a multi-tenant system.
- **TypeScript for hackability**: widely known, easy to read and extend.
- **Security by explicit knobs**: strong defaults, controlled escape hatches.

---

## Adding New Topics

Next available document number: **16**

Topics not yet covered that would be valuable additions:
- Auto-reply system (`src/auto-reply/`)
- Cron system deep-dive (`src/cron/`)
- TUI (Terminal UI) (`src/tui/`)
- Browser/computer-use tools (`src/browser/`)
- Canvas system (`src/canvas-host/`)
- Node host (remote execution) (`src/node-host/`)

---

*Last updated: March 2026. Based on OpenClaw v2026.3.3.*
