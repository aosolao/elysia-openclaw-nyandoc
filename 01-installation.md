# 第 1 章 · 安装与初始化

> 版本：v1.0.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [1.1 OpenClaw 是什么（小白先看）](#11-openclaw-是什么小白先看)
- [1.2 前置要求](#12-前置要求)
  - [各系统安装 Node.js](#各系统安装-nodejs)
- [1.3 安装 OpenClaw](#13-安装-openclaw)
- [1.4 首次启动与向导](#14-首次启动与向导)
  - [平台服务差异](#平台服务差异)
- [1.5 关键目录结构](#15-关键目录结构)
- [1.6 必做的第一步：自检](#16-必做的第一步自检)
- [1.7 备份建议（重要）](#17-备份建议重要)
- [1.8 下一步](#18-下一步)

<!-- TOC END -->

## 1.1 OpenClaw 是什么（小白先看）

OpenClaw 是一个"AI 助手运行平台"：它在你的电脑上跑一个常驻服务（Gateway），连接大模型（豆包、百炼、OpenAI 等），并通过聊天渠道（Telegram、网页、Discord 等）和你对话，还能调用工具（读写文件、跑命令、搜索网页、记忆等）。

你可以把它理解成：**一个住在你电脑里的 AI 助手的"身体"，而大模型是它的"大脑"**。

## 1.2 前置要求

| 组件 | 版本/要求 | 说明 |
|---|---|---|
| Node.js | 建议 22+（本地 embeddings 插件建议 24） | LTS 版本即可 |
| 一个大模型 API key | 豆包/百炼/OpenAI/本地模型等 | 至少有一个能对话的模型 |
| 磁盘空间 | 约 500MB（核心）+ 数据 | 记忆、日志、模型会逐渐增长 |
| 终端 | 任意 | 用于执行配置命令 |

### 各系统安装 Node.js

- **macOS**：`brew install node@22`（需要先装 [Homebrew](https://brew.sh)）
- **Linux (Debian/Ubuntu)**：用 [NodeSource](https://github.com/nodesource/distributions) 或 nvm：
  ```bash
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
  nvm install 22
  ```
- **Windows**：从 [nodejs.org](https://nodejs.org) 下载 LTS 安装包，或用 `winget install OpenJS.NodeJS.LTS`。推荐在 PowerShell 7 中操作。

## 1.3 安装 OpenClaw

OpenClaw 通过 npm 全局安装：

```bash
npm install -g openclaw
```

验证：

```bash
openclaw --version
```

> **中国大陆用户注意**：如果 npm 慢，可换淘宝镜像：
> ```bash
> npm config set registry https://registry.npmmirror.com
> npm install -g openclaw
> ```
> 但注意：部分原生插件（如 llama.cpp）从镜像安装可能缺预编译包，遇到问题切回官方源。

## 1.4 首次启动与向导

```bash
openclaw
```

首次运行会进入交互式向导，引导你：
1. 选择运行模式（本地 personal / 远程等）——个人使用选 **local**。
2. 配置第一个模型提供商和 API key。
3. 创建 Gateway 服务（macOS/Linux 上是 LaunchAgent/systemd 用户服务；Windows 上是计划任务或后台进程）。
4. 设置访问 token。

向导跑完后，Gateway 会在后台运行，默认监听 `http://127.0.0.1:18789`。

### 平台服务差异

| 系统 | Gateway 托管方式 | 配置/日志位置 |
|---|---|---|
| **macOS** | LaunchAgent（`~/Library/LaunchAgents/ai.openclaw.gateway.plist`） | `~/.openclaw/` |
| **Linux** | systemd user service（`~/.config/systemd/user/openclaw-gateway.service`） | `~/.openclaw/` |
| **Windows** | 后台进程 / 计划任务（取决于安装方式） | `%USERPROFILE%\.openclaw\` |

常用服务命令：

```bash
openclaw gateway status     # 查看状态
openclaw gateway restart    # 重启
openclaw gateway stop       # 停止
openclaw gateway start      # 启动
```

## 1.5 关键目录结构

```
~/.openclaw/
├── openclaw.json              # 主配置文件（核心）
├── exec-approvals.json        # 命令审批策略与白名单
├── workspace/                 # 默认工作区（记忆、人格、笔记）
│   ├── AGENTS.md              # 给 AI 的行为规则
│   ├── SOUL.md                # AI 人格设定
│   ├── USER.md                # 关于你的信息
│   ├── MEMORY.md              # 长期记忆
│   └── memory/                # 每日笔记
├── agents/main/agent/         # agent 数据（SQLite 索引等）
├── memory/lancedb/            # LanceDB 向量记忆库（装了插件后）
├── secrets/                   # 密钥读取脚本（如有）
└── npm/projects/              # 通过插件安装的外部 npm 包
```

> Windows 上把 `~` 换成 `C:\Users\<你的用户名>`。

## 1.6 必做的第一步：自检

装完后先跑：

```bash
openclaw status          # 整体状态
openclaw doctor          # 诊断问题
openclaw security audit  # 安全检查
```

健康的状态应该看到：
- Gateway `running` / `reachable`
- 至少一个模型可用
- 没有 critical 级别的安全问题

## 1.7 备份建议（重要）

定期备份 `~/.openclaw/openclaw.json` 和整个 `~/.openclaw/workspace/`。
如果你用了 LanceDB，也备份 `~/.openclaw/memory/`。

macOS 可用 Time Machine；Linux/Windows 建议把 `~/.openclaw` 纳入你的备份方案。
**注意：配置里可能有 token/API key，备份要加密存储。**

## 1.8 下一步

- 配置模型 → [第 2 章 模型配置](./02-models.md)
- 上锁安全 → [第 3 章 安全与执行审批](./03-security-and-exec.md)
