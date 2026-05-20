# OpenClaw Project Exploration

## 1. Project Overview

**OpenClaw** is a personal, self-hosted AI assistant that runs locally on user devices and integrates with 30+ messaging channels (WhatsApp, Telegram, Discord, etc.), voice (macOS/iOS/Android), and a visual Live Canvas. It follows a gateway-first control plane architecture with multi-agent routing, sandboxed sessions, and an extensible plugin SDK.

- **Repo**: `https://github.com/openclaw/openclaw`
- **History**: Evolved through Warelay -> Clawdbot -> Moltbot -> OpenClaw
- **Tagline**: "The AI that actually does things. It runs on your devices, in your channels, with your rules."

### Core Capabilities

| Capability | Description |
|---|---|
| Multi-channel messaging | WhatsApp, Telegram, Discord, Slack, Matrix, IRC, Signal, LINE, iMessage, MS Teams, Google Chat, Feishu, Nostr, and more |
| Voice/Talk Mode | Real-time talk sessions with WebRTC, transcription, TTS relay |
| Live Canvas | A2UI (AI-to-UI) visual canvas with interactive widgets |
| Plugin SDK | Extensible runtime with code plugins and bundle-style plugins |
| MCP Support | Both server and runtime integration surface |
| Sandboxed agents | Docker/SSH/OpenShell execution sandboxing |
| Model failover | Multi-provider routing with fallback |
| Companion apps | macOS (Swift), iOS (Swift), Android (Kotlin) |

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 24 (recommended), Node.js 22.19+ minimum |
| Language | TypeScript (v6.0.3), strict ESM |
| Package Manager | pnpm (workspace monorepo), Bun (Docker binary) |
| UI Framework | Lit v3.3.3 (Web Components) |
| Build | tsdown, Vite (UI), multi-stage Docker |
| Test Framework | Vitest (colocated `*.test.ts`), Playwright v1.60.0 |
| Formatter/Linter | oxfmt (not Prettier), oxlint |
| Type Check | tsgo lanes (`pnpm tsgo*`) |
| Key Libraries | `@modelcontextprotocol/sdk` v1.29.0, `@openclaw/proxyline` v0.3.3, sharp, sqlite-vec |
| Infrastructure | Docker (Debian Bookworm slim), tini init, non-root `node` user |
| Mobile | Swift (macOS/iOS), Kotlin (Android) |

---

## 3. Repository Structure

```
openclaw/
  src/              # Core TypeScript source (4787 prod files, 3396 test files)
    agents/         # Agent runtime, provider streams, approval system (~937 files)
    channels/       # Channel abstraction layer (93 files)
    cli/            # CLI commands and argument parsing (259 files)
    commands/       # Command definitions (422 files)
    config/         # Configuration system (293 files)
    cron/           # Scheduled jobs (96 files)
    daemon/         # Background daemon (70 files)
    flows/          # Workflow orchestration (33 files)
    gateway/        # Gateway server - the control plane core (493 files)
      protocol/     # Gateway protocol (version 4), schema, versioning
    hooks/          # Lifecycle hooks system (51 files)
    infra/          # Infrastructure: env, errors, IPC (568 files)
    mcp/            # MCP server/client integration (15 files)
    media/          # Media handling (77 files)
    plugin-sdk/     # Public plugin SDK surface (492 files)
    plugins/        # Plugin loader, registry, discovery (481 files)
    routing/        # Message routing and session resolution (19 files)
    security/       # Security audits, SSRF, auth (83 files)
    sessions/       # Session lifecycle management (22 files)
    talk/           # Real-time voice/talk mode (27 files)
    tools/          # Tool system: descriptors, execution, planning (13 files)
    tui/            # Terminal UI (43 files)
    ui-adjacent/    # Shared UI types and helpers
    ...
  extensions/       # 134 plugins (3579 prod TS files)
  ui/               # Lit-based web UI (362 TS files)
  apps/             # Companion apps
    macos/          # Swift macOS app (Package.swift)
    ios/            # Swift iOS app (project.yml)
    android/        # Kotlin Android app (Gradle)
  packages/         # Shared packages
    plugin-sdk/     # Published plugin SDK package
    sdk/            # Client SDK
    memory-host-sdk/
    plugin-package-contract/
  skills/           # 58 bundled skills (1password, github, canvas, etc.)
  docs/             # Documentation site
  scripts/          # Build, CI, benchmark scripts (~338 files)
  test/             # Test helpers and fixtures
  qa/               # QA lab assets and scenarios
  config/           # Default configuration templates
  deploy/           # Deployment configs
  security/         # Security audit files
```

---

## 4. Architecture

### 4.1 Gateway Control Plane

The gateway (`src/gateway/`) is the central control plane. It handles:

- **WebSocket server**: Client connections for chat, control UI, and node events
- **HTTP endpoints**: OpenAI-compatible API, models API, embeddings, MCP HTTP
- **Session management**: Session lifecycle, transcript storage, compaction
- **Plugin orchestration**: Bootstrap, activation, capability routing
- **Auth**: Token-based, device auth, shared auth rotation
- **Channel runtime**: Channel health monitoring, status, presence
- **Cron**: Scheduled jobs and notifications
- **Talk relay**: Real-time voice session relay

Key files:
- `src/gateway/server.ts` - Main gateway server
- `src/gateway/client.ts` - Client connection handling
- `src/gateway/protocol/version.ts` - Protocol version 4
- `src/gateway/auth.ts` - Authentication system
- `src/gateway/hooks.ts` - Lifecycle hooks

### 4.2 Agent System

The agent system (`src/agents/`, ~937 files) manages:

- **Provider streams**: Anthropic, OpenAI, Google, xAI, Mistral, DeepSeek, etc.
- **Approval system**: Multi-tier approval (native, gateway, client)
- **Agent commands**: Chat, delete, scope management
- **Tool execution**: System run approval, exec boundaries
- **Model routing**: Provider selection, failover, model overrides

### 4.3 Plugin Architecture

**Boundary rules** (critical):
- Core stays plugin-agnostic; no bundled IDs/defaults/policy in core
- Plugins cross into core only via `openclaw/plugin-sdk/*`
- Plugin prod code: no `src/**`, no `src/plugin-sdk-internal/**`, no other plugin code
- Core/tests: no deep plugin internals; use public barrels and SDK facade

**Two plugin styles**:
1. **Code plugins**: Runtime extension via hooks, providers, channels, tools
2. **Bundle-style plugins**: Package stable surfaces (skills, MCP servers, config)

**Plugin lifecycle**:
1. Discovery (`src/plugins/discovery.ts`)
2. Manifest scan (`src/plugins/manifest-registry.ts`)
3. Activation planning (`src/plugins/activation-planner.ts`)
4. Runtime loading (`src/plugins/loader.ts`)
5. Capability registration (`src/plugins/capability-provider-runtime.ts`)

### 4.4 Channel System

Channels (`src/channels/`) are implementation details. Plugin authors get SDK seams. The channel layer handles:

- Inbound message processing and debouncing
- Outbound reply pipeline and streaming
- Session binding and thread management
- Allowlist/mention gating
- Presence and typing indicators
- Transport-specific adapters

### 4.5 Routing

`src/routing/` handles message routing:
- Account resolution and binding
- Channel route targets
- Session key resolution with continuity
- Peer kind matching

### 4.6 Tool System

`src/tools/` provides:
- Tool availability and boundary checks
- Descriptor schema and caching
- Execution planning and dispatch
- Protocol-level tool definitions

### 4.7 MCP Integration

`src/mcp/` provides:
- MCP server (stdio and HTTP)
- Channel-bridged MCP tools
- Plugin-hosted MCP tool handlers

---

## 5. Extensions (Plugins)

134 plugins covering:

| Category | Examples |
|---|---|
| Model providers | openai, anthropic, google, xai, mistral, deepseek, ollama, openrouter, together, fireworks, groq, cerebras, nvidia, amazon-bedrock, azure, litellm, qwen, minimax, moonshot, deepinfra, chutes, venice, vllm, sglang, lmstudio, huggingface, arcee, perplexity |
| Messaging channels | telegram, discord, whatsapp, slack, matrix, irc, signal, imessage, line, feishu, msteams, googlechat, mattermost, nostr, tlon, qqbot, zalo, xiaomi, synology-chat, nextcloud-talk, twitch |
| Voice/TTS | elevenlabs, azure-speech, deepgram, talk-voice, tts-local-cli, senseaudio |
| Image generation | fal, comfy, image-generation-core |
| Video generation | runway, video-generation-core |
| Music generation | music-generation-core |
| Search/Web | brave, duckduckgo, exa, tavily, searxng, firecrawl, web-readability |
| Memory | memory-core, memory-lancedb, memory-wiki, active-memory |
| Browser | browser |
| Coding agents | codex, opencode, opencode-go, kilocode, kimi-coding |
| Observability | diagnostics-otel, diagnostics-prometheus |
| Misc | canvas, diffs, webhooks, file-transfer, document-extract, clickclack, phone-control, skill-workshop, lobotomy, tokenjuice, admin-http-rpc |

---

## 6. Build & Docker

### Build System

- **Build command**: `pnpm build` (uses tsdown)
- **UI build**: `pnpm ui:build` (Vite)
- **Dev mode**: `pnpm dev`

### Docker

Multi-stage Dockerfile:
1. `pnpm install --frozen-lockfile`
2. Native addon verification (e.g., `matrix-sdk-crypto*.node`)
3. UI bundling
4. QA lab asset generation
5. Runtime pruning (`pnpm prune --prod`)

**Build args**:
- `OPENCLAW_EXTENSIONS` - Select extensions to include
- `OPENCLAW_INSTALL_BROWSER` - Install Playwright browser
- `OPENCLAW_INSTALL_DOCKER_CLI` - Include Docker CLI
- `OPENCLAW_IMAGE_APT_PACKAGES` / `OPENCLAW_IMAGE_PIP_PACKAGES` - Extra packages

**Runtime**:
- Base: Debian Bookworm slim
- Init: tini
- User: non-root `node`
- Security: `no-new-privileges`, `cap_drop`

**Compose**: `docker-compose.yml` defines `openclaw-gateway` and `openclaw-cli` services with volume mounts and healthchecks.

---

## 7. Key Environment Variables

| Variable | Purpose |
|---|---|
| `OPENCLAW_GATEWAY_TOKEN` | Gateway authentication token |
| `CLAUDE_AI_SESSION_KEY` | Claude AI session key |
| `CLAUDE_WEB_SESSION_KEY` | Claude web session key |
| `CLAUDE_WEB_COOKIE` | Claude web cookie |
| `TZ` | Timezone |
| `OTEL_EXPORTER_OTLP_*` | OpenTelemetry export config |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | Allow insecure private WebSocket |
| `OPENCLAW_AUTH_STORE_READONLY` | Force read-only auth store |
| `OPENCLAW_GATEWAY_STARTUP_TRACE` | Enable startup tracing |

Default gateway port: `18789`, bind mode: `lan` (0.0.0.0) or `loopback`.

---

## 8. Development Workflow

### Commands

```bash
pnpm install                # Install dependencies
pnpm build                  # Build project
pnpm dev                    # Dev mode
pnpm openclaw ...           # CLI commands
pnpm test <path>            # Run tests (Vitest)
pnpm test:changed           # Test changed files
pnpm test:serial            # Serial test execution
pnpm test:extensions        # Extension tests
pnpm check:changed          # Check changed files
pnpm check                  # Full check suite
pnpm format:*               # Format with oxfmt
pnpm lint:*                 # Lint with oxlint
pnpm tsgo*                  # Type check lanes
pnpm gateway:watch          # Dev gateway watch
```

### Testing

- **Framework**: Vitest, colocated `*.test.ts`
- **E2E**: `*.e2e.test.ts`
- **Live**: `*.live.test.ts` (run with `OPENCLAW_LIVE_TEST=1`)
- **Workers**: Max 16; memory pressure: `OPENCLAW_VITEST_MAX_WORKERS=1`
- **No concurrent Vitest runs** in one worktree (cache race risk)

### TypeScript Configuration

- Target: `es2023`, module: `NodeNext`
- `experimentalDecorators` enabled (Lit compatibility)
- `useDefineForClassFields` disabled
- Path aliases for `openclaw/plugin-sdk`, `@openclaw/*`, extension APIs

---

## 9. Codebase Statistics

| Metric | Count |
|---|---|
| Production TS files (`src/`) | 4,787 |
| Test TS files (`src/`) | 3,396 |
| Production TS files (`extensions/`) | 3,579 |
| TS files (`ui/`) | 362 |
| Extensions (plugins) | 134 |
| Skills | 58 |
| Gateway protocol version | 4 |
| Gateway files | ~493 |
| Agent files | ~937 |
| Scripts | ~338 |
| package.json lines | 1,857 |

---

## 10. Key Architectural Decisions

1. **Gateway-first**: All control flows through the gateway server; clients, nodes, and channels connect to it
2. **Plugin-agnostic core**: Core contains no bundled IDs, defaults, or policy; everything flows through manifest/registry/capability contracts
3. **Strict plugin boundaries**: Plugins only interact with core via `openclaw/plugin-sdk/*`; no direct core imports
4. **Channel isolation**: Channels are implementation under `src/channels/**`; plugin authors get SDK seams, not channel internals
5. **Hot path optimization**: Prepared facts (provider ID, model ref, channel ID, target, capability family) are carried forward, not re-discovered
6. **Lean code**: No internal shims, aliases, legacy names, broad fallbacks, or defensive branches for unrealistic edge cases
7. **Security by default**: Secure defaults with explicit knobs for trusted high-power workflows
8. **Terminal-first setup**: Setup is explicit so users see auth, permissions, and security posture up front
9. **TypeScript for hackability**: Chosen for wide familiarity, fast iteration, and easy modification
10. **Bundle-style preferred**: When plugins can express capability as bundles instead of code, prefer bundles (smaller interface, better security boundaries)

---

## 11. Companion Apps

| Platform | Language | Build System |
|---|---|---|
| macOS | Swift | Package.swift |
| iOS | Swift | project.yml (XcodeGen) |
| Android | Kotlin | Gradle (Kotlin DSL) |

There is also a `macos-mlx-tts` variant for MLX-based text-to-speech on Apple Silicon.
