# elysia-openclaw-nyandoc

> 📖 面向初入 [OpenClaw](https://github.com/openclaw/openclaw) 的用户的完整配置指南。

---

## 关于本指南

本仓库是一份 **OpenClaw 完整配置指南**，覆盖从安装初始化到沙箱隔离、向量记忆、多渠道消息等高级功能。

### 🤝 人类与 Agent 共创

本指南由**人类与 AI Agent 共同创作**，基于 **Qwen3.8-Max** 大语言模型驱动。面向**初入 OpenClaw、感到迷茫且需要帮助的用户**。

### 🍎 平台说明

本指南基于 **Apple macOS（Apple Silicon）** 环境验证。其他操作系统（Linux、Windows）同样支持，文档中在适用处标注了平台差异和配置思路。

## 文档目录

| # | 章节 | 内容 |
|---|------|------|
| 0 | [总览](./index.md) | 配置快照、阅读建议、通用约定 |
| 1 | [安装与初始化](./01-installation.md) | 安装、首次启动、目录结构 |
| 2 | [模型配置](./02-models.md) | 提供商、API Key、超时、思考过程 |
| 3 | [安全与执行审批](./03-security-and-exec.md) | Gateway 绑定、命令审批、工具策略 |
| 4 | [记忆系统](./04-memory.md) | LanceDB、Embedding、自动召回、文件记忆 |
| 5 | [渠道配置](./05-channels.md) | Telegram、Discord、网页、表情回应、贴纸 |
| 6 | [本地服务与插件](./06-local-services.md) | Embedding 服务、Keychain、代理、搜索、浏览器 |
| 7 | [工作区与项目目录](./07-workspace.md) | 目录布局、Git 管理、关键文件 |
| 8 | [日常维护与排错](./08-maintenance.md) | 日志、常见问题、备份、安全清单 |
| 9 | [沙箱与隔离环境](./09-sandbox.md) | Docker 沙箱、自定义镜像、Elevated 逃逸 |
| 10 | [常用聊天命令速查](./10-slash-commands.md) | 会话管理、模型切换、思考控制、状态诊断 |

## 前置要求

- **Node.js** 22+（推荐 LTS 版本）
- 至少一个大模型 API Key（如 OpenAI、豆包、百炼、Ollama）
- 约 500 MB 磁盘空间（随记忆和日志增长）

## 快速开始

```bash
# 安装
npm install -g openclaw

# 启动交互式配置向导
openclaw

# 验证
openclaw status
openclaw doctor
```

详细说明见[第 1 章](./01-installation.md)。

## 沙箱镜像

本仓库提供预构建的 OpenClaw 沙箱镜像，支持 **amd64** 和 **arm64** 架构：

```bash
docker pull aosolao/openclaw-sandbox:py313-node26-trixie
```

镜像基于 Debian Trixie，预装 Python 3.13、Node 26、Playwright、金融数据分析工具链等。详细说明见[第 9 章](./09-sandbox.md)。

如需自行构建，Dockerfile 位于 `docker-sandbox/` 目录。

## 参与贡献

欢迎提交 Issue 和 Pull Request！

请遵循以下风格：
- 每节一个主题，先讲"为什么"再讲"怎么做"
- 提供可直接复制执行的命令
- 标注平台差异（macOS / Linux / Windows）
- 修改后递增文件版本号（见[版本约定](#版本约定)）

## 版本约定

每份文档标题下方标注版本号，遵循语义化版本：

| 变更类型 | 示例 |
|---|---|
| 错别字、格式、注释等不改变含义的修改 | v1.0.0 → v1.0.1 |
| 新增内容、补充说明、向后兼容的改动 | v1.0.0 → v1.1.0 |
| 大重构、章节重组或不兼容的改写 | v1.0.0 → v2.0.0 |

## 许可证

本文档基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议发布。

您可以自由共享和改编本材料，前提是：
- **署名** — 您必须给出适当的署名并提供指向原项目的链接。
- **非商业性使用** — 您不得将本材料用于商业目的。
- **相同方式共享** — 如果您改编了本材料，您必须基于相同许可证分发您的贡献。
