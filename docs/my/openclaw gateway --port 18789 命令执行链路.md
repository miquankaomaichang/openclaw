我来追踪这个命令的完整执行路径。

用户问 `openclaw gateway --port 18789` 这个命令对应源码中的哪些代码。我需要追踪从命令入口到 gateway 启动的完整路径。

让我先查看主要文件：

1. 命令入口
2. gateway 命令的注册和处理
3. 启动逻辑

先从 openclaw\.mjs 开始，然后到 src/entry.ts，再到 cli/ 目录下的相关文件，最后到 gateway/server.ts。

好的，现在我需要找到 `gateway` 命令的具体实现，特别是 `--port` 参数的处理。让我看看 `gateway-cli/` 目录下的文件。

好的，现在我看到了命令参数注册的地方。接下来我需要查看 `run.js` 文件，它实际启动 gateway 的逻辑，以及 `gateway/server.ts` 等文件。

现在让我看完整的执行路径，从 `run.ts` 到实际的 gateway 启动：

现在让我看看完整的命令执行路径。我需要从入口开始追踪，一直到实际启动gateway。

好的，现在我需要找到 `runGatewayCommand` 函数，它应该在 `run.ts` 中，然后追踪到 gateway 的实际启动。让我继续看。

好的，现在继续看 `startGatewayServer` 是怎么调用的。

完美！现在我已经找到了完整的命令执行路径。让我整理一下：

1. 用户执行 `openclaw gateway --port 18789`
2. 从 `openclaw.mjs` 入口开始
3. 到 `src/entry.ts` 主入口
4. 然后到 `src/cli/run-main.ts`
5. 注册 gateway 命令在 `src/cli/gateway-cli/register.ts` 或 `run-main.ts`
6. 命令处理在 `src/cli/gateway-cli/run-command.ts` 中
7. 实际执行在 `src/cli/gateway-cli/run.ts`
8. 启动 gateway 在 `src/gateway/server.ts` -> `src/gateway/server.impl.ts`

现在让我把这些路径总结出来给用户。
完美！现在我找到了完整的命令执行路径。让我为你整理一下 `openclaw gateway --port 18789` 对应的源码链路：

## `openclaw gateway --port 18789` 命令执行链路

### 1. 入口层

- **启动入口**: [`openclaw.mjs`](file:///Users/lixinlei/Documents/openclaw/openclaw.mjs) - CLI 引导脚本，处理 Node 版本检查和编译缓存
- **主入口**: [`src/entry.ts`](file:///Users/lixinlei/Documents/openclaw/src/entry.ts) - 主进程入口，启动 CLI 运行时

### 2. 命令注册

- **CLI 主逻辑**: [`src/cli/run-main.ts`](file:///Users/lixinlei/Documents/openclaw/src/cli/run-main.ts) - 构建 Commander 命令树，支持 fast path 执行
- **Gateway 命令定义**: [`src/cli/gateway-cli/run-command.ts`](file:///Users/lixinlei/Documents/openclaw/src/cli/gateway-cli/run-command.ts)
  - 第 12 行: `--port <port>` 参数定义
  - 第 56-59 行: 命令处理 action

### 3. 参数解析与启动逻辑

- **选项解析**: [`src/cli/gateway-cli/run.ts`](file:///Users/lixinlei/Documents/openclaw/src/cli/gateway-cli/run.ts)
  - 第 302 行: `resolveGatewayRunOptions()` - 解析命令行选项
  - 第 466 行: `runGatewayCommand()` - 主执行函数
  - 第 537 行: `port = portOverride ?? resolveGatewayPort(cfg)` - 解析 `--port 18789`
  - 第 798-818 行: 调用 `startGatewayServer()`

### 4. Gateway 启动

- **服务器入口**: [`src/gateway/server.ts`](file:///Users/lixinlei/Documents/openclaw/src/gateway/server.ts)
  - 第 24 行: `startGatewayServer()` - 服务器入口函数，延迟加载实现
- **服务器实现**: [`src/gateway/server.impl.ts`](file:///Users/lixinlei/Documents/openclaw/src/gateway/server.impl.ts) - 实际的 WebSocket 服务器构建和启动

### 5. 命令执行流程图

```
openclaw gateway --port 18789
    ↓
openclaw.mjs (入口)
    ↓
src/entry.ts (主进程)
    ↓
src/cli/run-main.ts (命令注册)
    ↓
src/cli/gateway-cli/run-command.ts (option 定义)
    ↓
src/cli/gateway-cli/run.ts (参数解析和启动逻辑)
    ↓
src/gateway/server.ts (服务器入口)
    ↓
src/gateway/server.impl.ts (WebSocket 服务器实际启动)
```

### 核心关键点

| 文件               | 位置      | 功能                        |
| :--------------- | :------ | :------------------------ |
| `run-command.ts` | 第 12 行  | `--port` 参数定义             |
| `run.ts`         | 第 537 行 | 解析并设置端口                   |
| `run.ts`         | 第 808 行 | 调用 `startGatewayServer()` |
| `server.ts`      | 第 24 行  | Gateway 服务器入口函数           |

