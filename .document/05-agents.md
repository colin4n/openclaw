# 05 — Agents

**Source files**: `src/agents/pi-embedded-runner/`, `src/agents/pi-embedded-subscribe.ts`, `src/agents/subagent-registry.ts`, `src/agents/subagent-spawn.ts`, `src/agents/pi-tools.ts`, `src/agents/system-prompt.ts`, `src/agents/skills/`, `src/agents/compaction.ts`

---

## 1. Overview

The Agents layer is the **AI session engine** of OpenClaw. It:

- Wraps LLM provider calls (Anthropic, OpenAI, Gemini, Ollama, etc.)
- Manages session lifecycle (create, run, compact, persist)
- Implements the tool pipeline (policy → invoke → result guard)
- Spawns and manages sub-agents
- Loads skills and injects them into system prompts
- Handles model auth, rotation, failover, and catalog management

The core abstraction is called **"Pi"** — referring to the `pi-agent-core` library dependency that handles the underlying LLM conversation state machine.

---

## 2. "Pi" — The Embedded Agent Runner

### 2.1 What Is Pi?

"Pi" is the primary AI session runner. The name comes from the `@mariozechner/pi-agent-core` npm dependency, which provides the core conversation loop primitives.

The OpenClaw wrapper around it is `src/agents/pi-embedded-runner/` — a large module (~52 files) that:
- Initializes the Pi session with the correct system prompt, tools, and auth
- Streams responses and handles tool calls
- Manages compaction (context window trimming)
- Persists session transcripts

### 2.2 Session Lifecycle

```
1. Session key derived from channel + account (or manual override)
2. Session transcript loaded from disk (if resuming)
3. System prompt built (identity + tools + skills + memory)
4. Pi session created with model, tools, and transcript
5. User message appended
6. LLM API call → streaming response
7. Tool calls intercepted and dispatched (see Tool Pipeline)
8. Final reply assembled
9. Transcript persisted to disk
10. On context overflow: compaction triggered (see Compaction)
```

### 2.3 Session Storage

Sessions are stored in `~/.openclaw/agents/<agentId>/sessions/`. Each session is a `.jsonl` file (JSON Lines) containing the full transcript of messages and tool calls.

Session files:
- `<sessionKey>.jsonl` — the transcript
- `<sessionKey>.lock` — write lock (prevents concurrent writes)
- A session store (`sessions.json`) maps session keys to metadata

---

## 3. System Prompt Building

`src/agents/system-prompt.ts` — `buildSystemPrompt()` assembles the full system prompt for a session. It is composed of sections:

| Section | Condition |
|---------|-----------|
| Identity line (who the assistant is) | Always |
| Runtime info (OS, date, workspace) | Full/minimal mode |
| Authorized senders (owner ID) | Full mode only |
| Skills (mandatory reading rules) | If skills are loaded |
| Memory recall instructions | If memory tool is available |
| Tooling section (available tools) | Full/minimal mode |
| Workspace info (files, dirs) | Full mode |
| Canvas instructions | If canvas is enabled |
| Sub-agent info | If spawned as sub-agent |
| Sandbox info | If running in sandbox |

**`PromptMode`** controls how much the system prompt contains:
- `"full"` — complete prompt (for the main agent)
- `"minimal"` — reduced (for sub-agents; Tooling/Workspace/Runtime only)
- `"none"` — just the identity line

---

## 4. Tool Pipeline

Every tool call from the LLM goes through a pipeline before executing:

```
LLM requests tool call
  │
  ▼
tool-policy-pipeline.ts — policy chain
  ├── allowlist check (which tools are allowed for this session/agent)
  ├── exec approval check (for shell commands)
  ├── sandbox policy (is this session sandboxed?)
  └── plugin tool policy
  │
  ▼ (if allowed)
Tool invocation (invoke tool function)
  │
  ▼
session-tool-result-guard.ts — result guard
  ├── validates result format
  ├── applies output size limits
  └── persists tool result to transcript
  │
  ▼
Result returned to LLM
```

### 4.1 Tool Categories

Tools in OpenClaw are organized by category:

| Category | Examples |
|----------|---------|
| Filesystem | `read`, `write`, `edit`, `list_directory`, `apply_patch` |
| Execution | `bash`, `system.run` |
| Agent | `sessions_spawn`, `sessions_list`, `sessions_get` |
| Channel | `message.send`, `message.list` |
| Memory | `memory_search`, `memory_get`, `memory_update` |
| Web | `browser.navigate`, `browser.screenshot` |
| Canvas | `canvas.*` tools |
| Custom | Plugin-provided tools |

Tools are defined in `src/agents/tools/` and registered per-session based on config.

### 4.2 Tool Loop Detection

`src/agents/tool-loop-detection.ts` detects when the agent is stuck in a tool call loop (calling the same tool repeatedly with the same arguments). It breaks the loop by injecting a warning into the context.

### 4.3 Exec Approval for Shell Commands

When the agent tries to run a shell command (`bash`, `system.run`) that is not on the safe-bin allowlist, the tool pipeline forwards an approval request to the gateway's `ExecApprovalManager`. The agent blocks until the human approves or denies.

---

## 5. Compaction

`src/agents/compaction.ts` handles the **context window overflow problem**: LLM context windows have a maximum token limit. When a session approaches the limit, compaction is triggered.

Compaction:
1. Detects that the context is near the token limit
2. Selects which messages to summarize/remove
3. Runs a separate LLM call to summarize the removed portion
4. Replaces the removed messages with the summary
5. Continues the session with the compacted context

This allows long-running sessions to continue indefinitely.

Key policies:
- `bootstrap-budget.ts` — manages the token budget for system prompt and bootstrap context
- `compaction.ts` — orchestrates the compaction run
- Compaction preserves tool call identifiers for OpenAI compatibility

---

## 6. Sub-Agents

Sub-agents are **spawned child agent sessions** that run in parallel or sequentially. They implement the `sessions_spawn` tool.

### 6.1 Spawning

`src/agents/subagent-spawn.ts` handles sub-agent creation:
1. Validates spawn request (depth limits, allowlist)
2. Creates a new session with a derived session key
3. Optionally inherits parent sandbox settings
4. Runs the sub-agent session
5. Returns the result to the parent agent

### 6.2 Registry

`src/agents/subagent-registry.ts` (~1,000 LOC) is the **central registry for all active sub-agents**. It:
- Tracks lifecycle (pending, running, completed, failed)
- Handles announce dispatch (sub-agent publishes results to parent)
- Manages depth limits (prevents infinite nesting)
- Handles steer (canceling/redirecting a running sub-agent)
- Persists sub-agent state across gateway restarts

### 6.3 Depth Limits

`src/agents/subagent-depth.ts` — sub-agents can spawn their own sub-agents, but there is a maximum depth. The default limit prevents runaway agent trees.

### 6.4 Announce

`src/agents/subagent-announce.ts` — when a sub-agent completes, it announces its result back to the parent. The announce system handles formatting, deduplication, and delivery ordering.

---

## 7. Skills

Skills are markdown files that instruct the agent how to perform specific tasks. They are injected into the system prompt as a "Skills" section.

### 7.1 Types of Skills

| Type | Location | Description |
|------|----------|-------------|
| Workspace skills | `<workspace>/skills/*/SKILL.md` | Project-specific skills |
| Managed skills | `~/.openclaw/agents/<id>/skills/` | Agent-specific managed skills |
| Bundled skills | `skills/` (repo root) | Default bundled skills |

### 7.2 Skill Loading

`src/agents/skills/` — key files:
- `skills.ts` — main entry, `buildWorkspaceSkillsPrompt()` assembles the skills section
- `skills-install.ts` — installs skills from npm or ClawHub
- `skills-status.ts` — reports skill status

The system prompt skills section lists available skills with descriptions. The agent reads `SKILL.md` when a skill is relevant (lazy loading — not all skill files are read upfront).

---

## 8. Model Management

### 8.1 Model Catalog

`src/agents/model-catalog.ts` — the model catalog lists all available models across all configured providers. It is loaded from config + provider discovery.

### 8.2 Model Selection

`src/agents/model-selection.ts` — resolves which model to use for a session based on:
- Explicit model config in `agents.defaults.model`
- Per-channel model overrides
- Session-level model overrides

### 8.3 Auth Profiles

`src/agents/auth-profiles/` — manages multiple API key profiles per provider. Supports:
- Round-robin rotation across multiple keys
- Cooldown on failed keys
- Priority ordering
- LastGood tracking (prefer the last known-working key)

### 8.4 Model Failover

`src/agents/model-fallback.ts` — if the primary model fails (API error, rate limit), the system falls back to a secondary model per the configured failover chain.

---

## 9. Key Files Summary

| File | Role |
|------|------|
| `src/agents/pi-embedded-runner/` | Core LLM session runner (~52 files) |
| `src/agents/pi-embedded-subscribe.ts` | Session event stream handler |
| `src/agents/pi-tools.ts` | Tool registration for Pi sessions |
| `src/agents/system-prompt.ts` | System prompt builder |
| `src/agents/compaction.ts` | Context window compaction |
| `src/agents/subagent-registry.ts` | Sub-agent lifecycle registry |
| `src/agents/subagent-spawn.ts` | Sub-agent creation |
| `src/agents/subagent-announce.ts` | Sub-agent result announcement |
| `src/agents/subagent-depth.ts` | Depth limit enforcement |
| `src/agents/skills/` | Skill loading and status |
| `src/agents/skills-install.ts` | Skill installation |
| `src/agents/model-catalog.ts` | Model catalog management |
| `src/agents/model-selection.ts` | Model resolution |
| `src/agents/model-fallback.ts` | Model failover |
| `src/agents/auth-profiles/` | API key profile rotation |
| `src/agents/tool-policy-pipeline.ts` | Tool call policy chain |
| `src/agents/tool-loop-detection.ts` | Loop detection |
| `src/agents/session-tool-result-guard.ts` | Tool result validation |
| `src/agents/session-write-lock.ts` | Session write lock |
| `src/agents/session-utils.ts` | Session utility functions |
