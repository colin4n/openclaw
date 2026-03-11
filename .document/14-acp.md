# 14 — ACP (Agent Control Protocol)

**Source files**: `src/acp/translator.ts`, `src/acp/commands.ts`, `src/acp/runtime/types.ts`, `src/acp/types.ts`

---

## 1. What Is ACP?

**ACP (Agent Control Protocol)** is an open protocol for client-agent communication. It defines a standard interface for:
- Sending prompts to an AI agent
- Streaming text responses and tool events
- Managing session lifecycle (create, resume, steer, cancel)
- Advertising agent capabilities

OpenClaw implements ACP as a transport adapter — the gateway exposes itself as an ACP-compatible agent server, enabling ACP-aware clients (apps, IDEs, other tools) to connect without knowing OpenClaw's internal gateway API.

The implementation uses `@agentclientprotocol/sdk`.

---

## 2. Architecture: Translation Layer

`src/acp/translator.ts` — `AcpGatewayAgent` is the **bridge** between ACP protocol and OpenClaw's internal gateway:

```
ACP Client (mobile app, IDE plugin, etc.)
    │  ACP PromptRequest / streaming events
    ▼
AcpGatewayAgent (src/acp/translator.ts)
    │  Translates ACP ↔ OpenClaw EventFrame
    ▼
Gateway EventFrame bus (src/gateway/)
    │
    ▼
Pi embedded runner (src/agents/)
```

`AcpGatewayAgent` implements the ACP `Agent` interface:
- `prompt(request)` — runs a conversation turn, returns async stream of events
- `capabilities()` — advertises what the agent can do
- `sessions()` — lists active sessions

---

## 3. ACP Runtime

`src/acp/runtime/types.ts` — `AcpRuntime` interface abstracts the backend session:

```typescript
interface AcpRuntime {
  ensureSession(input: AcpRuntimeEnsureInput): Promise<AcpRuntimeHandle>;
  runTurn(input: AcpRuntimeTurnInput): AsyncIterable<AcpRuntimeEvent>;
  getCapabilities(): AcpCapabilities;
  setMode(mode: SessionMode): Promise<void>;
  setConfigOption(key: string, value: unknown): Promise<void>;
  doctor(): Promise<DiagnosticsResult>;
}
```

Key types:

```typescript
// Session handle (returned by ensureSession)
type AcpRuntimeHandle = {
  sessionKey: string;
  backend: string;
  runtimeSessionName: string;
  cwd?: string;
  acpxRecordId?: string;
  backendSessionId?: string;
};

// Prompt turn input
type AcpRuntimeTurnInput = {
  handle: AcpRuntimeHandle;
  text: string;
  mode: "prompt" | "steer";
  requestId: string;
  signal: AbortSignal;
};

// Streaming events from a turn
type AcpRuntimeEvent =
  | { type: "text_delta"; text: string }
  | { type: "status"; status: string }
  | { type: "tool_call"; name: string; input: unknown }
  | { type: "done" }
  | { type: "error"; error: Error };
```

---

## 4. Session Modes

ACP supports two session lifecycle modes:

| Mode | Description |
|------|-------------|
| `persistent` | Session persists across turns; conversation history maintained |
| `oneshot` | Each turn is stateless; no history |

Modes map to OpenClaw's session key scoping — `persistent` sessions use the standard session key, `oneshot` generates a fresh session key per prompt.

---

## 5. Available Commands

`src/acp/commands.ts` — ACP clients can send slash commands. OpenClaw exposes 30+ commands:

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/config` | Get/set configuration options |
| `/whoami` | Show agent identity |
| `/subagents` | List active sub-agents |
| `/model` | Get/set active model |
| `/think` | Set thinking mode |
| `/activation` | Show activation status |
| `/queue` | Show pending message queue |
| `/dock-*` | Channel dock management |
| `/reset` | Reset session |
| `/new` | Start a new session |
| `...` | 20+ more commands |

Commands are dispatched through the same command pipeline as channel messages (`/` prefix commands).

---

## 6. Rate Limiting

`src/acp/translator.ts` applies a **fixed-window rate limiter** on session creation:
- Default: 120 requests per 10-second window
- Prevents runaway client reconnection loops
- Separate limits per client identity

---

## 7. Session Identity and Bindings

`src/acp/runtime/session-identity.ts` — ACP sessions are mapped to OpenClaw session keys using the routing module (see `11-routing-system.md`).

`src/acp/persistent-bindings.ts` — ACP sessions can be bound to specific agents, persisted across gateway restarts.

---

## 8. Media Handling

`src/acp/translator.ts` — ACP prompts can include media attachments. The translator:
1. Parses `MEDIA:` tokens from prompt text
2. Resolves media URLs or base64 payloads
3. Passes media through the media pipeline (see `15-media-pipeline.md`)

---

## 9. Key Files Summary

| File | Role |
|------|------|
| `src/acp/translator.ts` | `AcpGatewayAgent` — ACP ↔ Gateway translation |
| `src/acp/commands.ts` | Available slash commands catalog |
| `src/acp/runtime/types.ts` | `AcpRuntime` interface + input/event types |
| `src/acp/types.ts` | ACP-specific type definitions |
| `src/acp/session.ts` | ACP session management |
| `src/acp/session-mapper.ts` | ACP session ↔ OpenClaw session key mapping |
| `src/acp/event-mapper.ts` | Gateway event → ACP event translation |
| `src/acp/persistent-bindings.ts` | Persistent agent bindings for ACP sessions |
| `src/acp/meta.ts` | ACP metadata reading helpers |
