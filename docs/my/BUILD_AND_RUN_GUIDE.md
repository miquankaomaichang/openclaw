# OpenClaw 编译、打包与运行指南

## 目录

1. [环境准备](#环境准备)
2. [源码获取](#源码获取)
3. [依赖安装](#依赖安装)
4. [编译构建](#编译构建)
5. [运行OpenClaw](#运行OpenClaw)
6. [打包部署](#打包部署)
7. [常见问题](#常见问题)

---

## 1. 环境准备

### 1.1 系统要求

| 操作系统 | 版本要求 | 说明 |
|---------|---------|------|
| macOS | 10.15+ | 原生支持，推荐使用 |
| Linux | Ubuntu 20.04+ / Debian 11+ | 支持 |
| Windows | Windows 10+ (WSL2) | 强烈建议使用 WSL2 |

### 1.2 Node.js 环境

**必需版本**: Node 24 (推荐) 或 Node 22.19+

```bash
# 检查 Node.js 版本
node --version

# 推荐使用 nvm 管理 Node.js 版本
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 24
nvm use 24
```

### 1.3 包管理器

推荐使用 `pnpm`（项目默认）：

```bash
# 安装 pnpm
npm install -g pnpm@9.x
```

---

## 2. 源码获取

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 可选：切换到特定版本
git checkout <version-tag>
```

---

## 3. 依赖安装

```bash
# 安装项目依赖
pnpm install

# 如果遇到 Sharp/libvips 构建失败，使用以下命令
SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install
```

**安装完成后验证**:

```bash
# 检查依赖是否安装成功
pnpm list --depth=0
```

---

## 4. 编译构建

### 4.1 完整构建（推荐）

```bash
# 完整构建所有代码
pnpm build
```

### 4.2 分阶段构建（进阶）

如果需要更精细的控制，可以分步执行构建：

```bash
# 1. 编译 TypeScript 代码
node scripts/tsdown-build.mjs

# 2. 检查 CLI 引导导入
node scripts/check-cli-bootstrap-imports.mjs

# 3. 运行时后处理
node scripts/runtime-postbuild.mjs

# 4. 构建插件资源
pnpm plugins:assets:build

# 5. 复制插件资源
pnpm plugins:assets:copy

# 6. 生成类型声明（Plugin SDK）
pnpm build:plugin-sdk:dts
```

### 4.3 开发模式构建

```bash
# 开发模式（跳过渠道加载，更快启动）
OPENCLAW_SKIP_CHANNELS=1 pnpm gateway:dev
```

### 4.4 构建验证

```bash
# 检查构建产物是否完整
ls -la dist/

# 验证 CLI 是否可用
node scripts/run-node.mjs --help
```

---

## 5. 运行 OpenClaw

### 5.1 首次设置（推荐）

```bash
# 交互式引导设置
pnpm openclaw onboard

# 或安装为守护进程（推荐用于生产环境）
pnpm openclaw onboard --install-daemon
```

### 5.2 启动 Gateway（开发模式）

```bash
# 方式一：使用 watch 模式（自动重载）
pnpm gateway:watch

# 方式二：使用 dev 模式（跳过渠道）
OPENCLAW_SKIP_CHANNELS=1 pnpm gateway:dev

# 方式三：直接运行
pnpm openclaw gateway --port 18789 --verbose
```

### 5.3 启动 Gateway（生产模式）

```bash
# 启动网关服务
pnpm openclaw gateway --port 18789

# 后台运行（Linux/macOS）
nohup pnpm openclaw gateway --port 18789 > /var/log/openclaw.log 2>&1 &

# 使用 systemd（推荐）
# 创建 /etc/systemd/user/openclaw.service
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/pnpm openclaw gateway --port 18789
WorkingDirectory=/path/to/openclaw
Restart=always
RestartSec=5

[Install]
WantedBy=default.target

# 启用并启动服务
systemctl --user daemon-reload
systemctl --user enable openclaw
systemctl --user start openclaw
```

### 5.4 测试运行

```bash
# 发送测试消息
pnpm openclaw message send --target +1234567890 --message "Hello from OpenClaw"

# 与代理对话
pnpm openclaw agent --message "Hello" --thinking high

# 列出已配置的渠道
pnpm openclaw channels list

# 检查状态
pnpm openclaw gateway status
```

---

## 6. 打包部署

### 6.1 Docker 构建

```bash
# 使用项目脚本构建
pnpm build:docker

# 或者手动构建 Docker 镜像
docker build -t openclaw:latest .

# 运行 Docker 容器
docker run -d -p 18789:18789 --name openclaw openclaw:latest
```

### 6.2 macOS 应用打包

```bash
# 打包为 macOS 应用
pnpm mac:package

# 打开打包后的应用
pnpm mac:open
```

### 6.3 打包为全局 CLI

```bash
# 构建完整产物
pnpm build

# 链接到全局
pnpm link --global

# 验证全局安装
openclaw --version
```

---

## 7. 常见问题

### 7.1 构建失败

**问题**: Sharp/libvips 构建失败

**解决方案**:
```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install
```

### 7.2 端口占用

**问题**: Gateway 无法启动，提示端口被占用

**解决方案**:
```bash
# 检查端口占用
lsof -i :18789

# 杀死占用进程
kill -9 <PID>

# 使用其他端口
pnpm openclaw gateway --port 18790
```

### 7.3 依赖冲突

**问题**: 安装依赖时出现版本冲突

**解决方案**:
```bash
# 清除缓存
pnpm store prune

# 重新安装
pnpm install --shamefully-hoist
```

### 7.4 渠道连接失败

**问题**: WhatsApp/Telegram/Slack 等渠道无法连接

**解决方案**:
```bash
# 检查渠道配置
pnpm openclaw channels list

# 重新配置渠道
pnpm openclaw channels configure <channel-name>

# 运行诊断
pnpm openclaw doctor
```

### 7.5 内存不足

**问题**: 构建或运行时出现内存不足错误

**解决方案**:
```bash
# 增加 Node.js 内存限制
NODE_OPTIONS="--max-old-space-size=8192" pnpm build
```

---

## 参考命令速查

| 命令 | 用途 |
|------|------|
| `pnpm install` | 安装依赖 |
| `pnpm build` | 完整构建 |
| `pnpm gateway:watch` | 开发模式启动（自动重载） |
| `pnpm gateway:dev` | 开发模式启动（跳过渠道） |
| `pnpm openclaw gateway` | 启动网关 |
| `pnpm openclaw onboard` | 首次设置向导 |
| `pnpm openclaw doctor` | 诊断问题 |
| `pnpm test` | 运行测试 |
| `pnpm check` | 运行检查 |

---

## 配置文件位置

- **主配置**: `~/.openclaw/openclaw.json`
- **工作区**: `~/.openclaw/workspace/`
- **凭证**: `~/.openclaw/credentials/`
- **会话存储**: `~/.openclaw/sessions/`
- **日志**: `~/.openclaw/logs/`

---

## 更多资源

- [官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [Discord 社区](https://discord.gg/clawd)
- [安全指南](https://docs.openclaw.ai/gateway/security)