# 03 — Entry & CLI

**Source files**: `src/entry.ts`, `src/index.ts`, `src/cli/`, `src/commands/`

---

## 1. Two Entry Points

OpenClaw has two entry files that serve different purposes:

### `src/entry.ts` — The Modern Entry Point

This is the **primary process entry point**, invoked from `openclaw.mjs` (the npm binary wrapper). It is responsible for:

1. Process title (`process.title = "openclaw"`)
2. Node compile cache (`enableCompileCache()`) for startup performance
3. Read-only auth store guard (for `secrets audit` command)
4. `--no-color` env normalization
5. Respawn guard (experimental warning suppression — see below)
6. Profile argument parsing
7. Fast-path dispatching: `--version` and `--help` without loading the full program
8. Lazy loading the actual CLI runner via `import("./cli/run-main.js")`

Key design: everything is **lazy-loaded via dynamic import** to keep startup fast for common paths like `--version` or `--help`.

### `src/index.ts` — The Legacy/Public API Entry

This file serves a dual purpose:
1. **Legacy CLI entry**: can run as `node dist/index.js` and invoke `program.parseAsync()`
2. **Public API surface**: exports utilities for direct programmatic use (`loadConfig`, `runExec`, etc.)

The `isMainModule()` guard prevents double-initialization when `index.ts` is imported as a dependency (e.g., by the bundler).

---

## 2. Respawn Mechanism

The respawn mechanism in `src/entry.ts` is a workaround for a Node.js behavior: `--disable-warning=ExperimentalWarning` cannot be set via `NODE_OPTIONS`, so the process spawns a child with the flag added to `execArgv`.

```
Process A (openclaw.mjs)
  └─ entry.ts runs
       └─ hasExperimentalWarningSuppressed() → false
       └─ Spawns Process B with --disable-warning=ExperimentalWarning
             └─ Process B runs entry.ts again
                  └─ OPENCLAW_NODE_OPTIONS_READY=1 → skip respawn
                  └─ Runs actual CLI
  └─ Process A bridges stdin/stdout to B and waits for exit
```

The `attachChildProcessBridge` in `src/process/child-process-bridge.ts` handles signal forwarding and stdin bridging between the wrapper process and the respawned child.

Respawn is skipped when:
- `OPENCLAW_NO_RESPAWN=1`
- `OPENCLAW_NODE_OPTIONS_READY=1` (already respawned)
- The warning is already suppressed via `NODE_OPTIONS` or `execArgv`
- The subcommand does not need it (`shouldSkipRespawnForArgv` checks for daemon/respawn-exempt commands)

---

## 3. Profile Selection

The `--profile <name>` flag (parsed in `src/cli/profile.ts`) allows running OpenClaw with a different environment profile:

```bash
openclaw --profile work gateway run
```

Profiles map to environment variable files:
- `~/.openclaw/profiles/<name>.env` or similar
- `applyCliProfileEnv({ profile })` in `src/cli/profile.ts` injects the env vars

This is useful for managing multiple API keys, gateways, or configurations on the same machine.

After profile env is applied, `process.argv` is normalized to remove the `--profile` flag so Commander does not see it.

---

## 4. CLI Program Structure

The CLI is built with [Commander.js](https://github.com/tj/commander.js) and organized in `src/cli/program/`:

```
src/cli/program.ts         → re-exports buildProgram()
src/cli/program/
  ├── index.ts             → buildProgram() — assembles all commands
  ├── root-options.ts      → global options (--verbose, --log-level, --profile, etc.)
  └── ...                  → per-command sub-programs
```

`buildProgram()` assembles the Commander program by importing and attaching all sub-commands. This is done lazily at the module level — when the file is first imported.

### Key Commands

| Command | Source | Description |
|---------|--------|-------------|
| `openclaw agent` | `src/commands/agent.ts` | Run agent with a message |
| `openclaw gateway run` | `src/cli/gateway-cli/` | Start the gateway server |
| `openclaw message send` | `src/commands/` | Send a message via a channel |
| `openclaw onboard` | `src/wizard/` | Run the onboarding wizard |
| `openclaw channels status` | `src/cli/channels-cli.ts` | Show channel status |
| `openclaw models` | `src/cli/models-cli.ts` | Manage model config |
| `openclaw plugins` | `src/cli/plugins-cli.ts` | Manage plugins |
| `openclaw config` | `src/cli/config-cli.ts` | Read/write config keys |
| `openclaw secrets` | `src/cli/secrets-cli.ts` | Manage secrets |
| `openclaw nodes` | `src/cli/nodes-cli/` | Remote node management |
| `openclaw browser` | `src/cli/browser-cli.ts` | Browser control |
| `openclaw skills` | `src/cli/skills-cli.ts` | Skill management |

---

## 5. Dependency Injection: `createDefaultDeps()`

`src/cli/deps.ts` defines the **`CliDeps`** type and its factory:

```typescript
export type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
};

export function createDefaultDeps(): CliDeps { ... }
```

Each send function in `createDefaultDeps()` is a lazy wrapper that dynamically imports a `*.runtime.ts` file on first call. This means the WhatsApp library is not loaded unless you actually send a WhatsApp message.

```
sendMessageWhatsApp() call
  → loadWhatsAppSenderRuntime()
  → await import("./deps-send-whatsapp.runtime.js")   // lazy, once
  → actual implementation
```

**Why this pattern?**
- Keeps startup time fast (avoids loading all channel libraries on boot)
- Prevents `[INEFFECTIVE_DYNAMIC_IMPORT]` bundler warnings (rule: do not mix static and dynamic imports for the same module)
- Makes testing easy — any command that takes `CliDeps` can be tested with mock send functions

The `CliDeps` object flows from CLI entry → gateway → agent commands as needed.

---

## 6. Windows argv Normalization

`src/cli/windows-argv.ts` handles a Windows-specific quirk: when running OpenClaw via `npx openclaw` or PowerShell on Windows, `process.argv` may contain unexpected Unicode or mangled arguments. The normalizer sanitizes these before passing to Commander.

This is transparent to non-Windows users.

---

## 7. Fast-Path Dispatching

For common operations, `src/entry.ts` short-circuits Commander entirely:

```typescript
// Fast path for --version: prints version without loading full program
if (!tryHandleRootVersionFastPath(process.argv)) {
  // ...
}

// Fast path for --help: shows help without loading all commands
if (!tryHandleRootHelpFastPath(process.argv)) {
  // ...
}

// Otherwise: load full CLI runner
import("./cli/run-main.js").then(({ runCli }) => runCli(process.argv))
```

The version fast path imports only `src/version.ts` (tiny). The help fast path imports `src/cli/program.ts`. Both avoid loading the gateway, agents, or channel libraries.

---

## 8. Environment Normalization

`src/infra/env.ts` — `normalizeEnv()` is called early in both `entry.ts` and `index.ts`. It:
- Normalizes boolean env vars (e.g., `OPENCLAW_DEBUG=true` → consistent truthiness)
- Applies any required env defaults before the rest of the program reads them

`src/infra/dotenv.ts` — `loadDotEnv()` loads `.env` from the workspace directory (called in `index.ts` with `quiet: true`). This allows local `.env` files for development.

---

## 9. Key Files Summary

| File | Role |
|------|------|
| `src/entry.ts` | Process bootstrap, respawn, profile, fast-path dispatch |
| `src/index.ts` | Legacy entry, public API exports |
| `src/cli/program.ts` | Commander program builder entry |
| `src/cli/program/` | All sub-command registrations |
| `src/cli/run-main.ts` | Actual CLI runner invoked after respawn |
| `src/cli/deps.ts` | `createDefaultDeps()` DI factory with lazy send functions |
| `src/cli/profile.ts` | `--profile` flag support |
| `src/cli/argv.ts` | Argv parsing utilities (version/help detection) |
| `src/cli/respawn-policy.ts` | Which commands skip respawn |
| `src/cli/windows-argv.ts` | Windows argv normalization |
| `src/infra/env.ts` | `normalizeEnv()` |
| `src/infra/dotenv.ts` | `loadDotEnv()` |
| `src/process/child-process-bridge.ts` | Signal/stdin bridging for respawn |
