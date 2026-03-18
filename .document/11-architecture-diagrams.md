# 11 — Architecture Diagrams

**Source files**: `src/gateway/server.impl.ts`, `src/agents/pi-embedded-runner/`, `src/channels/`, `src/plugins/`, `src/cli/`, `extensions/`

> 本文档以 Mermaid 图表形式直观呈现 OpenClaw 的整体架构、消息流、模块依赖和扩展体系。文字版说明请参阅 [02 — Overall Architecture](./02-overall-architecture.md)。

---

## 1. 整体分层架构图

```mermaid
graph TB
    subgraph USER["👤 用户侧"]
        CLI["CLI<br/>openclaw onboard / agent / gateway ..."]
        APPS["Native Apps<br/>iOS · Android · macOS"]
    end

    subgraph GATEWAY["🖥️ Gateway Layer  (src/gateway/)"]
        GW_HTTP["HTTP/WS Server<br/>server-http.ts"]
        GW_AUTH["Auth & Security<br/>auth.ts · origin-check.ts"]
        GW_CTRL["Control UI<br/>control-ui.ts"]
        GW_CHAT["Message Dispatch<br/>server-chat.ts"]
        GW_CRON["Cron Scheduler<br/>server-cron.ts"]
        GW_CFG["Config Reload<br/>config-reload.ts"]
        GW_EXEC["Exec Approval Manager<br/>exec-approval-manager.ts"]
    end

    subgraph AGENTS["🤖 Agents Layer  (src/agents/)"]
        PI["Pi Embedded Runner<br/>pi-embedded-runner/"]
        TOOLS["Tool Pipeline<br/>openclaw-tools.ts · bash-tools.ts"]
        SESSIONS["Session Manager<br/>sessions/"]
        SKILLS["Skills Loader<br/>skills/"]
        SUBAGENT["Sub-Agent Spawner<br/>acp-spawn.ts"]
        MODELS["Model Catalog & Auth<br/>model-catalog.ts · auth-profiles.ts"]
    end

    subgraph CHANNELS["📡 Channels Layer"]
        CH_CORE["Built-in Channels<br/>Telegram · Discord · Slack<br/>Signal · iMessage · WhatsApp"]
        CH_EXT["Extension Channels<br/>Matrix · MS Teams · Feishu · LINE<br/>Zalo · Mattermost · IRC · Nostr · ..."]
        CH_DOCK["Channel Dock<br/>src/channels/dock.ts"]
        CH_ROUTE["Router<br/>src/routing/resolve-route.ts"]
    end

    subgraph PLUGINS["🔌 Plugins Layer  (src/plugins/)"]
        PL_LOAD["Plugin Loader<br/>loader.ts · discovery.ts"]
        PL_HOOKS["Hook System<br/>hooks.ts"]
        PL_MEM["Memory Plugin<br/>extensions/memory-core/"]
        PL_VOICE["Voice Plugin<br/>extensions/talk-voice/"]
    end

    subgraph INFRA["⚙️ Infra Layer  (src/infra/)"]
        INF_FS["fs-safe<br/>atomic file ops"]
        INF_EXEC["Exec Approval<br/>safe-bins · allowlist"]
        INF_NET["Network Utils<br/>ports · TLS · mDNS"]
        INF_LOG["Logging<br/>src/logging/"]
    end

    subgraph LLM["☁️ LLM Providers"]
        LLM_ANT["Anthropic Claude"]
        LLM_OAI["OpenAI / Azure"]
        LLM_GEM["Google Gemini"]
        LLM_OLL["Ollama (local)"]
        LLM_MORE["Bedrock · Groq · Mistral<br/>HuggingFace · Qwen · ..."]
    end

    CLI --> GW_HTTP
    APPS --> GW_HTTP
    GW_HTTP --> GW_AUTH
    GW_AUTH --> GW_CHAT
    GW_CHAT --> CH_DOCK
    CH_DOCK --> CH_ROUTE
    CH_ROUTE --> PI
    PI --> MODELS
    PI --> TOOLS
    PI --> SESSIONS
    PI --> SKILLS
    PI --> SUBAGENT
    MODELS --> LLM_ANT
    MODELS --> LLM_OAI
    MODELS --> LLM_GEM
    MODELS --> LLM_OLL
    MODELS --> LLM_MORE
    CH_CORE --> CH_DOCK
    CH_EXT --> CH_DOCK
    GW_HTTP --> GW_CTRL
    GW_HTTP --> GW_CRON
    GW_HTTP --> GW_CFG
    GW_EXEC --> TOOLS
    PL_LOAD --> GW_HTTP
    PL_HOOKS --> PI
    PL_MEM --> PI
    PL_VOICE --> CH_DOCK
    TOOLS --> INF_EXEC
    SESSIONS --> INF_FS
    GW_HTTP --> INF_NET
    PI --> INF_LOG
```

---

## 2. 消息处理完整数据流

```mermaid
sequenceDiagram
    participant U as 用户 (Telegram/WhatsApp/Slack/...)
    participant CH as Channel Adapter
    participant DOCK as Channel Dock<br/>(dock.ts)
    participant GW as Gateway server-chat.ts
    participant PI as Pi Embedded Runner
    participant SYS as System Prompt Builder
    participant LLM as LLM API
    participant TOOL as Tool Pipeline
    participant OUT as Outbound Adapter
    participant FS as Session Store (~/.openclaw/)

    U->>CH: 发送消息
    CH->>DOCK: 接收并封装 (polling / webhook)
    DOCK->>DOCK: 验证 allowlist / pairing
    DOCK->>GW: 分发 envelope
    GW->>PI: 创建 / 恢复 session
    PI->>SYS: 构建 system prompt<br/>(identity + skills + memory + tools)
    PI->>LLM: 发起 API 调用
    LLM-->>PI: 流式返回 (text + tool_calls)
    loop 每个 tool call
        PI->>TOOL: policy check → invoke → result guard
        TOOL-->>PI: tool result
    end
    PI->>PI: 组装最终回复
    PI->>OUT: 发送回复到 channel
    OUT->>U: 用户收到回复
    PI->>FS: 持久化 session transcript
```

---

## 3. 扩展体系（Extensions）

```mermaid
graph LR
    subgraph CORE["Core (src/)"]
        SDK["plugin-sdk<br/>openclaw/plugin-sdk"]
        GATEWAY["Gateway"]
    end

    subgraph EXT["extensions/ (npm packages)"]
        direction TB
        E1["messaging channels<br/>matrix · msteams · feishu<br/>line · zalo · irc · nostr<br/>mattermost · nextcloud-talk<br/>synology-chat · tlon · twitch<br/>googlechat · bluebubbles"]
        E2["AI providers<br/>google-gemini-cli-auth<br/>minimax-portal-auth<br/>qwen-portal-auth<br/>copilot-proxy"]
        E3["capabilities<br/>memory-core · memory-lancedb<br/>talk-voice · voice-call<br/>phone-control · open-prose<br/>diffs · lobster · llm-task"]
        E4["platform<br/>imessage · signal · discord<br/>slack · telegram · whatsapp"]
    end

    SDK -->|"jiti alias at runtime"| E1
    SDK --> E2
    SDK --> E3
    SDK --> E4
    E1 -->|"ChannelPlugin interface"| GATEWAY
    E2 -->|"AuthProvider interface"| GATEWAY
    E3 -->|"Hook / Tool registration"| GATEWAY
    E4 -->|"ChannelPlugin interface"| GATEWAY
```

---

## 4. 原生 App 与 Gateway 的关系

```mermaid
graph TB
    subgraph APPS["Native Apps  (apps/)"]
        IOS["iOS App<br/>apps/ios/"]
        AND["Android App<br/>apps/android/"]
        MAC["macOS App<br/>apps/macos/<br/>(menubar, runs Gateway)"]
    end

    subgraph GW["Gateway (HTTP/WS :18789)"]
        API["REST API + WebSocket"]
        CTRL["Control UI (web dashboard)"]
        CANVAS["Canvas Host<br/>src/canvas-host/"]
    end

    subgraph NODE["Node Host  (src/node-host/)"]
        REMOTE["Remote Execution<br/>(SSH / Pi / VM)"]
    end

    MAC -->|"内嵌启动 Gateway"| GW
    IOS -->|"WebSocket 连接"| GW
    AND -->|"WebSocket 连接"| GW
    GW -->|"Node 协议"| NODE
    CANVAS -->|"a2ui bundle"| IOS
    CANVAS -->|"a2ui bundle"| AND
    CANVAS -->|"a2ui bundle"| MAC
```

---

## 5. 配置与密钥流

```mermaid
graph LR
    CFG_FILE["~/.openclaw/openclaw.json"]
    ENV["环境变量 (.env / shell)"]
    SECRETS["~/.openclaw/credentials/"]

    subgraph CONFIG["src/config/"]
        IO["io.ts — loadConfig()"]
        VAL["validation.ts — Zod schema"]
        SESS["sessions/ — session store"]
    end

    subgraph GATEWAY["Gateway"]
        MEM["内存中的 config 对象"]
        RELOAD["config-reload.ts<br/>(文件监听 → 热重载)"]
    end

    CFG_FILE --> IO
    ENV --> IO
    SECRETS --> IO
    IO --> VAL
    VAL --> MEM
    MEM --> RELOAD
    RELOAD -->|"无需重启"| MEM
```

---

## 6. 模块依赖关系（简化）

```mermaid
graph TD
    CLI["CLI (src/cli/)"]
    CMD["Commands (src/commands/)"]
    GW["Gateway (src/gateway/)"]
    AGT["Agents (src/agents/)"]
    CH["Channels (src/channels/)"]
    PL["Plugins (src/plugins/)"]
    SDK["Plugin SDK (src/plugin-sdk/)"]
    INF["Infra (src/infra/)"]
    CFG["Config (src/config/)"]
    LOG["Logging (src/logging/)"]
    SHARED["Shared (src/shared/)"]

    CLI --> CMD
    CLI --> GW
    CMD --> GW
    GW --> AGT
    GW --> CH
    GW --> PL
    AGT --> CFG
    AGT --> INF
    AGT --> LOG
    CH --> SDK
    PL --> SDK
    SDK --> SHARED
    INF --> LOG
    CFG --> SHARED
```

---

## 7. 关键文件速查

| 层次 | 文件 | 职责 |
|------|------|------|
| **CLI** | `src/entry.ts` | 进程启动、respawn guard |
| **CLI** | `src/cli/program.ts` | Commander 程序构建器 |
| **CLI** | `src/cli/deps.ts` | `createDefaultDeps()` DI 工厂 |
| **Gateway** | `src/gateway/server.impl.ts` | 主服务器实现 |
| **Gateway** | `src/gateway/server-chat.ts` | 消息 → Agent 分发 |
| **Gateway** | `src/gateway/server-channels.ts` | Channel 生命周期管理 |
| **Gateway** | `src/gateway/config-reload.ts` | 热重载配置 |
| **Agents** | `src/agents/pi-embedded-runner/pi-embedded-runner.ts` | LLM session 主循环 |
| **Agents** | `src/agents/openclaw-tools.ts` | 内置工具注册 |
| **Agents** | `src/agents/model-catalog.ts` | 模型目录与选择 |
| **Channels** | `src/channels/dock.ts` | Channel 消息入口 |
| **Channels** | `src/routing/resolve-route.ts` | 消息路由解析 |
| **Plugins** | `src/plugins/loader.ts` | 插件发现与加载 |
| **Plugins** | `src/plugins/hooks.ts` | Hook 系统 |
| **Infra** | `src/infra/` | exec 审批、fs-safe、端口、心跳 |
