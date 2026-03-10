# 01 — Vision & Philosophy

**Source files**: `VISION.md`, `README.md`, `AGENTS.md`

---

## 1. What is OpenClaw?

OpenClaw is described in one sentence in the project README:

> "A personal AI assistant you run on your own devices."

It is not a SaaS platform. It is not a shared multi-tenant service. It is a **self-hosted, personal AI gateway** that:

- Runs on your own hardware (laptop, VPS, Raspberry Pi, etc.)
- Connects to messaging platforms you already use (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, and 20+ more)
- Can execute real tasks on your computer via the agent engine
- Respects your privacy by running locally

The tagline "EXFOLIATE! EXFOLIATE!" is an inside joke from the Dr. Who Daleks — reflecting the project's playful personality.

---

## 2. Core Design Philosophy

### 2.1 "AI That Actually Does Things"

OpenClaw is not just a chat interface. The agent can:
- Execute shell commands (with approval gates)
- Read and write files
- Send messages across channels
- Spawn sub-agents for parallel tasks
- Browse the web
- Interact with macOS/iOS via companion apps

This is possible because the agent is wired to a **tool pipeline** — not just a prompt/response loop.

### 2.2 Runs on Your Devices, With Your Rules

The key differentiator is operator control:

- **No dependency on external orchestration**: the gateway is self-contained.
- **No hidden capability**: every dangerous action requires explicit opt-in (exec approvals, allowlists, sandbox settings).
- **Config-driven**: behavior is controlled via `~/.openclaw/openclaw.json`, not environment variables or hidden flags.

### 2.3 Lean Core, Plugin-First

The architecture is deliberately designed so that optional capabilities live in plugins and extensions — not core. This keeps the core small, auditable, and stable.

Principle: "Core stays lean; optional capability should usually ship as plugins."

This is enforced by the module boundary:
- `src/` = core runtime
- `extensions/` = optional channel and capability plugins
- `skills/` = bundled skills (minimal; new skills go to ClawHub first)

### 2.4 Terminal-First by Design

Setup is explicit. Users see docs, auth, permissions, and security posture up front. The onboarding wizard (`openclaw onboard`) guides through each step step-by-step.

This is a deliberate choice: "We do not want convenience wrappers that hide critical security decisions from users."

Long term, easier onboarding is planned, but only as hardening matures.

### 2.5 TypeScript for Hackability

OpenClaw is written in TypeScript (ESM) because:

> "OpenClaw is primarily an orchestration system: prompts, tools, protocols, and integrations. TypeScript was chosen to keep OpenClaw hackable by default. It is widely known, fast to iterate in, and easy to read, modify, and extend."

This is a conscious tradeoff against raw performance. The system is I/O-bound (LLM calls, network, file), so TypeScript is a good fit.

---

## 3. Product Evolution

The project evolved through names:
```
Warelay → Clawdbot → Moltbot → OpenClaw
```

The `packages/clawdbot/` and `packages/moltbot/` directories in the repo are remnants of this history.

---

## 4. Priority Order (as of March 2026)

Current focus (in priority order):
1. Security and safe defaults
2. Bug fixes and stability
3. Setup reliability and first-run UX

Next priorities:
- Supporting all major model providers
- Major messaging channel support
- Performance and test infrastructure
- Better computer-use and agent harness
- Ergonomics (CLI + web frontend)
- Companion apps: macOS, iOS, Android, Windows, Linux

---

## 5. Plugin & Memory Architecture Philosophy

- **One memory plugin at a time**: memory is a "special plugin slot." Only one memory backend (e.g., LanceDB, core memory) is active at a time.
- **Skills via ClawHub**: new skills should be published to `clawhub.ai`, not added to the core bundle by default.
- **MCP via mcporter bridge**: Model Context Protocol is supported through a bridge (`mcporter`), keeping MCP integration decoupled from the core runtime.

---

## 6. What Will NOT Be Merged (Current Guardrails)

These are deliberate product guardrails, not permanent law:

| Rejected | Reason |
|----------|--------|
| New core skills (when they can live on ClawHub) | Core skill additions are rare |
| Full-doc translation sets for all docs | Deferred; AI-generated translations planned |
| Commercial integrations not fitting the model-provider category | Out of scope |
| Wrapper channels duplicating already-supported channels | Must have clear capability/security gap |
| First-class MCP runtime in core | `mcporter` already provides this path |
| Agent-hierarchy frameworks (manager-of-managers) | Not the default architecture |
| Heavy orchestration layers duplicating existing agent/tool infra | Anti-pattern |

---

## 7. Contribution Norms

- One PR = one issue/topic. No bundled unrelated fixes.
- PRs over ~5,000 changed lines reviewed only in exceptional cases.
- No large batches of tiny PRs at once; each PR has review cost.
- For small related fixes, grouping into one focused PR is encouraged.

---

## 8. Key Files for This Topic

| File | Role |
|------|------|
| `VISION.md` | Project vision statement and guardrails |
| `README.md` | Product overview, install guide, channel list |
| `AGENTS.md` | Developer/agent operational guidelines |
| `CONTRIBUTING.md` | Contribution process |
| `SECURITY.md` | Security policy and trust model |
