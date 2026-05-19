# OpenClaw Code Wiki

## 项目概述

**OpenClaw** 是一个运行在用户设备上的**个人AI助手**，支持在多种通讯渠道上进行交互。它提供了一个统一的控制平面（Gateway）来管理会话、渠道和工具。

### 核心特性

- **多渠道支持**: WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、IRC、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、Nextcloud Talk、Nostr、Synology Chat、Tlon、Twitch、Zalo、WeChat、QQ、WebChat
- **多代理路由**: 将入站渠道/账户/对等方路由到隔离的代理
- **语音唤醒**: macOS/iOS上的唤醒词 + Android上的连续语音
- **Live Canvas**: 代理驱动的可视化工作空间
- **工具系统**: 浏览器、画布、节点、定时任务、会话管理

## 项目架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OpenClaw Gateway                               │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │   CLI    │   │ Gateway  │   │  Config  │   │   Plugin System  │    │
│  │  Command │   │  Server  │   │  Manager │   │    (Registry)    │    │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────────┬─────────┘    │
│       │              │               │                   │             │
│       ▼              ▼               ▼                   ▼             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Agent Execution Engine                        │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────────┐    │  │
│  │  │  PI     │ │  Codex  │ │  Claude │ │  Subagent Runtime   │    │  │
│  │  │ Runner  │ │  Runner │ │  Runner │ │                     │    │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └─────────┬───────────┘    │  │
│  └───────┼───────────┼───────────┼───────────────┼─────────────────┘  │
│          │           │           │               │                    │
│          ▼           ▼           ▼               ▼                    │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                     Channel Plugins                         │     │
│  │  WhatsApp | Telegram | Slack | Discord | WebChat | ...     │     │
│  └──────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 模块分层

| 层级 | 模块 | 职责 |
|------|------|------|
| **入口层** | `entry.ts`, `index.ts` | CLI入口、参数解析、进程管理 |
| **控制层** | `cli/`, `gateway/` | 命令处理、WebSocket服务器、API网关 |
| **核心层** | `agents/`, `plugins/`, `channels/` | 代理执行、插件系统、渠道管理 |
| **基础设施层** | `infra/`, `config/`, `security/` | 配置管理、安全审计、工具函数 |
| **扩展层** | `extensions/`, `skills/` | 外部插件扩展、技能系统 |

## 核心模块详解

### 1. 入口模块

#### 1.1 entry.ts

主入口文件，负责：
- 命令行参数解析
- 环境变量标准化
- 编译缓存管理
- 快速路径处理（版本、帮助）

```typescript
// 关键函数
export async function runMainOrRootHelp(argv: string[])
export async function tryHandleRootHelpFastPath(argv: string[], deps?: {...})
```

#### 1.2 index.ts

Legacy入口和库导出：
- 处理旧版CLI入口
- 导出库级API供外部消费

### 2. CLI模块 (`src/cli/`)

负责命令行界面的解析和执行。

**核心文件**:
- `run-main.ts`: CLI主执行逻辑
- `program.ts`: Commander程序构建
- `argv.ts`: 参数解析
- `profile.ts`: 配置文件支持

**命令分类**:
| 命令类别 | 示例 | 说明 |
|----------|------|------|
| 代理管理 | `openclaw agent`, `openclaw agents` | 代理会话管理 |
| 渠道管理 | `openclaw channels` | 消息渠道配置 |
| 模型管理 | `openclaw models` | AI模型配置 |
| 网关管理 | `openclaw gateway` | 网关服务器控制 |
| 配置管理 | `openclaw config`, `openclaw configure` | 配置管理 |

### 3. Gateway模块 (`src/gateway/`)

WebSocket网关服务器，负责：
- 接收来自客户端/节点的连接
- 消息路由和分发
- 实时通信管理

**核心文件**:
- `server.ts`: 服务器启动和管理
- `server.impl.ts`: 服务器实现
- `auth.ts`: 认证处理
- `events.ts`: 事件处理
- `net.ts`: 网络相关工具

### 4. 代理执行引擎 (`src/agents/`)

AI代理的核心执行引擎，包含多个子模块：

#### 4.1 pi-embedded-runner

基于Pi SDK的代理执行器：
- `run/`: 执行逻辑
- `model.ts`: 模型管理
- `history.ts`: 会话历史
- `compaction.ts`: 历史压缩

#### 4.2 cli-runner

CLI模式下的代理执行：
- `execute.ts`: 执行入口
- `prepare.ts`: 准备工作
- `session-history.ts`: 会话历史管理

#### 4.3 核心工具

| 文件 | 职责 |
|------|------|
| `context.ts` | 上下文管理 |
| `failover-policy.ts` | 故障转移策略 |
| `compaction.ts` | 历史压缩 |
| `model-catalog.ts` | 模型目录 |
| `auth-profiles/` | 认证配置文件管理 |

### 5. 插件系统 (`src/plugins/`)

插件系统是OpenClaw的核心扩展机制。

#### 5.1 插件类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **Provider Plugin** | AI模型提供者 | OpenAI、Anthropic、Groq |
| **Channel Plugin** | 消息渠道 | WhatsApp、Telegram、Slack |
| **Tool Plugin** | 工具扩展 | 浏览器、画布、节点 |
| **Hook Plugin** | 钩子扩展 | 生命周期钩子 |

#### 5.2 核心文件

- `types.ts`: 插件类型定义
- `registry.ts`: 插件注册表
- `loader.ts`: 插件加载器
- `manifest.ts`: 插件清单解析
- `runtime/types.ts`: 运行时类型

#### 5.3 插件接口

```typescript
// 核心插件注册接口
export type ProviderPlugin = {
  id: string;
  label: string;
  auth: ProviderAuthMethod[];
  catalog?: ProviderPluginCatalog;
  resolveDynamicModel?: (ctx) => ProviderRuntimeModel | null;
  buildReplayPolicy?: (ctx) => ProviderReplayPolicy | null;
  // ... 更多钩子
};
```

### 6. 渠道系统 (`src/channels/`)

管理各种消息渠道的接入和消息处理。

#### 6.1 核心组件

| 组件 | 职责 |
|------|------|
| `plugins/` | 渠道插件实现 |
| `message/` | 消息处理和路由 |
| `message-access/` | 消息访问控制 |
| `inbound-event/` | 入站事件处理 |
| `turn/` | 消息轮次处理 |

#### 6.2 消息处理流程

```
入站消息 → 分类 → 访问控制 → 路由 → 代理执行 → 出站发送
```

### 7. 配置系统 (`src/config/`)

配置管理模块，负责：
- 配置文件读写
- 配置验证
- 配置变更监听
- 配置迁移

**核心文件**:
- `config.ts`: 配置API导出
- `io.ts`: 配置IO操作
- `schema.ts`: 配置Schema定义
- `types.ts`: 配置类型定义
- `mutate.ts`: 配置变更

## 关键类与函数

### 核心类型定义

#### PluginRuntime

```typescript
export type PluginRuntime = PluginRuntimeCore & {
  subagent: {
    run: (params: SubagentRunParams) => Promise<SubagentRunResult>;
    waitForRun: (params: SubagentWaitParams) => Promise<SubagentWaitResult>;
    getSessionMessages: (params) => Promise<SubagentGetSessionMessagesResult>;
    deleteSession: (params: SubagentDeleteSessionParams) => Promise<void>;
  };
  nodes: {
    list: (params?: RuntimeNodeListParams) => Promise<RuntimeNodeListResult>;
    invoke: (params: RuntimeNodeInvokeParams) => Promise<unknown>;
  };
  channel: PluginRuntimeChannel;
};
```

#### ProviderPlugin

```typescript
export type ProviderPlugin = {
  id: string;
  label: string;
  aliases?: string[];
  envVars?: string[];
  auth: ProviderAuthMethod[];
  catalog?: ProviderPluginCatalog;
  resolveDynamicModel?: (ctx) => ProviderRuntimeModel | null;
  prepareDynamicModel?: (ctx) => Promise<void>;
  normalizeResolvedModel?: (ctx) => ProviderRuntimeModel | null;
  buildReplayPolicy?: (ctx) => ProviderReplayPolicy | null;
  createStreamFn?: (ctx) => StreamFn | null;
  wrapStreamFn?: (ctx) => StreamFn | null;
  // ...
};
```

### 关键函数

#### CLI入口

```typescript
// src/cli/run-main.ts
export async function runCli(argv: string[] = process.argv)
```

启动CLI命令执行，处理参数解析、插件加载、命令路由。

#### 网关启动

```typescript
// src/gateway/server.ts
export async function startGatewayServer(...args)
```

启动WebSocket网关服务器，处理客户端连接和消息路由。

#### 配置加载

```typescript
// src/config/io.ts
export function loadConfig(opts?: LoadConfigOptions): Promise<OpenClawConfig>
```

加载并验证OpenClaw配置。

## 依赖关系

### 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `commander` | ^12.x | CLI命令解析 |
| `zod` | ^3.x | 类型验证 |
| `pi-sdk` | - | Pi代理SDK |
| `@earendil-works/pi-agent-core` | - | Pi核心库 |
| `undici` | ^6.x | HTTP客户端 |
| `execa` | ^8.x | 进程执行 |
| `fast-xml-parser` | ^4.x | XML解析 |

### 内部模块依赖

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLI Layer                                │
│  (cli/run-main.ts → cli/program.ts → cli/route.ts)              │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Gateway Layer                               │
│  (gateway/server.ts → gateway/events.ts → gateway/auth.ts)      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Plugin Layer                                │
│  (plugins/registry.ts → plugins/loader.ts → plugins/runtime/)   │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Agent Layer                                 │
│  (agents/pi-embedded-runner/ → agents/cli-runner/)             │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Channel Layer                               │
│  (channels/plugins/ → channels/message/ → channels/message-access/)│
└──────────────────────────────────────────────────────────────────┘
```

## 项目运行方式

### 开发环境

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 首次设置
pnpm openclaw setup

# 开发模式（自动重载）
pnpm gateway:watch
```

### 生产构建

```bash
# 构建核心代码
pnpm build

# 构建UI
pnpm ui:build
```

### 启动网关

```bash
# 开发模式
pnpm gateway:watch

# 生产模式
openclaw gateway --port 18789 --verbose

# 作为守护进程运行
openclaw onboard --install-daemon
```

### CLI命令示例

```bash
# 发送消息
openclaw message send --target +1234567890 --message "Hello from OpenClaw"

# 与代理对话
openclaw agent --message "Ship checklist" --thinking high

# 列出渠道
openclaw channels list

# 配置模型
openclaw models configure
```

## 安全模型

### 默认安全策略

1. **DM配对**: 默认启用`dmPolicy="pairing"`，未知发件人需要配对码验证
2. **沙盒模式**: 非`main`会话默认运行在沙盒中
3. **权限控制**: 工具访问受权限策略控制

### 沙盒配置

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main"
      }
    }
  }
}
```

### 安全检查

```bash
# 运行安全检查
openclaw doctor
```

## 目录结构总结

```
openclaw/
├── src/                    # 核心TypeScript代码
│   ├── cli/                # CLI命令处理
│   ├── gateway/            # WebSocket网关
│   ├── agents/             # AI代理执行引擎
│   ├── plugins/            # 插件系统
│   ├── channels/           # 消息渠道
│   ├── config/             # 配置管理
│   ├── infra/              # 基础设施
│   ├── plugin-sdk/         # 插件SDK
│   └── ...
├── extensions/             # 外部插件扩展
├── skills/                 # 技能系统
├── ui/                     # 控制UI
├── apps/                   # 移动端应用
├── docs/                   # 文档
└── test/                   # 测试代码
```

## 开发流程

### 代码规范

- TypeScript ESM，严格模式
- 使用`zod`进行类型验证
- 避免`any`类型，优先使用`unknown`
- 早期返回，避免嵌套条件
- 动态导入使用`*.runtime.ts`边界

### 测试策略

```bash
# 运行测试
pnpm test <path-or-filter>

# 运行变更测试
pnpm test:changed

# 运行扩展测试
pnpm test:extensions

# 运行特定文件测试
pnpm test src/plugins/registry.test.ts
```

### CI检查

```bash
# 运行所有检查
pnpm check

# 运行变更检查
pnpm check:changed

# 格式检查
pnpm format:check
```

## 部署与分发

### 安装方式

```bash
# npm
npm install -g openclaw@latest

# pnpm
pnpm add -g openclaw@latest

# 开发安装
pnpm openclaw onboard --install-daemon
```

### 配置文件位置

- 主配置: `~/.openclaw/openclaw.json`
- 工作区: `~/.openclaw/workspace/`
- 凭证: `~/.openclaw/credentials/`
- 会话存储: `~/.openclaw/sessions/`

## 扩展开发

### 插件开发流程

1. 创建插件目录
2. 编写`manifest.json5`
3. 实现`api.ts`和`runtime-api.ts`
4. 添加测试
5. 注册到插件目录

### 插件清单示例

```json5
{
  id: "my-provider",
  name: "My Provider",
  version: "1.0.0",
  description: "Custom AI provider",
  kind: "provider",
  author: "Author Name",
  license: "MIT",
  configSchema: {
    apiKey: { type: "string", secret: true }
  },
  provides: ["providers/my-provider"]
}
```

---

## 参考链接

- [官方文档](https://docs.openclaw.ai)
- [GitHub仓库](https://github.com/openclaw/openclaw)
- [插件SDK文档](https://docs.openclaw.ai/plugins/sdk-overview)
- [架构文档](https://docs.openclaw.ai/concepts/architecture)